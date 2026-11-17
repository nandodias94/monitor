#!/usr/bin/env python3
"""Golo à Vista (versão simples)

Corre de 5 em 5 minutos no GitHub Actions: vai buscar os jogos ao vivo à
API-Football, calcula um índice de pressão (0-100) por equipa com base nos
últimos ~10 minutos e envia uma notificação (ntfy e/ou Telegram) quando o
índice passa o limiar.

Só usa a biblioteca padrão do Python (não precisa de pip install).
"""
import json
import os
import sys
import time
import urllib.error
import urllib.parse
import urllib.request
from datetime import datetime, timezone
from zoneinfo import ZoneInfo

# --------------------------------------------------------------------------
# Configuração (tudo vem de variáveis de ambiente; vazio = valor por defeito)
# --------------------------------------------------------------------------


def _env(name, default=None):
    v = (os.environ.get(name) or "").strip()
    return v if v else default


def _int(name, default):
    try:
        return int(_env(name, default))
    except (TypeError, ValueError):
        return default


def _float(name, default):
    try:
        return float(_env(name, default))
    except (TypeError, ValueError):
        return default


API = "https://v3.football.api-sports.io"
API_KEY = _env("API_FOOTBALL_KEY")
NTFY_TOPIC = _env("NTFY_TOPIC")
NTFY_SERVER = (_env("NTFY_SERVER", "https://ntfy.sh") or "").rstrip("/")
TG_TOKEN = _env("TELEGRAM_BOT_TOKEN")
TG_CHAT = _env("TELEGRAM_CHAT_ID")

THRESHOLD = _int("THRESHOLD", 60)              # índice mínimo para alertar (0-100)
COOLDOWN_MIN = _int("COOLDOWN_MIN", 15)        # minutos entre alertas do mesmo jogo/equipa
MIN_MINUTE = _int("MIN_MINUTE", 5)
MAX_MINUTE = _int("MAX_MINUTE", 90)
MAX_FIXTURES_PER_RUN = _int("MAX_FIXTURES_PER_RUN", 15)
DAILY_BUDGET = _int("DAILY_REQUEST_BUDGET", 90)  # pedidos à API por dia (UTC)
SCALE = _float("SCALE", 4.0)                   # pontos brutos -> índice
QUIET_START = _int("QUIET_START", None)        # hora (Lisboa) em que começa o silêncio
QUIET_END = _int("QUIET_END", None)            # hora (Lisboa) em que termina

# Se definido (ex.: "39,40,94"), só estas ligas são analisadas. Vazio = todas.
LEAGUES = {
    int(x) for x in (_env("LEAGUE_IDS", "") or "").split(",") if x.strip().isdigit()
}

LIVE_STATUSES = {"1H", "2H", "ET"}
STATE_PATH = os.path.join("state", "state.json")


class BudgetExceeded(Exception):
    pass


# --------------------------------------------------------------------------
# Utilitários
# --------------------------------------------------------------------------


def http_json(method, url, headers=None, body=None, label="pedido", timeout=20):
    hdrs = {"User-Agent": "golo-a-vista/1.0"}
    hdrs.update(headers or {})
    data = None
    if body is not None:
        data = json.dumps(body).encode("utf-8")
        hdrs["Content-Type"] = "application/json; charset=utf-8"
    req = urllib.request.Request(url, data=data, headers=hdrs, method=method)
    try:
        with urllib.request.urlopen(req, timeout=timeout) as resp:
            raw = resp.read().decode("utf-8", "replace")
            return (json.loads(raw) if raw.strip() else {}), resp.headers
    except urllib.error.HTTPError as e:
        detail = e.read().decode("utf-8", "replace")[:200]
        raise RuntimeError(f"HTTP {e.code} em {label}: {detail}") from None
    except urllib.error.URLError as e:
        raise RuntimeError(f"Falha de rede em {label}: {e.reason}") from None


def num(v):
    """Converte valores da API (None, '55%', 3) em número."""
    if v is None:
        return 0.0
    if isinstance(v, (int, float)):
        return float(v)
    try:
        return float(str(v).replace("%", "").strip() or 0)
    except ValueError:
        return 0.0


def in_quiet_hours():
    if QUIET_START is None or QUIET_END is None:
        return False
    h = datetime.now(ZoneInfo("Europe/Lisbon")).hour
    if QUIET_START <= QUIET_END:
        return QUIET_START <= h < QUIET_END
    return h >= QUIET_START or h < QUIET_END


def load_state():
    try:
        with open(STATE_PATH, "r", encoding="utf-8") as f:
            return json.load(f)
    except (FileNotFoundError, json.JSONDecodeError):
        return {"fixtures": {}, "usage": {}}


def save_state(state):
    os.makedirs(os.path.dirname(STATE_PATH), exist_ok=True)
    with open(STATE_PATH, "w", encoding="utf-8") as f:
        json.dump(state, f)


