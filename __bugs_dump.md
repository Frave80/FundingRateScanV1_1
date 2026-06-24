bugFundingRateScanV1
Documento di Debug e Verifica
Scoperta dei bug, lista strutturata e correzioni — parte implementata di FundingRateScan
Destinatario: programmatore / Claude Opus
Compito: individuare i bug elencati (e ulteriori), riprodurli, correggerli e verificarli
Ambito: C# · WinForms · .NET 8 · Visual Studio 2022
Versione documento: V1
Giugno 2026
## Scopo e modalità d'uso
Questo documento è un piano di debug operativo per la parte implementata di FundingRateScan, ossia: il nucleo funding (Parti I), il modulo di scoperta meme coin (Parte II) e i moduli aggiuntivi realizzati oltre il piano (On-Chain Capital Flow Monitor, Caccia gemme + PRO, Replica Python/Colab, Aree decisionali, navigazione a icone, collegamenti di quotazione, documentazione incorporata).
Il documento non riscrive il programma e non aggiunge funzionalità: serve a trovare e correggere i difetti di ciò che esiste già. È strutturato come una lista di bug candidati — ipotesi di difetto ad alta probabilità, ricavate dall'architettura descritta e dalle classi di errore tipiche di applicazioni WinServer/HTTP asincrone — ciascuna con sintomo, causa probabile, procedura di riproduzione e correzione consigliata.
Le parti già dichiarate corrette nel documento tecnico (ordine di docking di MainForm; layout e massimizzazione di MemeDiscoveryForm) non vanno reintrodotte come difetti: vanno solo verificate come regressioni potenziali.
## Indice
In Word: clic destro sull'indice → «Aggiorna campo» → «Aggiorna intero sommario».
## 1. Componenti sotto esame
La tabella mappa tutti i componenti implementati che rientrano nell'ambito del debug, con il file di riferimento e l'area funzionale. I codici bug del cap. 5+ si riferiscono a questi componenti.
### 1.1 Nucleo funding (Parte I)
### 1.2 Modulo meme (Parte II) e moduli aggiuntivi
## 2. Severità, priorità e ambiente di test
### 2.1 Scala di severità
Ogni bug è classificato per severità. La severità guida l'ordine di intervento: prima i bug che impediscono l'uso o falsano i risultati, poi quelli estetici.
### 2.2 Ambiente di test
## 3. Checklist sistematiche di ispezione
Prima di affrontare la lista dei bug candidati, eseguire queste checklist trasversali. Ciascuna voce è un punto da verificare in tutto il codice della parte implementata; spuntarla solo dopo l'ispezione effettiva.
### 3.1 Concorrenza e async/await
 - async void: presente solo negli handler di eventi? Ogni altro metodo asincrono restituisce Task. Un async void in un metodo non-evento rende le eccezioni non catturabili e può chiudere il processo.
 - ConfigureAwait: nei metodi di libreria/servizio (ScanService, client) le await usano ConfigureAwait(false)? In caso contrario, rischio di deadlock se mai chiamati in modo bloccante.
 - Aggiornamento UI da thread non-UI: ogni scrittura su controlli (tabella, log, status bar) dai task di scansione passa per Invoke/BeginInvoke o per un contesto di sincronizzazione? Altrimenti InvalidOperationException intermittente.
 - Accesso condiviso alle collezioni: le strutture aggregate per asset sono protette quando più task exchange scrivono in parallelo? Dictionary non è thread-safe.
 - Annullamento: il pulsante «Interrompi» propaga un CancellationToken fino alle chiamate HTTP? Senza, l'interruzione non ferma i download in corso.
### 3.2 Rete e parsing HTTP
 - HttpClient riusato: esiste un'istanza condivisa (o IHttpClientFactory) anziché un new HttpClient() per chiamata? Il pattern errato esaurisce le socket (SocketException sotto carico).
 - Timeout e retry: ogni chiamata ha timeout esplicito e gestione del 429 (rate limit) con backoff? Gli exchange limitano aggressivamente.
 - Codici non-200: le risposte 4xx/5xx e i corpi vuoti producono un risultato vuoto gestito, non un'eccezione che abortisce l'intera scansione?
 - Parsing difensivo: i campi JSON assenti o null sono gestiti? Un campo OI mancante su un exchange non deve far saltare l'aggregazione dell'asset.
 - InvariantCulture nel parsing: double.Parse / decimal.Parse usano CultureInfo.InvariantCulture su tutti i valori numerici provenienti dalle API?
### 3.3 Logica numerica e classificazione
 - Annualizzazione del funding: il fattore di conversione APR rispetta la cadenza di settlement di ogni exchange (8h, 4h, 1h)? Un fattore fisso falsa il confronto fra exchange con cadenze diverse.
 - Divisioni per zero: i ΔVolume e ΔOI in percentuale gestiscono il denominatore zero (volume/OI iniziale assente)? Rischio di Infinity/NaN che si propaga al punteggio.
 - NaN/Infinity nello score: il punteggio composito è protetto da valori non finiti? Un NaN ordina in modo imprevedibile e rompe il ranking.
 - Soglie e confini: i confronti usano ≥/≤ in modo coerente con la documentazione? Un > al posto di ≥ esclude i casi esattamente sulla soglia.
 - Coerenza dei segni: funding negativo (affollamento short) è trattato correttamente e non confuso con l'assenza di dato (0)?
### 3.4 UI WinForms e risorse
 - Dispose dei GDI+: in HeatmapControl e ScoreBreakdownControl, ogni Pen/Brush/Font/Bitmap creato è in un using o disposto? Le perdite GDI+ degradano e poi crashano il rendering.
 - Ridisegno e flicker: i controlli custom usano DoubleBuffered? Il ridisegno della heatmap a ogni resize non deve sfarfallare.
 - Coordinate e divisioni: il calcolo della cella della heatmap gestisce larghezze 0 (controllo non ancora dimensionato) senza DivideByZeroException?
 - Event handler doppi: gli handler non vengono agganciati più volte (es. ricostruzione UI) causando esecuzioni multiple?
 - Dispose delle Form: le finestre figlie (Detail, Discovery) vengono chiuse e disposte; i timer e i CancellationTokenSource fermati alla chiusura?
### 3.5 Persistenza, cache e file
 - Percorsi scrivibili: SettingsManager e MemeHolderStore scrivono in cartelle utente scrivibili (AppData), non accanto all'eseguibile in Program Files (accesso negato)?
 - Chiave di cache: la cache della heatmap per periodo (1/2/6/12 mesi) usa una chiave che include il periodo e il simbolo? Altrimenti mostra i dati di un'altra coin/periodo.
 - TTL e invalidazione: la cache ha scadenza? Dati stale possono mostrare un funding non più valido.
 - Concorrenza su file: scritture simultanee dello snapshot holder sono serializzate per evitare file corrotti?
 - Segreti in chiaro: le API key sono salvate in modo non leggibile a occhio (almeno DPAPI)? Verificare che non finiscano nei log.
