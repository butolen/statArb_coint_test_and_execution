# Statistical Arbitrage Pair Trading Notebook Pipeline

Dieses Projekt enthaelt eine allgemeine Notebook-Pipeline zum Testen, Optimieren, Backtesten und Ueberwachen einer Pair-Trading-Strategie auf Basis von Cointegration und Mean Reversion.

Die Strategie modelliert ein `PRIMARY_SYMBOL` gegen ein `HEDGE_SYMBOL`:

```text
log(primary) = alpha + beta * log(hedge) + residual
```

Der Residual-Spread wird anschliessend als z-Score ausgewertet. Hohe positive oder negative Abweichungen koennen als Mean-Reversion-Signale interpretiert werden, sofern die statistischen Filter erfuellt sind.

## Inhalt

| Datei | Zweck |
| --- | --- |
| `coint_test.ipynb` | Prueft, ob ein ausgewaehltes Instrumentenpaar statistisch sinnvoll fuer eine Cointegration-/Mean-Reversion-Strategie ist. |
| `config_search.ipynb` | Testet mehrere Parameterkombinationen fuer ein gewaehltes Zeitfenster und sucht robuste Konfigurationen. |
| `backtest.ipynb` | Fuehrt einen finalen Backtest mit ausgewaehlten Parametern aus und zeigt Performance, Drawdown, Trades und Monte-Carlo-Robustheit. |
| `exec.ipynb` | Berechnet das aktuellste Signal und einen Positionsplan auf Basis der gewaehlten Parameter. |

## Voraussetzungen

Empfohlen wird Python 3.10 oder neuer.

Installiere die benoetigten Pakete:

```bash
pip install -r requirements.txt
```

Starte danach Jupyter:

```bash
jupyter notebook
```

Alternativ koennen die Notebooks direkt in PyCharm, VS Code oder JupyterLab geoeffnet werden.

## Grundidee

Die Pipeline arbeitet mit zwei Instrumenten:

- `PRIMARY_SYMBOL`: Das Instrument, dessen fairer Log-Preis modelliert wird.
- `HEDGE_SYMBOL`: Das Instrument, das als erklaerende Hedge-Komponente verwendet wird.

Aus den historischen Preisen werden Log-Preise berechnet. Per OLS-Regression wird ein Hedge-Verhaeltnis `beta` geschaetzt. Der Spread ist die Abweichung des tatsaechlichen Log-Preises vom modellierten fairen Log-Preis.

Ein typisches Signal entsteht so:

- `zscore > ENTRY_Z`: Spread ist hoch; moegliches Short-Spread-Signal.
- `zscore < -ENTRY_Z`: Spread ist niedrig; moegliches Long-Spread-Signal.
- `zscore` zurueck nahe `TP_Z`: Mean-Reversion-Ziel erreicht.
- Residual-ADF-p-Wert zu hoch: Stationaritaetsfilter schlaegt fehl, daher kein Trade oder Exit.

## Empfohlener Workflow

### 1. Instrumentenpaar waehlen

Oeffne zuerst `coint_test.ipynb` und passe im ersten Codeblock diese Werte an:

```python
PRIMARY_SYMBOL = "SYMBOL_A"
HEDGE_SYMBOL = "SYMBOL_B"
PRIMARY_LABEL = "Primary"
HEDGE_LABEL = "Hedge"

INTERVAL = "1h"
PERIOD = "2y"
```

Die Symbole muessen von Yahoo Finance unterstuetzt werden, da die Daten ueber `yfinance` geladen werden.

Beispiele fuer `INTERVAL`:

- `"1h"` fuer Stundenkerzen
- `"2h"` oder `"4h"` fuer groebere Intraday-Daten
- `"1d"` fuer Tagesdaten

Beispiele fuer `PERIOD`:

- `"6mo"`
- `"1y"`
- `"2y"`
- `"5y"`
- `"10y"`