def api_get(path, params, state):
    day = datetime.now(timezone.utc).strftime("%Y-%m-%d")
    usage = state.setdefault("usage", {})
    if usage.get("day") != day:
        usage.clear()
        usage.update({"day": day, "count": 0})
    if usage["count"] >= DAILY_BUDGET:
        raise BudgetExceeded()
    usage["count"] += 1
    url = f"{API}{path}?{urllib.parse.urlencode(params)}"
    data, _ = http_json(
        "GET", url, headers={"x-apisports-key": API_KEY}, label=f"API-Football {path}"
    )
    errors = data.get("errors")
    if errors:  # a API devolve [] ou {} quando está tudo bem
        raise RuntimeError(f"API-Football devolveu erros: {errors}")
    return data.get("response") or []


def notify(title, message):
    """Envia por ntfy e/ou Telegram. Devolve True se pelo menos um canal enviou."""
    sent = False
    if NTFY_TOPIC:
        try:
            http_json(
                "POST",
                f"{NTFY_SERVER}/",
                body={
                    "topic": NTFY_TOPIC,
                    "title": title,
                    "message": message,
                    "priority": 4,
                    "tags": ["soccer"],
                },
                label="ntfy",
            )
            sent = True
        except RuntimeError as e:
            print(f"ERRO ntfy: {e}")
    if TG_TOKEN and TG_CHAT:
        try:
            http_json(
                "POST",
                f"https://api.telegram.org/bot{TG_TOKEN}/sendMessage",
                body={"chat_id": TG_CHAT, "text": f"{title}\n{message}"},
                label="Telegram",
            )
            sent = True
        except RuntimeError as e:
            print(f"ERRO Telegram: {e}")
    if not sent:
        print("AVISO: nenhuma notificação foi enviada (verifica os segredos).")
    return sent


# --------------------------------------------------------------------------
# Dados e cálculo
# --------------------------------------------------------------------------


def parse_live(items):
    matches = []
    for it in items:
        st = it["fixture"]["status"]
        if st.get("short") not in LIVE_STATUSES:
            continue
        league = it["league"]
        if LEAGUES and league["id"] not in LEAGUES:
            continue
        matches.append(
            {
                "id": it["fixture"]["id"],
                "minute": st.get("elapsed") or 0,
                "league_id": league["id"],
                "league": league.get("name", "?"),
                "country": league.get("country", ""),
                "home_id": it["teams"]["home"]["id"],
                "home": it["teams"]["home"]["name"],
                "away_id": it["teams"]["away"]["id"],
                "away": it["teams"]["away"]["name"],
                "gh": it["goals"]["home"] or 0,
                "ga": it["goals"]["away"] or 0,
            }
        )
    return matches


def parse_stats(resp, m):
    """Devolve {'home': {...}, 'away': {...}} ou None se a liga não tem estatísticas."""
    out = {}
    for item in resp:
        tid = item["team"]["id"]
        s = {x["type"]: x["value"] for x in item.get("statistics", [])}
        parsed = {
            "sot": num(s.get("Shots on Goal")),
            "off": num(s.get("Shots off Goal")),
            "blocked": num(s.get("Blocked Shots")),
            "corners": num(s.get("Corner Kicks")),
            "poss": num(s.get("Ball Possession")),
            "red": num(s.get("Red Cards")),
            "xg": num(s.get("expected_goals")),
        }
        if tid == m["home_id"]:
            out["home"] = parsed
        elif tid == m["away_id"]:
            out["away"] = parsed
    if "home" in out and "away" in out:
        return out
    return None


def pick_baseline(snaps, minute):
    """Snapshot anterior cuja distância em minutos de jogo está mais perto de 10 (entre 4 e 14)."""
    cands = [s for s in snaps if 4 <= minute - s["m"] <= 14]
    if not cands:
        return None
    return min(cands, key=lambda s: abs((minute - s["m"]) - 10))


def pressure(base, cur, side, opp, m):
    """Índice 0-100 para a equipa `side` com base na diferença entre dois snapshots."""
    b, c = base[side], cur[side]
    bo, co = base[opp], cur[opp]

    def d(k):
        return max(0.0, c[k] - b[k])

    raw = 3.0 * d("sot") + 1.0 * d("off") + 1.0 * d("blocked") + 2.0 * d("corners")
    raw += 10.0 * d("xg")
    raw += 4.0 * max(0.0, co["red"] - bo["red"])  # cartão vermelho ao adversário na janela
    if c["poss"] >= 60:
        raw += 1.0

    minute = cur["m"]
    goals_for = m["gh"] if side == "home" else m["ga"]
    goals_against = m["ga"] if side == "home" else m["gh"]
    mult = 1.0
    if minute >= 75:
        mult *= 1.15
    if minute >= 60 and goals_for == goals_against - 1:
        mult *= 1.10
    if c["red"] > 0:  # equipa reduzida a 10
        mult *= 0.8
    return min(100, round(raw * mult * SCALE)), raw


def priority(m):
    """Menor = mais prioritário (para decidir que jogos gastam pedidos)."""
    p = 0
    if m["league_id"] in LEAGUES:
        p -= 100
    if m["minute"] >= 60 and abs(m["gh"] - m["ga"]) <= 1:
        p -= 10
    return (p, -m["minute"])