## 4. Pattern di errore ricorrenti da cercare
Questi sono gli anti-pattern più probabili in un'app di questo tipo. Per ciascuno: come riconoscerlo nel codice e perché è pericoloso. Servono a guidare la ricerca di bug non elencati esplicitamente.
## 5. Bug candidati — Nucleo funding (BUG-CORE)
Lista di difetti ad alta probabilità nel nucleo funding (Parte I). Ogni scheda è un'ipotesi da confermare nel codice reale del componente indicato.
BUG-CORE-001  Race condition nell'aggregazione multi-exchange
Sintomo. Conteggi di coin variabili fra scansioni identiche; occasionale eccezione durante l'aggregazione; volumi/OI sommati in modo incoerente.
Causa probabile. I task per i diversi exchange scrivono in parallelo nella stessa struttura di aggregazione per asset (es. Dictionary) senza sincronizzazione; Dictionary non è thread-safe in scrittura concorrente.
Riproduzione:
 - Abilitare tutti gli exchange e avviare la scansione su un universo ampio (top N alto).
 - Ripetere la scansione più volte e confrontare il numero di asset aggregati e i totali volume/OI.
 - Osservare variazioni non spiegabili o un'eccezione sporadica.
Correzione consigliata:
 - Far produrre a ogni task una collezione locale e unire i risultati in un solo punto dopo Task.WhenAll, oppure usare ConcurrentDictionary con aggiornamenti atomici (AddOrUpdate).
 - Verificare che la somma di volume/OI e la media del funding siano calcolate dopo il join, non durante la scrittura concorrente.
BUG-CORE-002  Aggiornamento di tabella/log da thread di lavoro
Sintomo. Eccezione cross-thread (InvalidOperationException) intermittente durante la scansione, oppure UI che non si aggiorna fino al termine.
Causa probabile. Le continuazioni dei task di scansione scrivono direttamente su DataGridView/log/status bar senza marshalling sul thread UI.
Riproduzione:
 - Avviare una scansione lunga e osservare il log durante l'avanzamento.
 - Su alcune macchine compare l'eccezione; su altre l'UI resta congelata fino alla fine.
Correzione consigliata:
 - Incanalare ogni aggiornamento UI tramite IProgress<T> o Control.BeginInvoke.
 - Confermare che il corpo della scansione giri su un thread di lavoro (Task.Run) e che solo le notifiche tornino sul thread UI.
BUG-CORE-003  Istanze multiple di HttpClient ed esaurimento socket
Sintomo. Dopo scansioni ripetute o ampie, errori di connessione (SocketException / «only one usage of each socket address»), rallentamenti progressivi.
Causa probabile. Creazione di un nuovo HttpClient per ogni richiesta (tipicamente in un using), che lascia socket in TIME_WAIT ed esaurisce le porte effimere.
Riproduzione:
 - Eseguire 5–10 scansioni complete consecutive con molti exchange.
 - Monitorare le connessioni (netstat) e osservare l'accumulo di socket in attesa.
Correzione consigliata:
 - Usare una singola istanza statica di HttpClient riusata, o IHttpClientFactory.
 - Impostare timeout espliciti e header di default una sola volta.
BUG-CORE-004  Assenza di backoff sul rate limit (HTTP 429)
Sintomo. Durante scansioni ampie alcuni exchange restituiscono dati parziali o zero coin; il log mostra errori sporadici; risultati incompleti senza spiegazione.
Causa probabile. Le risposte 429 (Too Many Requests) non sono gestite con attesa/backoff; le richieste successive vengono respinte a catena.
Riproduzione:
 - Avviare la scansione senza chiavi API (limiti bassi) su top N elevato e molti exchange.
 - Osservare il calo di copertura su uno o più exchange.
Correzione consigliata:
 - Riconoscere il codice 429 (e header Retry-After dove presente) e applicare backoff esponenziale con jitter.
 - Limitare la concorrenza per-exchange (SemaphoreSlim) per restare sotto i limiti.
BUG-CORE-005  Annualizzazione del funding con fattore non coerente alla cadenza
Sintomo. Il funding APR di coin presenti su exchange con cadenze diverse (8h vs 1h) appare incoerente; alcune coin sembrano avere APR irrealisticamente alti o bassi rispetto ad altre.
Causa probabile. Conversione del funding periodico in APR con un fattore fisso (es. ×3×365) senza tener conto della reale cadenza di settlement dello specifico exchange/contratto.
Riproduzione:
 - Confrontare il funding APR della stessa coin su due exchange con cadenze diverse.
 - Verificare se il rapporto fra i due valori riflette la differenza di cadenza anziché il funding reale.
Correzione consigliata:
 - Derivare il fattore di annualizzazione dalla cadenza effettiva dichiarata dall'exchange (o dall'intervallo fra timestamp consecutivi dello storico).
 - Centralizzare la conversione in un'unica funzione documentata e testarla con casi noti.
BUG-CORE-006  Divisione per zero nei ΔVolume / ΔOI percentuali
Sintomo. Alcune coin mostrano variazioni «∞» o «NaN»; la classificazione di quelle coin è imprevedibile; l'ordinamento per ΔVol/ΔOI si comporta in modo anomalo.
Causa probabile. Calcolo della variazione percentuale come (corrente − iniziale) / iniziale con valore iniziale pari a zero o assente (coin di nuova quotazione, storico mancante).
Riproduzione:
 - Includere nella scansione coin con storico volume/OI assente o nullo all'inizio della finestra.
 - Osservare i valori di ΔVol/ΔOI e lo stato assegnato.
Correzione consigliata:
 - Trattare il denominatore zero come dato non disponibile (escludere dalla variazione) anziché produrre Infinity/NaN.
 - Validare i valori finiti prima della classificazione e del ranking.
BUG-CORE-007  Parsing numerico dipendente dalla cultura di sistema
Sintomo. Su macchine con impostazioni italiane (virgola decimale) i funding/volumi risultano errati di ordini di grandezza, o il parsing fallisce silenziosamente; in cultura inglese tutto appare corretto.
Causa probabile. double.Parse/decimal.Parse (o Convert.ToDouble) senza CultureInfo.InvariantCulture sui valori JSON, che gli exchange forniscono con punto decimale.
Riproduzione:
 - Impostare Windows su Italiano (virgola decimale) e avviare una scansione.
 - Confrontare i valori con quelli ottenuti in cultura inglese.
Correzione consigliata:
 - Forzare CultureInfo.InvariantCulture in tutti i parse dei valori provenienti dalle API.
 - Per la formattazione UI usare la cultura corrente, ma mai per il parsing dei dati grezzi.
BUG-CORE-008  Cache dello storico non invalidata o con chiave incompleta
Sintomo. Riaprendo una scansione o cambiando periodo, vengono mostrati dati non aggiornati; talvolta i dati di una coin compaiono su un'altra.
Causa probabile. Chiave di cache che non include tutti i discriminanti (simbolo + periodo + finestra) oppure assenza di TTL.
Riproduzione:
 - Eseguire una scansione, attendere, rieseguirla e verificare se i dati cambiano quando dovrebbero.
 - Aprire il dettaglio di due coin in sequenza e confrontare gli storici.