Hinweis: Yahoo Finance unterstuetzt nicht jede Kombination aus `INTERVAL` und `PERIOD`. Wenn keine Daten geladen werden, zuerst ein einfacheres Intervall wie `"1d"` testen.

### 2. Cointegration pruefen

Fuehre `coint_test.ipynb` komplett aus.

Wichtige Ausgaben:

- ADF-Test der einzelnen Log-Preise
- OLS-Hedge-Ratio `beta`
- ADF-Test des Residual-Spreads
- Engle-Granger-Cointegrationstest
- Johansen-Test
- Rolling-Diagnostik fuer Stabilitaet ueber die Zeit

Faustregeln:

- Einzelne Preisreihen duerfen nicht stationaer sein; das ist bei Finanzpreisen normal.
- Der Residual-Spread sollte moeglichst stationaer sein.
- Ein niedriger Engle-Granger-p-Wert spricht fuer Cointegration.
- Rolling-Diagnostiken sollten nicht nur in einem kleinen Teil der Historie gut aussehen.

Wenn die Cointegration schwach oder instabil ist, sollte das Paar nicht weiter optimiert werden.

### 3. Parameter suchen

Wenn das Paar statistisch geeignet wirkt, oeffne `config_search.ipynb`.

Passe dort mindestens diese Werte an:

```python
PRIMARY_SYMBOL = "SYMBOL_A"
HEDGE_SYMBOL = "SYMBOL_B"
PRIMARY_LABEL = "Primary"
HEDGE_LABEL = "Hedge"

USER_INTERVAL = "1h"
USER_PERIOD = "2y"
```

Danach koennen die Suchraster angepasst werden:

```python
BETA_WINDOWS = [120, 200, 300]
Z_WINDOWS = [60, 100, 150]
ENTRY_Z_OPTIONS = [2.0, 2.4, 2.8]
STOP_Z_OPTIONS = [4.5, 5.0, 5.5]
MAX_HOLD_BARS_OPTIONS = [24, 48, 96]
MAX_RESID_ADF_PVALUE_OPTIONS = [0.05, 0.10]
```

Bedeutung:

- `BETA_WINDOWS`: Anzahl Bars fuer die rollierende Hedge-Ratio-Schaetzung.
- `Z_WINDOWS`: Anzahl Bars fuer Mittelwert und Standardabweichung des Spreads.
- `ENTRY_Z_OPTIONS`: Einstiegsschwellen fuer Mean-Reversion-Trades.
- `STOP_Z_OPTIONS`: Notausstieg, wenn der Spread weiter gegen die Position laeuft.
- `MAX_HOLD_BARS_OPTIONS`: Zeitstopp in Bars.
- `MAX_RESID_ADF_PVALUE_OPTIONS`: maximal erlaubter p-Wert des rollierenden Residual-ADF-Tests.

Die besten Konfigurationen werden nach mehreren Kriterien gerankt, nicht nur nach Rendite. Achte besonders auf:

- ausreichende Anzahl Trades
- positive Rendite
- akzeptablen Drawdown
- nicht zu isolierte Equity-Kurve
- stabile Nachbarkonfigurationen
- plausible Stationaritaetsquote

### 4. Finalen Backtest ausfuehren

Oeffne `backtest.ipynb` und uebertrage die ausgewaehlte Konfiguration aus `config_search.ipynb`.

Typische Parameter:

```python
BETA_WINDOW = 200
Z_WINDOW = 100
ENTRY_Z = 2.8
TP_Z = 0.0
TP_TOLERANCE = 0.15
STOP_Z = 8.0
MAX_HOLD_DAYS = 10
MAX_RESID_ADF_PVALUE = 0.10
```

Ausserdem Kapital, Gebuehren und Positionsgroesse pruefen:

```python
INITIAL_CAPITAL = 10_000.0
BASE_PORTFOLIO_FRACTION = 0.10
MAX_PORTFOLIO_FRACTION = 0.80
FEE_RATE = 0.00015
SLIPPAGE_RATE = 0.0
```