# --------------------------------------------------------------------------
# Ciclo principal
# --------------------------------------------------------------------------


def run_cycle(state, now):
    fixtures = state.setdefault("fixtures", {})
    live = parse_live(api_get("/fixtures", {"live": "all"}, state))

    # Jogos já decididos (3+ golos de diferença) não valem os pedidos.
    candidates = [m for m in live if abs(m["gh"] - m["ga"]) < 3]
    candidates.sort(key=priority)
    selected = candidates[:MAX_FIXTURES_PER_RUN]
    analysed = 0

    for m in selected:
        fs = fixtures.setdefault(str(m["id"]), {"snaps": [], "alerts": {}})
        fs["seen"] = now
        if fs.get("no_stats_until", 0) > now:
            continue
        try:
            resp = api_get("/fixtures/statistics", {"fixture": m["id"]}, state)
        except BudgetExceeded:
            print("Orçamento diário de pedidos esgotado; a parar este ciclo.")
            break
        except RuntimeError as e:
            print(f"ERRO nas estatísticas do jogo {m['id']}: {e}")
            continue

        stats = parse_stats(resp, m)
        if stats is None:
            fs["no_stats_until"] = now + 1800  # liga sem estatísticas: não insistir 30 min
            continue
        analysed += 1

        snap = {"t": now, "m": m["minute"], "home": stats["home"], "away": stats["away"]}
        base = pick_baseline(fs["snaps"], m["minute"])
        fs["snaps"] = [s for s in fs["snaps"] if now - s["t"] <= 25 * 60] + [snap]
        if base is None:
            print(f"{m['home']}-{m['away']} {m['minute']}': a recolher dados (sem janela ainda)")
            continue

        scores = {
            "home": pressure(base, snap, "home", "away", m)[0],
            "away": pressure(base, snap, "away", "home", m)[0],
        }
        print(
            f"{m['home']} {m['gh']}-{m['ga']} {m['away']} {m['minute']}' "
            f"({m['league']}): pressão casa {scores['home']} / fora {scores['away']}"
        )

        side = max(scores, key=scores.get)
        idx = scores[side]
        if idx < THRESHOLD or not (MIN_MINUTE <= m["minute"] <= MAX_MINUTE):
            continue
        if now - fs["alerts"].get(side, 0) < COOLDOWN_MIN * 60:
            continue

        team = m["home"] if side == "home" else m["away"]
        title = f"⚽ Golo à vista: {team}"
        body = (
            f"{m['home']} {m['gh']}-{m['ga']} {m['away']} · {m['minute']}' · "
            f"Pressão {idx}/100\n{m['league']} ({m['country']})"
        )
        if notify(title, body):
            fs["alerts"][side] = now
            print(f"ALERTA enviado: {team} (índice {idx})")

    # Limpar jogos que já não aparecem há mais de 3 horas
    live_ids = {str(m["id"]) for m in live}
    for fid in list(fixtures):
        if fid not in live_ids and now - fixtures[fid].get("seen", 0) > 3 * 3600:
            del fixtures[fid]

    used = state.get("usage", {}).get("count", 0)
    print(
        f"{len(live)} jogos ao vivo, {len(selected)} escolhidos, {analysed} analisados. "
        f"Pedidos hoje: {used}/{DAILY_BUDGET}"
    )


def run_test():
    if API_KEY:
        try:
            data, _ = http_json(
                "GET",
                f"{API}/status",
                headers={"x-apisports-key": API_KEY},
                label="API-Football /status",
            )
            if data.get("errors"):
                print(f"API-Football: chave inválida ou erro: {data['errors']}")
            else:
                r = data.get("response") or {}
                req = r.get("requests", {}) if isinstance(r, dict) else {}
                plan = (r.get("subscription", {}) or {}).get("plan", "?") if isinstance(r, dict) else "?"
                print(
                    f"API-Football OK. Plano: {plan}. "
                    f"Pedidos hoje: {req.get('current', '?')}/{req.get('limit_day', '?')}"
                )
        except RuntimeError as e:
            print(f"ERRO a validar a chave da API-Football: {e}")
    else:
        print("AVISO: falta o segredo API_FOOTBALL_KEY.")
    ok = notify("⚽ Golo à Vista: teste", "Se estás a ler isto, as notificações estão a funcionar.")
    sys.exit(0 if ok else 1)


def main():
    if "--test" in sys.argv:
        run_test()
    if not API_KEY:
        sys.exit("Falta o segredo API_FOOTBALL_KEY.")
    if not (NTFY_TOPIC or (TG_TOKEN and TG_CHAT)):
        sys.exit("Falta configurar um canal: NTFY_TOPIC ou TELEGRAM_BOT_TOKEN + TELEGRAM_CHAT_ID.")
    if in_quiet_hours():
        print("Horas de silêncio: nada a fazer neste ciclo.")
        return
    state = load_state()
    try:
        run_cycle(state, int(time.time()))
    except BudgetExceeded:
        print("Orçamento diário de pedidos esgotado.")
    except RuntimeError as e:
        print(f"ERRO: {e}")
    finally:
        save_state(state)


if __name__ == "__main__":
    main()