Correzione consigliata:
 - Comporre la chiave di cache con simbolo, periodo e finestra temporale; aggiungere un TTL.
 - Invalidare la cache all'avvio di una nuova scansione esplicita.
BUG-CORE-009  Blocco o nomi mancanti quando CoinGecko è irraggiungibile
Sintomo. La colonna «Nome» resta vuota o l'avvio della scansione rallenta vistosamente quando CoinGecko è lento o irraggiungibile.
Causa probabile. Chiamata sincrona/bloccante senza timeout o senza fallback al simbolo; mancato caching della mappa simbolo→nome.
Riproduzione:
 - Simulare CoinGecko non raggiungibile (rete assente o dominio bloccato) e avviare la scansione.
 - Osservare ritardi o nomi vuoti.
Correzione consigliata:
 - Applicare timeout breve e fallback immediato al simbolo come nome.
 - Memorizzare la mappa e rinfrescarla in background, senza bloccare la scansione.
BUG-CORE-010  Export CSV con separatori e decimali ambigui
Sintomo. Il CSV esportato si apre male in Excel su sistemi italiani (colonne unite o numeri interpretati come testo).
Causa probabile. Uso della virgola sia come separatore di campo sia come separatore decimale (cultura IT), senza quoting né scelta coerente del delimitatore.
Riproduzione:
 - Esportare i risultati in cultura italiana e aprire il file in Excel.
 - Verificare la corretta separazione delle colonne e dei decimali.
Correzione consigliata:
 - Usare un delimitatore non ambiguo (punto e virgola) o quotare i campi, e formattare i numeri in modo coerente.
 - In alternativa, scrivere i numeri in InvariantCulture e dichiarare il separatore.
BUG-CORE-011  «Interrompi» non ferma i download in corso
Sintomo. Premendo Interrompi l'UI torna disponibile ma la rete continua a lavorare; i risultati possono arrivare dopo l'interruzione e sovrascrivere lo stato.
Causa probabile. CancellationToken non propagato alle chiamate HTTP o non verificato fra uno stadio e l'altro.
Riproduzione:
 - Avviare una scansione lunga e premere Interrompi a metà.
 - Monitorare la rete e osservare se le richieste proseguono.
Correzione consigliata:
 - Propagare un CancellationToken fino alle SendAsync e controllarlo all'inizio di ogni stadio.
 - Ignorare/scartare i risultati che arrivano dopo l'annullamento.
BUG-CORE-012  Eccezione di un singolo exchange aborta l'intera scansione
Sintomo. Se un exchange è geo-bloccato o cambia formato di risposta, la scansione complessiva fallisce invece di degradare.
Causa probabile. Mancanza di isolamento per-exchange: un'eccezione non catturata in un client si propaga al coordinatore.
Riproduzione:
 - Forzare il fallimento di un client (dominio bloccato) e avviare la scansione.
 - Verificare se gli altri exchange producono comunque risultati.
Correzione consigliata:
 - Isolare ogni client in un try/catch che logga e restituisce risultato vuoto, senza propagare.
 - Mostrare nel log quali exchange hanno fallito e perché.
## 6. Bug candidati — UI, dettaglio e heatmap (BUG-UI)
Difetti probabili nelle finestre e nei controlli grafici (DownloadForm, CoinDetailForm, HeatmapControl).
BUG-UI-001  Perdita di handle GDI+ nel rendering della heatmap
Sintomo. Dopo molte aperture del dettaglio o molti ridisegni, la heatmap smette di disegnarsi, l'app rallenta o lancia un'eccezione di risorse GDI esaurite.
Causa probabile. Pen/Brush/Font/Bitmap creati nel metodo di disegno senza Dispose (mancanza di using); gli handle GDI+ non vengono rilasciati.
Riproduzione:
 - Aprire e chiudere il dettaglio coin decine di volte, alternando i periodi 1/2/6/12 mesi.
 - Monitorare gli oggetti GDI del processo (Task Manager → colonna oggetti GDI) e osservarne la crescita.
Correzione consigliata:
 - Avvolgere ogni risorsa GDI+ in using, o riusare risorse statiche dove possibile.
 - Verificare che eventuali Bitmap di buffer siano disposte a ogni ridisegno.
BUG-UI-002  DivideByZero o celle invisibili a controllo non dimensionato
Sintomo. Eccezione all'apertura del dettaglio in certi layout, oppure heatmap vuota finché non si ridimensiona la finestra.
Causa probabile. Calcolo della dimensione cella dividendo per larghezza/altezza del controllo quando queste valgono 0 (controllo non ancora dimensionato al primo Paint).
Riproduzione:
 - Aprire il dettaglio coin su risoluzione bassa o con la finestra inizialmente minimizzata.
 - Osservare l'eccezione o l'assenza di disegno.
Correzione consigliata:
 - Verificare Width/Height > 0 e numero di colonne/righe > 0 prima di disegnare; uscire dal Paint altrimenti.
 - Forzare un Invalidate dopo il primo Resize valido.
BUG-UI-003  Tooltip o cella associata alla data errata (off-by-one)
Sintomo. Il tooltip mostra una data/valore spostati di un giorno rispetto alla cella sotto il cursore; i mesi con 28/30/31 giorni si disallineano.
Causa probabile. Indice giorno calcolato senza tener conto del primo giorno della settimana/mese, o conteggio dei giorni del mese fisso a 31.
Riproduzione:
 - Passare il mouse su celle note (es. primo e ultimo giorno del mese) e confrontare il tooltip con i dati reali.
 - Verificare febbraio e mesi da 30 giorni.
Correzione consigliata:
 - Calcolare i giorni reali del mese (DateTime.DaysInMonth) e mappare l'indice cella → data in modo esplicito.
 - Aggiungere test sulla corrispondenza cella→data per mesi di diversa lunghezza.
BUG-UI-004  Scala colore degenerata quando min ≈ max o tutti i valori uguali
Sintomo. La heatmap appare tutta di un colore o la barra colore mostra 0–0; con un solo valore o valori costanti il gradiente non si calcola.
Causa probabile. Normalizzazione (v − min)/(max − min) con max = min → divisione per zero o gradiente nullo.
Riproduzione:
 - Aprire il dettaglio di una coin con storico funding piatto o quasi costante.
 - Osservare la barra colore e le celle.
Correzione consigliata:
 - Gestire il caso max = min con un colore neutro e una barra colore degradante a default.
 - Imporre un intervallo minimo della scala per evitare divisioni instabili.
BUG-UI-005  Cambio rapido di periodo mostra i dati del periodo precedente
Sintomo. Cliccando velocemente 1→6→12 mesi, la heatmap può mostrare il risultato di una richiesta precedente (race tra fetch on-demand).
Causa probabile. Le richieste asincrone per periodo non sono serializzate né annullate; l'ultima risposta che arriva vince, non l'ultima richiesta.
Riproduzione:
 - Aprire il dettaglio e cambiare periodo rapidamente più volte.
 - Verificare se il grafico mostrato corrisponde al periodo selezionato per ultimo.