Der Backtest berechnet:

- Equity-Kurve
- Trade-Liste
- Rendite
- Sharpe Ratio
- Omega Ratio
- maximalen Drawdown
- Winrate
- durchschnittlichen Trade-PnL
- Monte-Carlo-Resampling der realisierten Trade-PnLs

Die Monte-Carlo-Auswertung zeigt nur Sequenzrisiko basierend auf historischen Trade-Ergebnissen. Sie ist keine Prognose zukuenftiger Renditen.

### 5. Aktuelles Signal berechnen

Oeffne `exec.ipynb` und setze dieselben Symbole und Parameter wie im finalen Backtest.

Wichtige Eingaben:

```python
PORTFOLIO_EQUITY_USD = 10_000.0
BASE_PORTFOLIO_FRACTION = 0.10
MAX_PORTFOLIO_FRACTION = 0.80
```

Das Notebook gibt aus:

- aktuellen Timestamp der letzten verfuegbaren Kerze
- aktuelle Preise
- `alpha`
- `beta`
- Spread
- z-Score
- Residual-ADF-p-Wert
- Signal
- Ziel-Gross-Notional
- Long-/Short-Plan fuer beide Legs

Moegliche Signale:

| Signal | Bedeutung |
| --- | --- |
| `LONG_SPREAD` | Primary long, Hedge short. |
| `SHORT_SPREAD` | Primary short, Hedge long. |
| `TAKE_PROFIT_ZONE` | Spread ist nahe am Mean-Reversion-Ziel; bestehende Position pruefen oder schliessen. |
| `NO_TRADE` | Kein neues Signal. |
| `NO_TRADE_STATIONARITY_FAIL` | Der statistische Filter ist nicht erfuellt. |

## Wichtige Parameter

| Parameter | Beschreibung |
| --- | --- |
| `PRIMARY_SYMBOL` | Instrument, dessen fairer Log-Preis modelliert wird. |
| `HEDGE_SYMBOL` | Instrument, das zur Absicherung bzw. Erklaerung verwendet wird. |
| `INTERVAL` / `USER_INTERVAL` | Datenfrequenz. |
| `PERIOD` / `USER_PERIOD` | Historienlaenge. |
| `BETA_WINDOW` | Rollierendes Fenster fuer die Hedge-Ratio-Schaetzung. |
| `Z_WINDOW` | Rollierendes Fenster fuer Spread-Mittelwert und Spread-Standardabweichung. |
| `ENTRY_Z` | Einstiegsschwelle fuer Spread-Abweichungen. |
| `TP_Z` | Zielwert fuer Take Profit, meist `0.0`. |
| `TP_TOLERANCE` | Toleranz um das Take-Profit-Ziel. |
| `STOP_Z` | Stop-Ausgang bei extremer Spread-Ausweitung. |
| `MAX_HOLD_DAYS` / `MAX_HOLD_BARS` | Zeitbasierter Exit. |
| `MAX_RESID_ADF_PVALUE` | Maximal erlaubter p-Wert fuer den Stationaritaetsfilter. |
| `BASE_PORTFOLIO_FRACTION` | Standard-Bruttoexposure pro Einstieg. |
| `MAX_PORTFOLIO_FRACTION` | Maximales Bruttoexposure relativ zum Portfolio. |
| `FEE_RATE` | Gebuehr pro Handelsseite als Anteil des gehandelten Notionals. |
| `SLIPPAGE_RATE` | Slippage pro Handelsseite als Anteil des gehandelten Notionals. |

## Positionslogik

Die Strategie verwendet Brutto-Notional:

```text
gross_notional = abs(primary_notional) + abs(hedge_notional)
```

Das Hedge-Verhaeltnis wird aus `beta` abgeleitet. Bei einem Long-Spread ist das Primary-Leg long und das Hedge-Leg short. Bei einem Short-Spread ist das Primary-Leg short und das Hedge-Leg long.

