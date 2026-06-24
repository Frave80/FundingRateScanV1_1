# FundingRateScan

Applicazione desktop **Windows (C# / WinForms, .NET 8)** per individuare le criptovalute
con **forte affollamento long** (funding rate positivo e persistente) su più exchange
centralizzati e DEX, e capire quali mostrano già un **pattern di volumi e open interest
crescenti**.

Pensata come evoluzione "applicativa" dello script Python `FundingRateHeatmap`: stessa
logica di fondo (funding rate come metrica primaria, volumi e OI come conferma), ma con
interfaccia a finestre, menu superiori, parametri configurabili e classificazione
ricalibrabile in tempo reale.

---

## Requisiti

- **Visual Studio 2022** (17.8 o successivo) con il carico di lavoro *.NET desktop development*
- **.NET 8 SDK** (oppure 9/10: il progetto compila anche con SDK più recenti)

## Come aprire / compilare

1. Apri `FundingRateScan.sln` con Visual Studio 2022.
2. Premi **F5** (Debug) o **Ctrl+F5** (senza debug) per avviare.

Da riga di comando:

```powershell
dotnet build FundingRateScan.sln      # compila
dotnet run --project src               # esegue
```

---

## Exchange supportati

| Exchange     | Tipo | Funding | Volume 24h | Open Interest | Storico OI/Vol |
|--------------|------|:-------:|:----------:|:-------------:|:--------------:|
| Binance      | CEX  | ✓ | ✓ | ✓ | ✓ (OI hist + klines) |
| Bybit        | CEX  | ✓ | ✓ | ✓ | ✓ (OI hist + klines) |
| OKX          | CEX  | ✓ | ✓ | ✓ | ✓ (rubik stat)       |
| Bitget       | CEX  | ✓ | ✓ | ✓ | funding hist          |
| Gate.io      | CEX  | ✓ | ✓ | ✓ | funding hist          |
| HTX          | CEX  | ✓ | ✓ | ✓ | funding hist          |
| Hyperliquid  | DEX  | ✓ | ✓ | ✓ | funding hist          |

Tutti gli endpoint usati sono di **market data pubblici**: l'app funziona senza chiavi API.
Le **API key** (impostabili nella finestra di scaricamento) servono solo per ottenere
limiti di rate più alti su alcuni exchange.

---

## Come si usa

### 1. Finestra di scaricamento (`Scansione → Nuova scansione`)
Permette di impostare:

- **Soglia affollamento long** — funding APR minimo (annualizzato) per considerare una coin "long-crowded";
- **Persistenza minima** — % di periodi con funding positivo nello storico;
- **Giorni di storico**, **numero max di candidate**, **volume minimo 24h**;
- **Crescita minima di volume e open interest** per il pattern attivo;
- **Exchange da usare** + relative **API key/secret/passphrase** (colonne editabili).

I parametri e le chiavi vengono salvati in
`%APPDATA%\FundingRateScan\settings.json`.

### 2. Scansione (fusione nativo + Python/Colab)
La scansione avviene in **due fasi** e la tabella mostra l'unione delle due selezioni:

**Fase 1 — Selezione nativa** (`ScanService`, mostrata subito):

1. scarica in parallelo i dati correnti da tutti gli exchange abilitati;
2. **aggrega per asset base** (es. `BTC` da più exchange → funding medio, volumi e OI sommati);
3. **filtra** le coin con funding APR ≥ soglia e volume ≥ minimo;
4. **arricchisce** le candidate con lo storico (funding, OI, volume);
5. calcola le metriche e **classifica**.

**Fase 2 — Selezione Python (Colab)**, eseguita subito dopo (pipeline completa
CoinGecko top 500 + storico funding 365→180→90gg su OKX/Hyperliquid/Bitget/HTX/Gate.io):

- le coin trovate **da entrambe** le selezioni vengono marcate `Entrambi` e arricchite
  con **Status Colab** e **APR Colab 30g**;
- le coin trovate **solo dal Python** vengono **aggiunte in coda** alla tabella
  (sfondo azzurro, Fonte = `Colab`).

Colonne dedicate: **Fonte** (Nativo / Colab / Entrambi), **Status Colab**, **APR Colab 30g**.

Inoltre: **clic sul simbolo** della coin → apre la pagina della coin su **CoinMarketCap**
(se la pagina non esiste, si apre automaticamente il grafico su un sito alternativo:
**TradingView** per le coin con perpetuo, **DEX Screener** per quelle solo-Colab);
pulsante **Dettagli** su ogni riga → criteri di scelta applicati (valori, soglie, esito) e
**probabilità stimata di alta volatilità a 30 / 60 / 120 giorni** (modello euristico
hazard giornaliero → probabilità cumulata, con motivazioni; non è un consiglio finanziario).

### 3. Classificazione
- **● Pattern attivo** — funding persistente **+ volume crescente + open interest crescente**;
- **○ Da osservare** — funding persistente, ma volume/OI non ancora confermati;
- *Neutra* — sotto soglia (nascosta di default).

### 4. Indicatori regolabili (barra in alto)
Funding APR, Persistenza, Crescita volume, Crescita OI: spostandoli e premendo
**Applica filtro** la classificazione viene **ricalcolata senza riscaricare i dati**.
Il menu a tendina *Mostra* filtra la vista (solo pattern attivo / pattern + osservare / tutte).

### 4-bis. Aree decisionali / Priorità
Menu **Visualizza → Aree decisionali / Priorità** (`Ctrl+P`, o pulsante *Aree / Priorità*):
ordina le coin candidate **per priorità** e le divide in due aree, indicando dove è
**attesa alta volatilità**.

- **Area Operativa** (più concreta) — funding persistente **confermato** da volumi e/o
  open interest crescenti: il pattern è già in atto → **priorità Alta** (entrambe le
  conferme) o **Media** (conferma parziale).
- **Area Accademica** (più teorica) — pressione funding persistente **senza** conferma di
  volumi/OI: si basa sulla tesi che i long affollati anticipino il movimento, ma è un
  segnale **probabilistico non confermato dai flussi** → **priorità Bassa**, sola osservazione.
- **Alta volatilità attesa** — long affollati (funding elevato) e persistenti con open
  interest in aumento = accumulo di leva → rischio di **squeeze / movimenti bruschi**
  (evidenziata in rosso, anche nell'area accademica).

Le stesse informazioni compaiono come colonne **Priorità** e **Vol. attesa** nella tabella
principale, ricalcolate quando sposti gli indicatori regolabili.

Il report ha tre schede:
1. **Priorità & Aree** — lista ordinata per priorità, area operativa/accademica e consiglio;
2. **Da monitorare (volatilità)** — **coin consigliate da monitorare** perché è attesa grande
   volatilità (long affollati persistenti + open interest in aumento = rischio squeeze),
   con indice di volatilità e motivazione per ciascuna;
3. **Più promettenti (accademico)** — le coin migliori secondo **criteri accademici**: un
   **modello fattoriale cross-sezionale** che standardizza ogni fattore (z-score
   sull'universo delle candidate) e li combina con pesi da letteratura sui perpetual
   futures — **Carry funding 30% · Persistenza 25% · Momentum Open Interest 20% ·
   Momentum Volume 15% · Liquidità 10%**. Lo *Score /100* è il rank percentile del
   composito, con la scomposizione per fattore (z) e i fattori dominanti di ogni coin.
   È un punteggio relativo e spiegabile, non una raccomandazione di acquisto.

### 5. Dettaglio coin + heatmap (stile Coinglass)
Doppio clic su una riga (oppure `Visualizza → Dettaglio coin selezionata`, `Ctrl+D`,
o il pulsante *Dettaglio / Heatmap*) apre una finestra con:

- i **dati numerici** completi della coin;
- la scheda **Heatmap funding**: calendario con righe = **mesi**, colonne = **giorni
  del mese (1→31)**; ogni cella è la **media giornaliera** del funding APR, colorata da
  **nero** (basso/negativo) a **rosso acceso** (funding elevato), con barra colore e tooltip.
  Un selettore **Periodo** permette di scegliere **1, 2, 6 o 12 mesi** (lo storico viene
  scaricato on-demand con paginazione dall'exchange di riferimento);
- le schede **Volume storico** e **Open interest storico** (heatmap a riga singola per data).

La tabella principale mostra anche la colonna **Nome** (nome completo della coin, da CoinGecko)
accanto al simbolo.

### 6. Esporta CSV
`Scansione → Esporta CSV` salva la tabella corrente.

### 7. OnChain Capital Flow Monitor
Menu **On-Chain → Monitor flussi on-chain** (o pulsante *On-Chain Monitor*, `Ctrl+O`):
sottosistema modulare che traccia i movimenti on-chain di un token e li correla col funding.

- **Input**: chain EVM (Ethereum, BNB Chain, Polygon, Arbitrum, Base, Optimism),
  contract address, simbolo perpetuo (per il funding), soglie whale e wallet milionario.
- **Schede**:
  - **Movimenti rilevanti** — trasferimenti con USD, tipo evento (deposito/prelievo CEX,
    acquisto/vendita DEX, transfer), etichette controparti ed evidenza (🐋 whale, CEX, LP, 💰);
  - **Whale wallet** — top holder con quota %, saldo, valore USD e classificazione
    (Exchange, Liquidity Pool, Contract, Whale, Mint/Burn…);
  - **Net flow & score** — aggregati per finestra (5m/1h/4h/24h/7g): net in/out, CEX in/out,
    whale buy/sell; contesto di mercato e funding; **punteggio esplicabile** con motivazioni;
  - **Alert** — storico alert con severità (Low/Medium/High/Critical) e filtro.
- **Provider** (adapter pattern, sostituibili): **Etherscan V2** (trasferimenti ERC-20,
  chiave gratuita), **Ethplorer** (top holder Ethereum, `freekey`), **DEX Screener**
  (prezzo/liquidità/pair LP, senza chiave). Chiavi e soglie in
  **On-Chain → Impostazioni provider** (salvate in `settings.json`).
- **Persistenza**: transazioni, holder snapshot e alert in JSON sotto
  `%APPDATA%\FundingRateScan\onchain\` (snapshot usati per rilevare nuovi wallet milionari
  e variazioni di concentrazione tra una sincronizzazione e l'altra).

> Lo scoring è un **supporto decisionale probabilistico**, non una previsione del prezzo:
> ogni contributo al punteggio è motivato e i flussi vanno interpretati nel contesto
> (exchange omnibus, bridge, vesting, market maker, wallet collegati).

### 7-bis. Caccia gemme — coin esplosive early-stage
Menu **On-Chain → Caccia gemme (coin esplosive)** (`Ctrl+G`, o pulsante *Caccia gemme*):
cerca potenziali **nuove PEPE/BRETT prima del large cap**. A differenza dello scanner
funding (coin già con perpetui), qui i candidati vengono dal feed **DEX Screener**
(token *trending/boosted* e *nuovi profili*, o ricerca testuale) e sono classificati con
uno **score esplosivo** multi-metrica:

- **Momentum 25%** — variazioni 1h/6h (con penalità da pump esaurito);
- **Volume 25%** — turnover (volume/market cap) + accelerazione (volume 1h vs 24h);
- **Pressione d'acquisto 20%** — quota buy sui txns;
- **Salute liquidità 15%** — liquidità assoluta + banda sana del rapporto liquidità/mcap;
- **Earliness 15%** — micro market cap + età recente (più "prima del large cap").

Filtri: chain, **market cap massimo** (default 5M $ per restare early), liquidità e volume
minimi, età massima. Ogni gemma ha la **scomposizione del punteggio** e **flag di rischio**
anti-scam (liquidità bassa, liquidità sottile vs mcap, pressione di vendita, appena creata,
pump esaurito). Doppio clic apre il token su DEX Screener; export CSV disponibile.

> Strumento di **scoperta probabilistica**: le micro-cap sono ad altissimo rischio (scam,
> rug pull, illiquidità). I flag aiutano a filtrare, ma vanno sempre verificati a mano.

### 7-ter. Caccia gemme PRO — scanner 100x/200x multi-provider
Menu **On-Chain → Caccia gemme PRO (scanner 100x)** (`Ctrl+Shift+G`, o pulsante
*Gemme PRO (100x)*): versione professionale dello scanner gemme, pensata per cercare
le nuove coin in grado di fare **100x/200x** (stile **PEPE · BONK · TURBO**).

**Fornitori di dati** (pannello in alto, con il campo **API key abbinato a ogni fornitore**;
chiavi salvate in `settings.json`):

| Provider | Tipo | Uso |
|---|---|---|
| DEX Screener | Gratuito | scoperta token e dati DEX (sempre attivo) |
| GeckoTerminal | Gratuito | verifica mcap/FDV e presenza su CoinGecko |
| GoPlus Security | Gratuito (chiave opz.) | honeypot, tasse buy/sell, mint, holders, LP lock |
| Honeypot.is | Gratuito | test honeypot ETH/BSC/Base (fallback) |
| Birdeye | API key | dati avanzati Solana (numero holders) |
| Moralis | API key | contratto verificato / segnalazione spam EVM |

**Pipeline**: scoperta + scoring di mercato → controlli di sicurezza sulle prime N gemme →
**MoonScore** = mercato 65% + sicurezza 35%; **Potenziale** = multiplo teorico verso la
capitalizzazione "classe TURBO" (~250M$). Opzione *Escludi honeypot* attiva di default.

**Dossier per ogni coin**: il pulsante **Dossier** su ogni riga genera un **documento di
approfondimento** (esportabile in `.md`/`.txt`) che spiega **tutti i parametri** usati
nella selezione (capitalizzazione/earliness, momentum, volume e accelerazione, pressione
d'acquisto, salute liquidità, sicurezza del contratto), le **motivazioni dei possibili
rialzi** (catalizzatori rilevati) e la **matematica del multiplo** (mcap implicita a
10x/50x/100x/200x confrontata con i benchmark PEPE/BONK/TURBO), più rischi e red flag.

### 8. Replica Python (Colab)
Menu **Scansione → Replica Python (Colab)** (`Ctrl+R`, o pulsante *Replica Colab*):
ricostruzione **1:1** dello script `FundingRateHeatmap_v8.py` per **verificare che i dati
combacino con Colab**.

A differenza dello scanner principale (che usa Binance/Bybit/OKX… e il funding *corrente*
aggregato), questa replica usa **esattamente** la pipeline del Python:

- coin: **CoinGecko top N** per market cap (variazioni 1y/30d/7d), stessa validazione simboli;
- exchange: **solo OKX · Hyperliquid · Bitget · HTX · Gate.io**, **storico** funding;
- conversione **APR** con stima dell'intervallo dai timestamp (`to_apr`);
- merge multi-exchange per **media sullo stesso timestamp**, poi **media giornaliera**;
- **fallback automatico** del periodo: 365 → 180 → 90 giorni;
- classificazione identica: **OPPORTUNITÀ / ESPLOSA / POMPATA / DELUSA / WATCH / STABILE**
  (soglie APR 30% medio / 20% recente, reazione prezzo 15%, min 3 giorni di dati).

Due **export CSV** per il confronto con Colab:
- **Esporta analisi CSV** — stesse colonne di `analysis_df`
  (`symbol,name,status,apr_medio,apr_90gg,apr_30gg,apr_picco,volatilita,pr_1a,pr_30gg,pr_7gg,prezzo`);
- **Esporta DataFrame CSV** — la matrice `coin × data` dei funding APR giornalieri
  (equivalente a `fund_df.to_csv()`), per il confronto cella per cella.

> Se i numeri dello scanner principale e di Colab non combaciavano, è perché usano fonti e
> metriche diverse: per il confronto con Colab usa **questa** scheda.

---

## Metriche

- **Funding APR** = `funding_rate × (24 / intervallo_ore) × 365 × 100`
  (intervallo 8h per i CEX, 1h per Hyperliquid).
- **Persistenza** = % di valori di funding positivi nello storico del periodo.
- **Δ Volume / Δ OI** = variazione % tra la media della **prima metà** e la **seconda metà**
  della serie temporale (positiva = trend crescente).

---

## Struttura del progetto

```
FundingRateScan.sln
src/
  Program.cs
  app.manifest
  Models/Models.cs                 # config, parametri, dati di mercato, risultati
  Services/
    HttpHelper.cs                  # HttpClient condiviso + parsing JSON
    SettingsManager.cs             # persistenza impostazioni
    ScanService.cs                 # orchestrazione: fetch → aggrega → filtra → arricchisci → classifica
    Exchanges/
      IExchangeClient.cs           # interfaccia comune + util simboli
      BinanceClient.cs  BybitClient.cs  OkxClient.cs
      BitgetClient.cs   GateioClient.cs HtxClient.cs  HyperliquidClient.cs
  Forms/
    MainForm.cs                    # menu, toolbar, indicatori, tabella, log
    DownloadForm.cs                # finestra di scaricamento (parametri + API)
    CoinDetailForm.cs              # dettaglio coin: dati numerici + schede heatmap
    HeatmapControl.cs              # heatmap GDI+ (stile Coinglass) con tooltip e barra colore
    OnChainForm.cs                 # OnChain Capital Flow Monitor (movimenti, whale, net flow, alert)
    OnChainSettingsForm.cs         # chiavi provider on-chain e soglie
  Colab/                           # replica 1:1 di FundingRateHeatmap_v8.py (confronto Colab)
    ColabModels.cs  ColabCoinGecko.cs  ColabFetchers.cs  ColabService.cs
  OnChain/                         # sottosistema on-chain (modulare, adapter pattern)
    Models/                        # modelli, enum, impostazioni, config monitor
    Providers/                     # IOnChainDataProvider + Etherscan/Ethplorer/DEXScreener/EVM
    Services/                      # WalletClassifier, FlowAggregator, OnChainMonitor (sync job)
    Scoring/                       # FlowSignalService (punteggio esplicabile)
    Alerts/                        # AlertService
    Repositories/                  # OnChainStore (persistenza JSON)
```

## Estendere con un nuovo exchange

1. Crea una classe in `src/Services/Exchanges/` che implementa `IExchangeClient`.
2. Aggiungi il nome a `SettingsManager.SupportedExchanges`.
3. Registralo nello `switch` del costruttore di `ScanService`.

---

## Note

- Alcuni exchange possono essere **geo-bloccati** in determinati paesi: in tal caso quel
  client restituisce 0 contratti e viene semplicemente ignorato (lo si vede nel log).
- L'app è uno strumento di **analisi/ricerca**, non esegue ordini né tocca fondi.