Correzione consigliata:
 - Annullare la richiesta precedente al cambio periodo (CancellationToken) e applicare solo la risposta del periodo correntemente selezionato.
 - Confrontare il periodo della risposta con quello attuale prima di disegnare.
BUG-UI-006  Parametri fuori range o non numerici accettati senza validazione
Sintomo. Inserendo soglie negative, percentuali assurde o testo nei campi numerici, la scansione parte con valori incoerenti o lancia un'eccezione di parsing.
Causa probabile. Mancanza di validazione/clamp dei campi prima di avviare la scansione.
Riproduzione:
 - Inserire valori limite (negativi, zero, enormi, testo) e premere Avvia.
 - Osservare il comportamento.
Correzione consigliata:
 - Validare e limitare ogni campo (range plausibili) e segnalare gli errori prima di avviare.
 - Disabilitare il pulsante Avvia finché gli input non sono validi.
BUG-UI-007  API secret/passphrase non mascherati o salvati in chiaro
Sintomo. I campi segreto/passphrase mostrano il testo in chiaro, o le credenziali risultano leggibili nel file di impostazioni.
Causa probabile. PasswordChar non impostato sui campi sensibili e/o salvataggio senza protezione (DPAPI).
Riproduzione:
 - Inserire credenziali e verificarne la visualizzazione; ispezionare il file di impostazioni salvato.
Correzione consigliata:
 - Impostare il mascheramento sui campi sensibili e proteggere i segreti a riposo (DPAPI/ProtectedData).
 - Assicurarsi che i segreti non finiscano nei log.
## 7. Bug candidati — Modulo meme (BUG-MEME)
Difetti probabili nel modulo di scoperta meme coin (IMemeSignalSource, MemeSignals, GemBackedMemeSignalSource, MarketMetaClient, MemeScoringService, MemeDiscoveryForm, ScoreBreakdownControl).
BUG-MEME-001  Copertura parziale gestita come zero anziché esclusa dalla media
Sintomo. Coin con dati on-chain mancanti ricevono punteggi artificialmente bassi (o alti) e si posizionano in modo fuorviante nel ranking.
Causa probabile. Una dimensione priva di dati viene conteggiata come 0 nella media pesata, invece di essere esclusa e ridurre la copertura, come previsto dal documento (cap. 10.4).
Riproduzione:
 - Eseguire la scoperta su token con dati holder/liquidità assenti (es. senza chiave Etherscan).
 - Verificare se il punteggio riflette i soli dati disponibili o viene penalizzato dallo zero implicito.
Correzione consigliata:
 - Escludere le dimensioni senza dati dalla media pesata, rinormalizzando i pesi su quelle disponibili.
 - Esporre e mostrare la copertura (percentuale di dimensioni con dati) come previsto.
BUG-MEME-002  Penalità di rischio fuori dall'intervallo [0,1]
Sintomo. Alcune coin ottengono punteggi negativi o superiori al massimo; il ranking presenta valori incoerenti.
Causa probabile. Il fattore di penalità moltiplicativo non è limitato a [0,1] (cap. 9.9); somma di penalità non vincolata può portare il moltiplicatore sotto 0 o sopra 1.
Riproduzione:
 - Costruire/selezionare una coin con più flag di rischio simultanei (concentrazione + liquidità non bloccata + mortalità).
 - Osservare il punteggio risultante.
Correzione consigliata:
 - Vincolare la penalità in [0,1] (clamp) e documentarne la composizione.
 - Validare che lo score finale resti in 0–100.
BUG-MEME-003  Risposte parziali di GoPlus/Honeypot/DEX Screener non gestite
Sintomo. Eccezioni o segnali errati quando un provider restituisce campi mancanti, schema diverso o errore; alcuni token spariscono o ottengono flag errati.
Causa probabile. Accesso a campi JSON senza controllo di null/assenza; assenza di fallback quando un provider non risponde.
Riproduzione:
 - Eseguire la scoperta quando uno dei provider è lento/irraggiungibile o restituisce un token senza alcune metriche.
 - Osservare eccezioni o segnali anomali.
Correzione consigliata:
 - Parsing difensivo con valori opzionali; marcare il segnale come «non disponibile» anziché assumere un default rischioso.
 - Isolare ogni provider: il fallimento di uno non deve compromettere gli altri segnali della coin.
BUG-MEME-004  Crescita holder errata al primo run o su snapshot mancante
Sintomo. Il segnale di crescita holder mostra valori assurdi (enormi o nulli) alla prima scansione o quando manca lo snapshot precedente.
Causa probabile. Calcolo della crescita rispetto a uno snapshot inesistente trattato come zero, oppure divisione per il conteggio precedente assente.
Riproduzione:
 - Cancellare lo store degli snapshot ed eseguire la prima scoperta.
 - Osservare i valori di crescita holder.
Correzione consigliata:
 - Al primo snapshot, segnalare «baseline» senza calcolare una crescita; calcolarla solo dal secondo in poi.
 - Gestire denominatore zero/assente come dato non disponibile.
BUG-MEME-005  Falso «nuovo listing» per assenza di baseline o reset cache
Sintomo. Il flag catalizzatore «nuovo listing» scatta su molte coin al primo avvio o dopo la pulizia della cache, generando falsi positivi.
Causa probabile. Rilevamento del nuovo listing per differenza rispetto a uno stato precedente che, se assente, viene interpretato come «comparso ora».
Riproduzione:
 - Avviare la scoperta dopo aver svuotato lo stato persistito e contare i flag di nuovo listing.
Correzione consigliata:
 - Distinguere «nessuno stato precedente» da «listing effettivamente nuovo»; sopprimere il flag alla baseline.
BUG-MEME-006  Breakdown non somma al totale o barre fuori scala
Sintomo. La somma visiva dei contributi non corrisponde al punteggio mostrato; barre che escono dall'area o si sovrappongono.
Causa probabile. Contributi disegnati senza normalizzazione alla copertura/peso effettivi, o scala basata su un massimo errato.
Riproduzione:
 - Aprire il breakdown per una coin a copertura parziale e confrontare la somma dei contributi con lo score.
Correzione consigliata:
 - Disegnare i contributi coerentemente con la stessa normalizzazione del punteggio finale (inclusa la rinormalizzazione per copertura).
 - Limitare le barre all'area e gestire i valori non finiti.
BUG-MEME-007  «Scopri e valuta» rilanciabile durante l'esecuzione
Sintomo. Premendo più volte «Scopri e valuta» partono scansioni sovrapposte; risultati mescolati o eccezioni; «Interrompi» non sempre efficace.
Causa probabile. Il pulsante non viene disabilitato durante l'esecuzione e non esiste un solo CancellationTokenSource gestito correttamente.
Riproduzione:
 - Avviare la scoperta e premere ripetutamente il pulsante; poi premere Interrompi.