Wenn `POSITION_SCALING_ENABLED = False`, wird jede neue Position mit der festen `BASE_PORTFOLIO_FRACTION` eroeffnet.

Wenn `POSITION_SCALING_ENABLED = True`, kann die Positionsgroesse mit staerkerem z-Score steigen. Das Verhalten wird kontrolliert durch:

- `POSITION_SCALING_STEP_Z`
- `POSITION_SCALING_MODE`
- `POSITION_SCALING_INCREASE`
- `MAX_PORTFOLIO_FRACTION`

Positionsskalierung erhoeht Risiko deutlich und sollte nur nach separater Pruefung aktiviert werden.

## Interpretation der Ergebnisse

Eine Konfiguration ist nicht automatisch gut, nur weil die historische Rendite hoch ist. Pruefe immer:

- Ist die Cointegration statistisch plausibel?
- Ist die Rolling-Stationaritaet stabil?
- Gibt es genug Trades fuer eine sinnvolle Bewertung?
- Ist der Drawdown akzeptabel?
- Sind Gebuehren und Slippage realistisch?
- Funktionieren benachbarte Parameter aehnlich gut?
- Ist die Performance auf einzelne Ausreisser-Trades angewiesen?
- Ist das Instrumentenpaar liquide genug fuer die geplante Positionsgroesse?

## Typische Fehlerquellen

### Keine oder zu wenig Daten

Moegliche Ursachen:

- Symbol wird von Yahoo Finance nicht unterstuetzt.
- `INTERVAL` und `PERIOD` sind nicht kompatibel.
- Zu wenig Historie fuer `BETA_WINDOW` oder `Z_WINDOW`.

Loesung:

- Anderes Symbol testen.
- `INTERVAL = "1d"` verwenden.
- `PERIOD` erhoehen.
- Fenstergroessen reduzieren.

### Keine Trades im Backtest

Moegliche Ursachen:

- `ENTRY_Z` zu hoch.
- `MAX_RESID_ADF_PVALUE` zu streng.
- Spread ist nicht ausreichend mean-reverting.
- Zeitraum enthaelt zu wenige extreme Abweichungen.

Loesung:

- Cointegration erneut pruefen.
- Parametersuche breiter aufsetzen.
- Einstiegsschwellen moderat reduzieren.
- Anderes Paar testen.

### Sehr gute Rendite, aber wenige Trades

Das ist haeufig instabil. Eine Strategie mit sehr wenigen Trades kann zufaellig gut aussehen. In diesem Fall nicht nur die Top-Konfiguration waehlen, sondern robuste Nachbarkonfigurationen pruefen.

### Live-Signal weicht vom Backtest ab

Moegliche Ursachen:

- Parameter in `exec.ipynb` stimmen nicht mit `backtest.ipynb` ueberein.
- Yahoo Finance hat neue oder korrigierte Daten geliefert.
- Die letzte Kerze ist noch nicht final.
- Zeitzonen oder Handelszeiten unterscheiden sich.

## Hinweise zur Nutzung

Dieses Projekt ist ein Research- und Analysewerkzeug. Es fuehrt keine Orders automatisch aus. Das `exec.ipynb` erzeugt nur einen Positionsplan, der manuell geprueft werden muss.

Vor realem Einsatz sollten zusaetzlich geprueft werden:

- Orderausfuehrung
- Liquiditaet
- Finanzierungskosten
- Margin-Anforderungen
- Short-Verfuegbarkeit
- Steuern
- Datenqualitaet
- Out-of-sample-Test
- Walk-forward-Test
- Robustheit auf anderen Zeitraeumen

## Haftungsausschluss

Dieses Projekt dient ausschliesslich zu Lern-, Analyse- und Research-Zwecken. Es ist keine Anlageberatung, keine Handelsempfehlung und keine Aufforderung zum Kauf oder Verkauf von Finanzinstrumenten. Historische Backtests garantieren keine zukuenftigen Ergebnisse.

