# Riconciliazione bancaria automatica

Script Python che automatizza la riconciliazione tra fatture emesse ed estratto conto bancario: il processo con cui un'azienda verifica quali fatture sono state pagate, da chi, e cosa resta in sospeso.

È un lavoro che nelle PMI viene ancora fatto a mano, incrociando due Excel riga per riga — ore ogni mese. Questo progetto simula il problema con dati realistici e lo risolve con un motore di matching a 4 livelli.

## Risultati

Su 53 movimenti bancari simulati:

- **83% riconciliato automaticamente** (44 movimenti)
- **9% pre-istruito per la verifica umana** (5 movimenti con suggerimento allegato)
- **8% correttamente segnalato come estraneo** (tasse, stipendi, commissioni bancarie)

Ogni movimento riceve una spiegazione: nessun caso resta senza esito.

## Il problema (perché non basta un confronto di importi)

Nella realtà l'estratto conto è "sporco":

- le causali sono scritte in mille modi ("FT 9", "SALDO FATT. 35/2026", "PAGAMENTO FORNITURA")
- alcuni bonifici arrivano con commissioni trattenute (importo che non torna mai esatto)
- un cliente può pagare 2-3 fatture con un unico bonifico cumulativo
- alcune fatture non vengono pagate affatto
- in mezzo ci sono movimenti che non c'entrano nulla (F24, stipendi, giroconti)

Il dataset simulato riproduce volutamente tutti questi casi.

## Come funziona: matching a 4 livelli

Il motore funziona a imbuto: ogni livello lavora solo sui movimenti non ancora matchati dal precedente, e una fattura già assegnata non può essere ricandidata.

**Livello 1 — Match esatto.** La causale contiene il numero della fattura e l'importo corrisponde al centesimo. Il numero viene estratto con una regex e validato contro le fatture esistenti: così un numero "estraneo" (come l'anno 2026, o il 24 di "F24") si scarta da solo perché nessuna fattura ha quel numero. Confidenza alta.

**Livello 2 — Importo e data.** La causale non dice nulla, ma esiste una fattura aperta con lo stesso importo e una data di pagamento compatibile con la scadenza (da 5 giorni prima a 30 dopo). Il match scatta solo se il candidato è unico: se due fatture aperte hanno lo stesso importo, il caso finisce nelle eccezioni. Confidenza media.

**Livello 3 — Tolleranza con prova.** L'importo è vicino ma non esatto (tipicamente commissioni bancarie trattenute, es. -2,50€). La tolleranza più larga è accettata solo se c'è una prova in più: il numero fattura o il nome del cliente in causale. La differenza viene annotata in una colonna dedicata, perché contabilmente quei 2,50€ sono un costo bancario da registrare, non uno scarto da ignorare.

**Livello 4 — Bonifici cumulativi.** Nessuna fattura da sola corrisponde all'importo, ma la somma di 2-3 fatture aperte dello stesso cliente sì. Il vincolo sul cliente (estratto dalla causale) riduce il problema combinatorio a poche combinazioni da testare.

## Il principio guida

**In caso di dubbio, non decidere.** In contabilità un match sbagliato è peggio di un match mancante: l'errore si propaga in silenzio, mentre un'eccezione costa due minuti di verifica umana. Per questo:

- i casi ambigui non vengono mai matchati automaticamente
- per i casi "quasi certi" lo script non decide ma **suggerisce**: allega alla riga il candidato più probabile con la differenza di importo, e lascia la decisione all'umano

Il risultato sono tre esiti possibili — match automatico, suggerimento da verificare, non riconosciuto — invece di un sì/no secco.

## Struttura del progetto

- `riconciliazione.ipynb` — notebook completo: generazione dati simulati + motore di matching
- `fatture.csv` — 60 fatture simulate (numero, cliente, importo, emissione, scadenza)
- `estratto_conto.csv` — 53 movimenti bancari simulati con causali realistiche

I dati sono generati con seed fisso: chiunque cloni il repo ottiene esattamente gli stessi dati e gli stessi risultati.

## Tecnologie

Python, pandas, regex (`re`), `itertools.combinations`.

## Limiti noti ed estensioni possibili

- Il riconoscimento del cliente in causale è un confronto esatto sul nome: con nomi storpiati ("ROSSI COSTR." vs "Rossi Costruzioni SRL") servirebbe fuzzy matching (es. `rapidfuzz`). Non implementato perché i dati simulati non lo richiedono — regola: non complicare il codice per problemi che non si hanno ancora.
- La validazione dei numeri di fattura sfrutta il fatto che i numeri progressivi (1-60) non si confondono con altri numeri in causale (es. l'anno): con migliaia di fatture servirebbe un pattern più robusto.
- Il livello 4 testa combinazioni di 2-3 fatture: bonifici che coprono più fatture richiederebbero un approccio più generale.
- Estensione naturale: scheduling automatico (esecuzione periodica) e output Excel multi-foglio per il reparto amministrativo.

## Nota sul processo

Durante lo sviluppo, il livello 4 inizialmente non trovava nessun match. La diagnostica ha rivelato che il bug non era nel matching ma nei dati generati: i bonifici cumulativi raggruppavano fatture di clienti diversi, situazione impossibile nella realtà. È stato corretto il generatore, non il matching — quando test e codice sono in disaccordo, non è detto che sia il codice a sbagliare.