Correzione consigliata:
 - Disabilitare «Scopri e valuta» durante l'esecuzione e riabilitarlo al termine/annullamento.
 - Gestire un unico CancellationTokenSource per esecuzione, fermato da Interrompi.
BUG-MEME-008  Verifica regressione: massimizzazione e spaziatura barra filtri
Sintomo. (Verifica) La finestra non si apre massimizzata, o lo spazio fra «Scopri e valuta», «Max età» e «Interrompi» torna non uniforme.
Causa probabile. Le correzioni registrate nel changelog (12.6) potrebbero regredire se i controlli a monte cambiano dimensione e il posizionamento non resta a cascata.
Riproduzione:
 - Aprire MemeDiscoveryForm e verificare WindowState = Maximized.
 - Misurare i gap fra i controlli della barra filtri (devono essere uniformi, 8 px).
Correzione consigliata:
 - Confermare che il posizionamento resti a cascata (.Right + gap) e non torni a coordinate fisse.
## 8. Bug candidati — Moduli aggiuntivi (BUG-EXTRA)
Difetti probabili nei moduli realizzati oltre il piano: On-Chain Capital Flow Monitor, Caccia gemme + PRO (100x), Replica Python/Colab, Aree decisionali/Priorità, navigazione a icone, collegamenti di quotazione, documentazione incorporata.
BUG-EXTRA-001  Degrado non gestito senza chiave Etherscan
Sintomo. Senza chiave Etherscan, il monitor resta vuoto, lancia errori ripetuti o blocca l'UI, invece di degradare mostrando i soli dati disponibili.
Causa probabile. Assenza di percorso di fallback quando la chiave manca o il rate limit gratuito è superato; errori non incanalati.
Riproduzione:
 - Avviare il monitor on-chain senza chiave configurata.
 - Osservare se i dati di holder/mercato restano comunque disponibili (come dichiarato) o se tutto si blocca.
Correzione consigliata:
 - Rilevare l'assenza di chiave e mostrare i dati ottenibili dagli altri provider, con avviso non bloccante.
 - Applicare backoff sul rate limit gratuito.
BUG-EXTRA-002  Score esplosivo e moltiplicatore non finiti o non normalizzati
Sintomo. Token con metriche estreme producono score «∞»/«NaN» o moltiplicatori 100x irrealistici che dominano il ranking.
Causa probabile. Combinazione multi-metrica con divisioni non protette o senza normalizzazione/clamp; outlier non gestiti.
Riproduzione:
 - Eseguire la caccia gemme sul feed DEX Screener includendo token a liquidità/volume minimi o estremi.
 - Osservare gli score e i moltiplicatori risultanti.
Correzione consigliata:
 - Normalizzare e limitare ogni metrica; proteggere le divisioni; validare la finitezza dello score.
 - Applicare un cap ragionevole al moltiplicatore e segnalare gli outlier anziché lasciarli dominare.
BUG-EXTRA-003  Esito dei controlli di sicurezza ambiguo in caso di provider muto
Sintomo. Quando GoPlus/Honeypot.is non rispondono, un token rischioso può apparire «sicuro» (o viceversa), perché l'assenza di risposta è trattata come esito positivo.
Causa probabile. Mancata distinzione fra «controllo superato» e «controllo non eseguibile»; default permissivo.
Riproduzione:
 - Eseguire i controlli con i provider di sicurezza irraggiungibili.
 - Verificare l'etichetta di sicurezza assegnata.
Correzione consigliata:
 - Introdurre uno stato esplicito «non verificato» distinto da «sicuro»; default prudente (non promuovere).
 - Mostrare chiaramente nel dossier quali controlli non è stato possibile eseguire.
BUG-EXTRA-004  Disallineamento numerico fra export CSV e DataFrame Colab
Sintomo. Confrontando l'export con Colab, i valori non coincidono o il CSV non si carica correttamente in pandas (decimali/separatori).
Causa probabile. Formattazione numerica dipendente dalla cultura nell'export, o intestazioni/ordini colonna non allineati alla replica.
Riproduzione:
 - Esportare l'analisi e il DataFrame e caricarli in Colab/pandas.
 - Confrontare valori, colonne e tipi.
Correzione consigliata:
 - Esportare in InvariantCulture con separatore esplicito e intestazioni stabili.
 - Allineare l'ordine e i nomi delle colonne alla replica di riferimento.
BUG-EXTRA-005  Z-score multi-fattore instabile con campione piccolo o varianza nulla
Sintomo. Il ranking accademico cross-sezionale produce valori estremi o NaN quando ci sono poche coin o una metrica è costante.
Causa probabile. Calcolo dello z-score con deviazione standard zero (varianza nulla) o n troppo piccolo.
Riproduzione:
 - Eseguire la classificazione su pochissime coin o su un insieme con una metrica costante.
 - Osservare gli z-score e il ranking.
Correzione consigliata:
 - Gestire deviazione standard zero (assegnare z=0) e imporre una numerosità minima per il ranking.
 - Escludere dal calcolo le metriche degeneri segnalandolo.
BUG-EXTRA-006  Doppio clic apre la pagina di quotazione sbagliata o un URL malformato
Sintomo. Il doppio clic sul simbolo apre la sede errata (DEX Screener per una coin affermata o viceversa), o un link rotto se il simbolo contiene caratteri speciali.
Causa probabile. Euristica nuova-vs-affermata fragile e mancata codifica del simbolo nell'URL; fallback TradingView non sempre attivato.
Riproduzione:
 - Doppio clic su una coin affermata e su una nuova; provare simboli con caratteri inusuali.
Correzione consigliata:
 - Rendere robusta la scelta della sede (criterio esplicito) e codificare l'URL; verificare il fallback a TradingView.
BUG-EXTRA-007  Icone della toolbar sfocate o tagliate a scaling elevato
Sintomo. A 150%/200% di scaling le icone GDI+ appaiono sfocate, disallineate o tagliate; i separatori non si vedono.
Causa probabile. Disegno a dimensioni fisse senza adattamento al DPI corrente.
Riproduzione:
 - Avviare l'app a 150% e 200% di scaling e osservare la toolbar.
Correzione consigliata:
 - Scalare le icone e le spaziature in base al DPI; impostare un manifest DPI-aware coerente.
BUG-EXTRA-008  Apertura/scaricamento del documento incorporato fallisce in sola lettura
Sintomo. Dal menu Aiuto, l'apertura del documento Word incorporato fallisce se la cartella dell'eseguibile è in sola lettura (Program Files).
Causa probabile. Estrazione del documento accanto all'eseguibile invece che in una cartella temporanea/utente scrivibile.
Riproduzione:
 - Installare/eseguire l'app da una cartella in sola lettura e usare la voce di menu per aprire il documento.
Correzione consigliata:
 - Estrarre la risorsa in una cartella temporanea/utente scrivibile e aprirla da lì.
