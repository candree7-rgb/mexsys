# MEXC Futures Trading Bot - Briefing für Claude Code

## Projekt-Übersicht

Wir bauen einen halbautomatischen Trading-Bot für MEXC Futures mit folgenden Eigenschaften:
- **Strategie:** Reversal-Zone-Durchbruch bei Oversold-Bedingungen
- **Timeframe:** 15-Minuten-Kerzen
- **RR:** 1:1 (z.B. 0.4% TP / 0.4% SL)
- **Target WR:** 70%+
- **Trades pro Tag:** 5-20 max

---

## Technischer Ansatz

### Warum User-Token statt Browser-Automation?

MEXC hat die offizielle Futures-API für Order-Platzierung deaktiviert. Wir nutzen einen Hybrid-Ansatz:

| Aktion | Methode | Grund |
|--------|---------|-------|
| Account Balance | Offizielle API ✅ | Erlaubt, stabil |
| Max Leverage pro Coin | Offizielle API ✅ | Erlaubt, stabil |
| Position Details | Offizielle API ✅ | Erlaubt, stabil |
| Order Status checken | Offizielle API ✅ | Erlaubt, stabil |
| **Order platzieren** | User-Token 🔑 | API deaktiviert |
| **Order canceln** | User-Token 🔑 | API deaktiviert |
| **TP/SL setzen** | User-Token 🔑 | API deaktiviert |

### Basis-Repository

Wir nutzen ein GitHub-Repo als Basis, das den MEXC User-Token-Bypass bereits implementiert hat. Dieses Repo klonen und als Foundation verwenden.

---

## Kern-Anforderungen

### 1. Order-Platzierung (1 Request)

```python
# Pseudo-Code für Order mit TP/SL
place_order(
    symbol="BTCUSDT",
    side="BUY",  # oder "SELL"
    type="LIMIT",
    price=50000,
    quantity=0.1,
    take_profit=50200,  # +0.4%
    stop_loss=49800     # -0.4%
)
```

**Wichtig:** TP und SL müssen im gleichen Request wie die Order gesetzt werden.

### 2. Position Sizing

```python
# Input-Parameter
RISK_PERCENT = 2  # Max 2% vom Account pro Trade
MAX_LEVERAGE = 20
MAX_POSITION_USDT = 500

# Berechnung
def calculate_position_size(account_balance, entry_price, sl_percent):
    risk_amount = account_balance * (RISK_PERCENT / 100)
    position_size = risk_amount / (sl_percent / 100)
    
    # Limits checken
    position_size = min(position_size, MAX_POSITION_USDT)
    
    # Jitter für natürlicheres Aussehen
    jitter = random.uniform(0.95, 1.05)
    return position_size * jitter
```

### 3. Order-Management pro Candle

```python
# Alle 15 Minuten
def on_candle_close():
    # 1. Offene Limit-Order vorhanden?
    if has_pending_order():
        cancel_order()  # Token-Request
    
    # 2. Neue Entry-Werte vom Strategy-Script holen
    entry, tp, sl, side = get_strategy_signal()
    
    # 3. Falls Signal vorhanden
    if entry:
        # Account-Daten holen (offizielle API)
        balance = get_account_balance()
        max_lev = get_max_leverage(symbol)
        
        # Position Size berechnen
        size = calculate_position_size(balance, entry, sl)
        
        # Order platzieren (Token-Request)
        place_order(
            symbol=symbol,
            side=side,
            price=entry,
            quantity=size,
            take_profit=tp,
            stop_loss=sl
        )
```

---

## Anti-Detection Maßnahmen

### Pflicht:
- **Jitter auf Timing:** Nicht exakt bei Candle-Close, sondern +/- 2-8 Sekunden random
- **Jitter auf Position Size:** ×0.95 bis ×1.05
- **Echter User-Agent:** Chrome/Firefox Header verwenden
- **Session-Token:** Wie normaler Browser-Login

### Optional aber empfohlen:
- Zufällige Delays zwischen API-Calls (1-3 Sekunden)
- Nicht mehr als 50 Requests pro Stunde

### Request-Volumen Einschätzung:
- Pro Trade: 1-2 Token-Requests (Place + evtl. Cancel)
- Pro Tag bei 20 Trades: ~40-50 Token-Requests
- Das ist **weniger** als manuelles Trading mit Page-Refreshes

---

## Konfiguration

```python
# config.py

# Trading
SYMBOL = "BTCUSDT"
TIMEFRAME = "15m"
TP_PERCENT = 0.4
SL_PERCENT = 0.4
RISK_PERCENT = 2

# Limits (selbst gesetzt, nicht von API)
MAX_LEVERAGE = 20
MAX_POSITION_USDT = 500
MIN_POSITION_USDT = 10

# Anti-Detection
TIMING_JITTER_SECONDS = (2, 8)
SIZE_JITTER_RANGE = (0.95, 1.05)
REQUEST_DELAY_SECONDS = (1, 3)

# API
MEXC_API_KEY = "..."      # Für offizielle Read-Only Endpoints
MEXC_API_SECRET = "..."   # Für offizielle Read-Only Endpoints
MEXC_USER_TOKEN = "..."   # Für Order-Platzierung (aus Browser)
```

---

## Datei-Struktur

```
mexc-bot/
├── config.py           # Konfiguration
├── main.py             # Hauptloop
├── api/
│   ├── official.py     # Offizielle API (Balance, Leverage, etc.)
│   └── token.py        # Token-basierte Requests (Orders)
├── strategy/
│   └── reversal.py     # Deine Reversal-Strategie Logik
├── utils/
│   ├── position.py     # Position Sizing
│   └── jitter.py       # Anti-Detection Helpers
└── logs/
    └── trades.log      # Trade-History
```

---

## Token-Beschaffung

Der User-Token muss aus dem Browser extrahiert werden:

1. MEXC Futures öffnen, einloggen
2. DevTools öffnen (F12)
3. Network Tab
4. Irgendeine Aktion machen (Order-Fenster öffnen o.ä.)
5. Request suchen der an Futures-API geht
6. Headers → Authorization oder Cookie mit Token finden

**Token läuft ab!** Muss regelmäßig erneuert werden (täglich oder bei Logout).

---

## Offene Fragen für Implementierung

1. **Welches Basis-Repo nutzen wir?** (Link zum MEXC-Bypass-Repo)
2. **Wie sieht der Strategy-Output aus?** (Entry, TP, SL, Side als JSON/Dict?)
3. **Soll der Bot dauerhaft laufen oder manuell gestartet werden?**
4. **Mehrere Coins gleichzeitig oder nur einer?**

---

## Nächste Schritte

1. [ ] Basis-Repo klonen
2. [ ] Token-Extraktion testen
3. [ ] Offizielle API-Calls implementieren (Balance, Leverage)
4. [ ] Order-Platzierung mit Token implementieren
5. [ ] Position Sizing Logik
6. [ ] Candle-Loop mit Jitter
7. [ ] Strategy-Integration
8. [ ] Testing mit Minimal-Amounts

---

## Wichtige Hinweise

⚠️ **Risiko:** Token-Nutzung ist gegen MEXC ToS. Bei Detection: Account-Sperre möglich.

⚠️ **Funds:** Nie mehr auf MEXC lassen als du bereit bist zu verlieren.

⚠️ **Testing:** Erst mit minimalen Beträgen testen!

⚠️ **Backups:** Profits regelmäßig abziehen.