## 9. Workflow operativo per il programmatore / Opus
Procedura passo-passo per affrontare il debug in modo ordinato e verificabile. Seguire le fasi nell'ordine: ogni fase produce un esito registrabile.
### 9.1 Fasi
 - Preparazione. Aprire la soluzione in VS 2022, compilare in Debug e Release, annotare tutti i warning. Avviare l'app e verificare l'avvio pulito.
 - Ispezione statica. Eseguire le ricerche testuali del cap. 4 (new HttpClient, .Result, double.Parse, new Pen, catch {}) e marcare le occorrenze.
 - Conferma dei candidati. Per ogni BUG-* dei cap. 5–8, aprire il componente indicato e confermare o smentire l'ipotesi nel codice reale.
 - Riproduzione. Per i bug confermati, riprodurre con la procedura indicata; catturare log/eccezione/screenshot.
 - Correzione. Applicare la correzione consigliata (o una migliore), mantenendo invariate le interfacce pubbliche dove possibile.
 - Verifica. Rieseguire la riproduzione: il sintomo deve sparire. Rieseguire i test di accettazione del cap. 11.
 - Registrazione. Aggiornare la tabella di tracciamento (cap. 10) con stato, causa radice effettiva, file e righe modificate.
 - Regressione. Verificare che le correzioni note (docking MainForm; layout MemeDiscoveryForm) non siano regredite.
### 9.2 Regole di intervento
 - Non distruttività: non modificare il comportamento documentato corretto; correggere solo i difetti. Le interfacce IExchangeClient e IMemeSignalSource non cambiano firma.
 - Una causa, una correzione: evitare correzioni che mascherano il sintomo senza rimuovere la causa (es. try/catch che ingoia invece di gestire).
 - Test prima di chiudere: nessun bug è «risolto» senza una verifica riproducibile e una build pulita.
 - Priorità per severità: affrontare prima Critico e Alto; Medio e Basso a seguire.
## 10. Tabella di tracciamento dei bug
Da compilare durante il debug. Per ogni bug: stato (Aperto / In corso / Risolto / Non riproducibile / Non un bug), causa radice effettiva trovata nel codice, e file/righe modificate. La colonna «Conferma» indica se l'ipotesi del documento è stata confermata.
Legenda Conferma: «Sì» = ipotesi confermata nel codice; «No» = non presente; «Variante» = difetto reale diverso da quello ipotizzato.
## 11. Test di accettazione
Suite minima da superare a fine debug. Ogni test ha un esito atteso; registrare Pass/Fail.
### 11.1 Build e avvio
### 11.2 Scansione funding
### 11.3 Dettaglio e heatmap
### 11.4 Modulo meme e aggiuntivi
## 12. Riepilogo e priorità d'intervento
Sintesi della lista bug per severità. L'ordine consigliato di lavorazione segue la severità decrescente; entro la stessa severità, prima i bug che falsano i risultati, poi quelli che impattano stabilità e UX.

[TABELLA 1]
Mandato per il programmatore / Opus 1. Trattare ogni voce della lista come ipotesi da verificare nel codice reale: confermare o smentire ispezionando la classe indicata. 2. Per ogni bug confermato: riprodurlo, individuare la causa radice, applicare la correzione, e registrare l'esito nella colonna «Stato» della tabella di tracciamento (cap. 12). 3. Cercare anche bug non elencati: le checklist (cap. 3) e i pattern di errore (cap. 4) guidano l'ispezione sistematica. 4. Criterio di chiusura: build pulita (0 warning, 0 errori), avvio senza eccezioni, e superamento dei test di accettazione del cap. 11.

[TABELLA 2]
Componente | File | Area
Motore di scansione | ScanService.cs | Orchestrazione: download, aggregazione, filtro, arricchimento, classificazione
Contratto exchange | IExchangeClient.cs | Interfaccia dati comune
Client exchange | Binance/Bybit/Okx/Bitget/ Gateio/Htx/Hyperliquid Client.cs | Accesso REST a funding/volume/OI
Infrastruttura HTTP | HttpHelper.cs | Retry, header, timeout
Impostazioni | SettingsManager.cs | Persistenza chiavi/parametri
Nomi coin | CoinNameProvider.cs | Mappa simbolo→nome (CoinGecko)
UI principale | MainForm.cs | Menu, toolbar, indicatori, tabella, log
UI download | DownloadForm.cs | Parametri e credenziali
UI dettaglio | CoinDetailForm.cs | Pannello numerico + schede
Controllo heatmap | HeatmapControl.cs | Rendering GDI+ calendario

[TABELLA 3]
Componente | File | Area
Contratto segnali meme | src/Meme/IMemeSignalSource.cs | Liquidità, holder, concentrazione, partecipazione, sopravvivenza
Modelli + punteggio | src/Meme/MemeSignals.cs | Modelli, score composito, stati ★◆◇✕
Fonte segnali | GemBackedMemeSignalSource.cs | DEX Screener, GoPlus, Honeypot.is
Meta di mercato | MarketMetaClient.cs | CoinGecko: community/dev score, n. exchange
Scoring | src/Meme/MemeScoringService.cs | Pesi, penalità, copertura
UI scoperta | src/Forms/MemeDiscoveryForm.cs | Tabella per punteggio, barra filtri
Controllo breakdown | src/Forms/ScoreBreakdownControl.cs | GDI+: contributo dimensioni
Capital Flow Monitor | (modulo On-Chain) | Etherscan/Ethplorer/DEX Screener: whale, flussi, alert
Caccia gemme + PRO (100x) | (modulo Gem Hunter) | Feed DEX Screener, score esplosivo, sicurezza contratto
Replica Python (Colab) | (modulo Replica) | Ricostruzione FundingRateHeatmap, export CSV
Aree decisionali / Priorità | (modulo Priorità) | Volatilità attesa, z-score multi-fattore
Persistenza holder | MemeHolderStore | Snapshot del numero di holder fra scansioni

[TABELLA 4]
Nota sui nomi dei file dei moduli aggiuntivi Il documento tecnico nomina alcuni moduli aggiuntivi per funzione e non per file esatto. Dove il nome file non è esplicitato, il programmatore lo individui nella soluzione (cartelle src/Forms e src/Meme, o equivalenti) e annoti il percorso reale nella tabella di tracciamento.

[TABELLA 5]
Severità | Definizione | Esempi
Critico | Crash, blocco dell'UI, perdita di dati, o risultati gravemente errati che inducono decisioni sbagliate. | Eccezione non gestita; classificazione invertita
Alto | Funzione importante non opera o produce dati inaffidabili in casi comuni. | Aggregazione errata; cache stale
Medio | Difetto evidente con impatto limitato o aggirabile. | Tooltip errato; ordinamento incoerente
Basso | Cosmetico o marginale, senza impatto sui risultati. | Allineamento; etichetta imprecisa

[TABELLA 6]
Aspetto | Riferimento
IDE | Visual Studio 2022 (Community o superiore)
Target | net8.0-windows  (compila anche con SDK .NET 9/10)
Configurazioni | Build sia in Debug sia in Release; verificare i warning solo-Release
Rete | Test con connessione piena, con rete assente, e con un exchange geo-bloccato
Chiavi API | Test senza chiavi (solo market-data pubblici) e con chiave Etherscan configurata
Risoluzione/DPI | Test a 100%, 150% e 200% di scaling, e su risoluzione bassa (1366×768)
Cultura di sistema | Test con cultura IT (virgola decimale) e EN (punto decimale): vedere BUG-CORE-007

[TABELLA 7]
Il test della cultura è prioritario Le applicazioni .NET che fanno parsing di JSON numerico dagli exchange e poi formattano per l'UI sono storicamente fragili rispetto alla cultura di sistema: un funding «0.0123» può essere letto come 123 in cultura italiana se il parsing non forza InvariantCulture. Questo difetto è insidioso perché invisibile sulle macchine in cultura EN. Eseguire l'intera suite anche con Impostazioni internazionali italiane.

[TABELLA 8]
Anti-pattern | Come riconoscerlo | Rischio
new HttpClient() per chiamata | Istanza creata dentro un metodo di fetch, in un using. | Esaurimento socket; SocketException sotto carico.
Parse senza InvariantCulture | double.Parse(s) senza secondo argomento. | Numeri errati in cultura IT; risultati falsati.
.Result / .Wait() | Bloccaggio sincrono di un Task in un handler UI. | Deadlock dell'UI.
UI da thread di lavoro | Assegnazioni a controlli dentro Task.Run o continuazioni. | InvalidOperationException cross-thread.
catch (...) {} | Blocchi catch vuoti che ingoiano l'errore. | Bug silenziosi; dati mancanti senza traccia.
Divisione percentuale senza guardia | (b-a)/a con a possibile 0. | Infinity/NaN nel punteggio e nel ranking.
GDI+ non disposto | new Pen/Brush/Font senza using. | Perdita di handle; crash del rendering nel tempo.
Fattore APR fisso | Stesso moltiplicatore per tutti gli exchange. | Funding non confrontabile fra cadenze diverse.
Chiave di cache incompleta | Cache indicizzata solo per simbolo, non per periodo. | Heatmap che mostra il periodo sbagliato.
Aggregazione su Dictionary non sincronizzato | Scrittura concorrente da task paralleli. | Dati persi o eccezioni intermittenti.
Confronto float con == | Uguaglianza esatta tra valori in virgola mobile. | Confronti che non scattano mai.
Ordinamento con valori null/NaN | Sort su colonne con celle vuote o non finite. | Ordine instabile; eccezioni nel comparer.

[TABELLA 9]
Come usare questo capitolo Per ogni anti-pattern, eseguire una ricerca testuale nella soluzione (es. cercare new HttpClient, .Result, double.Parse, new Pen, catch). Ogni occorrenza è un candidato da valutare. Molti dei bug del capitolo 5 derivano direttamente da questi pattern: confermarli nel codice reale prima di correggere.

[TABELLA 10]
Severità | Componente | Tipo
Critico | ScanService.cs | Concorrenza / aggregazione

[TABELLA 11]
Severità | Componente | Tipo
Critico | MainForm.cs | Threading UI

[TABELLA 12]
Severità | Componente | Tipo
Alto | HttpHelper.cs | Rete / risorse

[TABELLA 13]
Severità | Componente | Tipo
Alto | client exchange | Rete / rate limit

[TABELLA 14]
Severità | Componente | Tipo
Alto | client exchange | Logica numerica

[TABELLA 15]
Severità | Componente | Tipo
Critico | ScanService.cs | Logica numerica

[TABELLA 16]
Severità | Componente | Tipo
Alto | client exchange / MainForm.cs | Cultura / parsing

[TABELLA 17]
Severità | Componente | Tipo
Medio | ScanService.cs | Cache

[TABELLA 18]
Severità | Componente | Tipo
Medio | CoinNameProvider.cs | Rete / fallback

[TABELLA 19]
Severità | Componente | Tipo
Medio | MainForm.cs | Export / cultura

[TABELLA 20]
Severità | Componente | Tipo
Medio | ScanService.cs / MainForm.cs | Annullamento

[TABELLA 21]
Severità | Componente | Tipo
Basso | client exchange | Robustezza

[TABELLA 22]
Severità | Componente | Tipo
Alto | HeatmapControl.cs | GDI+ / risorse

[TABELLA 23]
Severità | Componente | Tipo
Medio | HeatmapControl.cs | Logica grafica

[TABELLA 24]
Severità | Componente | Tipo
Medio | HeatmapControl.cs | Mappatura dati

[TABELLA 25]
Severità | Componente | Tipo
Medio | HeatmapControl.cs | Scala cromatica

[TABELLA 26]
Severità | Componente | Tipo
Medio | CoinDetailForm.cs | Async / cache

[TABELLA 27]
Severità | Componente | Tipo
Basso | DownloadForm.cs | Validazione input

[TABELLA 28]
Severità | Componente | Tipo
Basso | DownloadForm.cs | Persistenza segreti

[TABELLA 29]
Severità | Componente | Tipo
Critico | MemeScoringService.cs | Logica punteggio

[TABELLA 30]
Severità | Componente | Tipo
Alto | MemeScoringService.cs | Logica numerica

[TABELLA 31]
Severità | Componente | Tipo
Alto | GemBackedMemeSignalSource.cs | Parsing / null

[TABELLA 32]
Severità | Componente | Tipo
Alto | MemeHolderStore | Persistenza / crescita holder

[TABELLA 33]
Severità | Componente | Tipo
Medio | MarketMetaClient.cs | Logica listing

[TABELLA 34]
Severità | Componente | Tipo
Medio | ScoreBreakdownControl.cs | GDI+ / dati

[TABELLA 35]
Severità | Componente | Tipo
Medio | MemeDiscoveryForm.cs | Threading / annullamento

[TABELLA 36]
Severità | Componente | Tipo
Basso | MemeDiscoveryForm.cs | Regressione layout

[TABELLA 37]
Severità | Componente | Tipo
Alto | On-Chain Capital Flow Monitor | Rete / chiave API

[TABELLA 38]
Severità | Componente | Tipo
Alto | Caccia gemme + PRO (100x) | Logica numerica

[TABELLA 39]
Severità | Componente | Tipo
Alto | Caccia gemme + PRO (100x) | Sicurezza contratto

[TABELLA 40]
Severità | Componente | Tipo
Medio | Replica Python (Colab) | Export / cultura

[TABELLA 41]
Severità | Componente | Tipo
Medio | Aree decisionali / Priorità | Statistica

[TABELLA 42]
Severità | Componente | Tipo
Basso | Collegamenti di quotazione | Routing / URL

[TABELLA 43]
Severità | Componente | Tipo
Basso | Navigazione a icone | UI / DPI

[TABELLA 44]
Severità | Componente | Tipo
Basso | Documentazione incorporata | Risorse / IO

[TABELLA 45]
Criterio di «fatto» Un bug è chiuso quando: la causa radice è documentata, la correzione è applicata, la riproduzione non mostra più il sintomo, la build è pulita (0 warning / 0 errori) e i test di accettazione pertinenti passano. Solo allora lo stato in tabella diventa «Risolto».

[TABELLA 46]
ID | Sev. | Stato | Conferma | Causa radice / file modificati
BUG-CORE-001 | Critico | ☐ Aperto |  | 
BUG-CORE-002 | Critico | ☐ Aperto |  | 
BUG-CORE-003 | Alto | ☐ Aperto |  | 
BUG-CORE-004 | Alto | ☐ Aperto |  | 
BUG-CORE-005 | Alto | ☐ Aperto |  | 
BUG-CORE-006 | Critico | ☐ Aperto |  | 
BUG-CORE-007 | Alto | ☐ Aperto |  | 
BUG-CORE-008 | Medio | ☐ Aperto |  | 
BUG-CORE-009 | Medio | ☐ Aperto |  | 
BUG-CORE-010 | Medio | ☐ Aperto |  | 
BUG-CORE-011 | Medio | ☐ Aperto |  | 
BUG-CORE-012 | Basso | ☐ Aperto |  | 
BUG-UI-001 | Alto | ☐ Aperto |  | 
BUG-UI-002 | Medio | ☐ Aperto |  | 
BUG-UI-003 | Medio | ☐ Aperto |  | 
BUG-UI-004 | Medio | ☐ Aperto |  | 
BUG-UI-005 | Medio | ☐ Aperto |  | 
BUG-UI-006 | Basso | ☐ Aperto |  | 
BUG-UI-007 | Basso | ☐ Aperto |  | 
BUG-MEME-001 | Critico | ☐ Aperto |  | 
BUG-MEME-002 | Alto | ☐ Aperto |  | 
BUG-MEME-003 | Alto | ☐ Aperto |  | 
BUG-MEME-004 | Alto | ☐ Aperto |  | 
BUG-MEME-005 | Medio | ☐ Aperto |  | 
BUG-MEME-006 | Medio | ☐ Aperto |  | 
BUG-MEME-007 | Medio | ☐ Aperto |  | 
BUG-MEME-008 | Basso | ☐ Aperto |  | 
BUG-EXTRA-001 | Alto | ☐ Aperto |  | 
BUG-EXTRA-002 | Alto | ☐ Aperto |  | 
BUG-EXTRA-003 | Alto | ☐ Aperto |  | 
BUG-EXTRA-004 | Medio | ☐ Aperto |  | 
BUG-EXTRA-005 | Medio | ☐ Aperto |  | 
BUG-EXTRA-006 | Basso | ☐ Aperto |  | 
BUG-EXTRA-007 | Basso | ☐ Aperto |  | 
BUG-EXTRA-008 | Basso | ☐ Aperto |  | 

[TABELLA 47]
# | Test | Esito atteso
T1 | Compilazione in Debug e Release | 0 errori, 0 warning
T2 | Avvio dell'applicazione | Nessuna eccezione; finestra principale visibile, prima riga non coperta dal menu
T3 | Avvio con rete assente | Avvio regolare; messaggio di assenza dati, nessun crash

[TABELLA 48]
# | Test | Esito atteso
T4 | Scansione con tutti gli exchange | Risultati popolati; nessuna race; log coerente
T5 | Scansione con un exchange geo-bloccato | Gli altri exchange producono risultati; il fallito è loggato
T6 | Cultura italiana (virgola decimale) | Valori numerici identici alla cultura inglese
T7 | Spostamento indicatori a soglia | Riclassificazione immediata senza riscaricare
T8 | Coin con storico iniziale nullo | Nessun Infinity/NaN; stato coerente
T9 | «Interrompi» a metà scansione | Download fermati; stato non sovrascritto da risultati tardivi
T10 | Export CSV e riapertura in Excel (IT) | Colonne e decimali corretti

[TABELLA 49]
# | Test | Esito atteso
T11 | Aperture ripetute del dettaglio (≥50) | Nessuna crescita anomala di oggetti GDI; nessun degrado
T12 | Cambio rapido di periodo 1/2/6/12 mesi | La heatmap mostra sempre il periodo selezionato per ultimo
T13 | Coin con funding piatto/costante | Scala colore e barra gestite senza errori
T14 | Tooltip su primo/ultimo giorno e febbraio | Data e valore corrispondono alla cella

[TABELLA 50]
# | Test | Esito atteso
T15 | Scoperta senza chiave Etherscan | Dati holder/mercato/sociale disponibili; nessun blocco
T16 | Coin a copertura parziale | Score sui soli dati presenti; copertura mostrata
T17 | Coin con più flag di rischio | Penalità in [0,1]; score finale in 0–100
T18 | Primo run senza snapshot holder | Baseline senza crescita assurda
T19 | MemeDiscoveryForm | Apertura massimizzata; barra filtri a spaziatura uniforme
T20 | Provider di sicurezza muti (gem PRO) | Stato «non verificato» distinto da «sicuro»
T21 | Doppio clic su coin nuova vs affermata | Apertura della sede corretta; URL valido

[TABELLA 51]
Severità | Conteggio | ID principali
Critico | 4 | BUG-CORE-001, BUG-CORE-002, BUG-CORE-006, BUG-MEME-001
Alto | 11 | BUG-CORE-003/004/005/007, BUG-UI-001, BUG-MEME-002/003/004, BUG-EXTRA-001/002/003
Medio | 13 | BUG-CORE-008/009/010/011, BUG-UI-002/003/004/005, BUG-MEME-005/006/007, BUG-EXTRA-004/005
Basso | 9 | BUG-CORE-012, BUG-UI-006/007, BUG-MEME-008, BUG-EXTRA-006/007/008

[TABELLA 52]
I tre rischi sistemici da chiudere per primi 1. Concorrenza (BUG-CORE-001/002): aggregazione e UI sotto carico parallelo — falsa i risultati e causa crash intermittenti, difficili da diagnosticare dopo. 2. Robustezza numerica (BUG-CORE-006, BUG-MEME-001/002): divisioni per zero, NaN e copertura parziale — corrompono punteggi e ranking, cioè l'output stesso del programma. 3. Cultura e parsing (BUG-CORE-007): errori invisibili sulle macchine EN ma gravi su quelle IT — vanno chiusi prima di qualunque rilascio in Italia.

[TABELLA 53]
Nota finale Questa lista è un punto di partenza ad alta probabilità, non un inventario esaustivo: i bug reali vanno confermati nel codice. Le checklist (cap. 3) e i pattern (cap. 4) servono proprio a scoprire i difetti non elencati. A debug concluso, la tabella di tracciamento (cap. 10) e i test di accettazione (cap. 11) costituiscono la prova del lavoro svolto. Il software resta uno strumento di analisi probabilistica e non costituisce consulenza finanziaria.
