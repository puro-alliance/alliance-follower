# WIP — Guida all'uso

> Che cos'è WIP, come funziona e che cosa permette di fare su un altro lavoro (job) IBM i in esecuzione.

**Stato**: strumento esistente, in fase di stabilizzazione. Il file di salvataggio con gli oggetti pronti
all'uso sarà pubblicato nella cartella [`oggetti/`](oggetti/), insieme alle istruzioni di installazione.

## Video

[![Tutorial video di WIP, 4 minuti](tutorial-wip.jpg)](tutorial-wip.mp4)

**[Guarda il tutorial (4 minuti)](tutorial-wip.mp4)**: l'elenco dei lavori e il comando dentro un lavoro, la QTEMP,
il blocco, il Service Job e la copia dello schermo, filmati su un terminale 5250 vero, su un sistema di prova.

## 1. Che cos'è

WIP («Work with other jobs», lavorare su altri lavori) è un'utility per IBM i che permette, da un normale
terminale 5250, di **intervenire dall'interno su un lavoro che sta girando**: un lavoro interattivo di un
collega, un batch, un lavoro di un'applicazione web.

Normalmente un lavoro IBM i si può solo guardare da fuori (`WRKJOB`, `DSPJOB`) o chiudere. Con WIP si può
entrare nel suo ambiente: eseguire un comando come se lo avesse digitato quel lavoro, leggere e modificare
i file della sua libreria `QTEMP`, vedere la schermata che ha davanti in quel momento.

È pensato per chi sviluppa e per chi fa assistenza: per capire che cosa sta succedendo in un lavoro senza
doverlo fermare e rilanciare.

## 2. Che cosa permette di fare

| Funzione | In breve |
|---|---|
| **Comando nel lavoro** | Esegue un comando CL dentro il lavoro scelto, nel suo ambiente (lista librerie, `QTEMP`, file aperti, sostituzioni di file). |
| **Copia e modifica della QTEMP** | Copia file e aree dati della `QTEMP` del lavoro in una libreria normale, li fa esaminare e modificare, e su richiesta li riporta nella `QTEMP` del lavoro. |
| **Copia dello schermo** | Mostra sul proprio terminale la schermata che un lavoro interattivo ha davanti in quel momento. |
| **Blocco del lavoro** | Tiene fermo il lavoro in un punto, a disposizione, per eseguirvi più operazioni di seguito. |
| **Service Job** | Avvia il servizio sul lavoro (per esempio per il debug) e apre una riga comandi; all'uscita lo chiude. |
| **Chiusura e dettaglio** | Chiude il lavoro o apre la gestione standard del lavoro. |

## 3. Come funziona

WIP usa una funzione standard del sistema operativo: l'API **Call Job Interrupt Program** (`QWCJBITP`),
che chiede a un lavoro in esecuzione di sospendere per un momento quello che sta facendo ed eseguire un
programma, per poi riprendere da dove era.

```
 Il vostro terminale                          Il lavoro scelto
 ┌──────────────────┐   1. interruzione      ┌──────────────────────┐
 │ WIP              │ ─────────────────────► │ sospende il programma│
 │ (elenco lavori,  │                        │ ed esegue WIPSVR:    │
 │  opzioni)        │   2. risposta          │ comando, QTEMP,      │
 │                  │ ◄───────────────────── │ schermo              │
 └──────────────────┘   (coda dati)          │ poi riprende         │
                                             └──────────────────────┘
```

1. Scegliete un lavoro e un'opzione. WIP chiede al sistema di interrompere quel lavoro.
2. Il lavoro esegue il programma di servizio di WIP (`WIPSVR`), che svolge l'operazione richiesta
   **dentro** il lavoro e manda l'esito su una coda dati.
3. WIP legge l'esito e ve lo mostra. Il lavoro riprende il suo programma dal punto in cui era.

Tre cose da sapere, perché sono regole del sistema operativo e non di WIP:

- Il programma di servizio va **registrato** una volta nel punto di uscita `QIBM_QWC_JOBITPPGM`
  (lo fa il comando `WIPINIT`).
- Il valore di sistema `QALWJOBITP` deve permettere l'interruzione dei lavori (valore 2).
  `WIPINIT` lo controlla e avvisa se non è così.
- L'operazione viene eseguita nel lavoro scelto, ma **con il profilo utente di chi usa WIP**, non con
  quello del lavoro: non si ottengono autorizzazioni che non si hanno già.

## 4. Requisiti

- IBM i 7.3 o successivo.
- La libreria di WIP nella lista librerie di chi lo usa.
- Per lavorare su lavori di altri utenti serve l'autorizzazione speciale di controllo dei lavori
  (`*JOBCTL`); sui propri lavori non serve.
- Installazione una tantum: ripristino della libreria e comando `WIPINIT`, lanciato da un profilo
  autorizzato a registrare programmi di uscita (di norma il responsabile del sistema).

## 5. Avvio e schermata principale

Si lancia con il comando `WIP`. Compare l'elenco dei lavori attivi:

```
                    Gestione lavori dall'interno                      SISTEMA
                                                             18/11/26  12:30:15
 Immettere opzioni, premere Invio.
 4=Chiudi 5=Lavoro L=Blocca U=Sblocca C=Comando Q=QTEMP S=Schermo J=Service Job
Opz  Sottosist.   Lavoro       Utente       Numero   Tipo  Stato Funzione
     __________   __________   __________   ______   ___   ____
 _   QINTER       QPADEV0003   ROSSI        123456   INT   DSPW  PGM-ORD100
 _   QBATCH       FATTURE      BIANCHI      123460   BCH   RUN   PGM-FAT200
 _   QUSRWRK      QZDASOINIT   QUSER        123470   PJ    TIMW

Cmd==> ______________________________________________________________________

 F3=Fine F12=Indietro
```

- **Filtri**: la riga sotto le intestazioni serve a restringere l'elenco. Si scrive l'inizio del
  sottosistema, del nome del lavoro, dell'utente, del numero, del tipo o dello stato e si preme Invio.
- **Riga comandi** (`Cmd==>`): esegue un comando nel **proprio** lavoro, come una normale riga comandi.
- **Tasti**: F3 e F12 escono, F5 rilegge l'elenco, F4 al posto di Invio apre la richiesta parametri
  del comando (vale per la riga comandi e per le opzioni 4 e 5).

## 6. Le funzioni, una per una

### C — Comando nel lavoro

Si apre una finestra in cui scrivere un comando CL (fino a 210 caratteri). Premendo Invio il comando
viene eseguito **dentro il lavoro scelto**. La finestra resta aperta per altri comandi; F3 o F12 la chiude.

Il comando vede l'ambiente di quel lavoro: la sua lista librerie, la sua `QTEMP`, i file che ha aperti,
le sostituzioni di file (`OVRDBF`) in vigore. Esempi d'uso:

- copiare in una libreria normale un file di lavoro che esiste solo nella `QTEMP` del lavoro;
- cambiare un'area dati o creare un oggetto che il programma si aspetta di trovare;
- controllare la lista librerie o le sostituzioni di file effettive, quando un programma legge il file
  sbagliato.

In risposta WIP mostra `Ok` oppure l'avviso che il comando ha dato errore (il dettaglio è nel joblog
del lavoro scelto).

> Attenzione: un comando che apre una schermata la apre **sul terminale di quel lavoro**, non sul vostro.

### Q — Copia e modifica della QTEMP

La libreria `QTEMP` è privata di ogni lavoro: dall'esterno non si vede. Con l'opzione Q:

1. WIP copia i file e le aree dati della `QTEMP` del lavoro in una libreria normale, chiamata
   `QTMP` seguito dal numero del lavoro (per esempio `QTMP123456`).
2. Apre la gestione oggetti (PDM) su quella libreria: potete guardare i dati, interrogarli, modificarli.
3. All'uscita chiede: **«Desideri aggiornare la QTEMP del lavoro originale?»**
   - **N** (valore proposto): la `QTEMP` del lavoro resta com'è.
   - **Y**: file e aree dati della `QTEMP` del lavoro vengono sostituiti con quelli della libreria.

Cose da sapere:

- vengono copiati solo file e aree dati;
- se la copia riesce solo in parte, WIP lo segnala;
- la sostituzione cancella i file della `QTEMP` e li ricrea: un file che il lavoro tiene aperto in quel
  momento non può essere sostituito, e WIP lo segnala;
- le librerie `QTMP...` create in una sessione vengono cancellate da WIP alla chiusura.

### S — Copia dello schermo

Vale per i lavori interattivi. WIP mostra sul vostro terminale la schermata che quel lavoro ha davanti
in quel momento: utile in assistenza, quando l'utente dice «mi dà un errore» e si vuole vedere che cosa
vede lui.

- F5 aggiorna la schermata;
- F3 torna all'elenco.

La copia è una fotografia dell'istante in cui si sceglie l'opzione: se l'utente continua a lavorare, sul vostro
terminale resta quella schermata. Per vedere quella nuova si preme F5, che interrompe di nuovo il lavoro e rifà la copia.

Ogni schermata copiata viene anche registrata nel file di log di WIP (`WIP_LOG`): tenetene conto se le
schermate contengono dati riservati.

### L e U — Blocca e sblocca il lavoro

Normalmente ogni opzione interrompe il lavoro per un istante e lo lascia subito riprendere. Con **L** il
lavoro viene invece **tenuto fermo**, a disposizione di WIP, finché non lo si sblocca con **U** o non si
esce da WIP. Serve per eseguire più operazioni di seguito sullo stesso stato: più comandi, una copia della
`QTEMP` e poi un comando, e così via.

- Si può bloccare un lavoro per volta; il lavoro bloccato è evidenziato nell'elenco e indicato in alto.
- Mentre un lavoro è bloccato, le opzioni C, Q e S valgono solo su quel lavoro.
- Chi sta usando quel lavoro lo vede fermo finché non viene sbloccato.
- Sicurezza: se la sessione di WIP si interrompe, il lavoro riparte da solo dopo una decina di secondi.

### J — Service Job

Avvia il servizio sul lavoro scelto (comando `STRSRVJOB`) e apre una finestra con una riga comandi, da cui
lanciare per esempio il debug (`STRDBG`) e i comandi collegati. I comandi di questa finestra girano nel
proprio lavoro. F4 apre la richiesta parametri.

F3 o F12 chiude il servizio (`ENDSRVJOB`) e torna all'elenco. Se il servizio non si può chiudere, di solito
perché il debug è ancora attivo, WIP lo segnala e resta nella finestra: si chiude il debug e si ripreme F3.

### 4 e 5 — Chiudi il lavoro, lavora con il lavoro

- **5** apre la gestione standard del lavoro (`WRKJOB`).
- **4** chiude il lavoro (`ENDJOB`). **Non chiede conferma**: per controllare i parametri prima
  dell'esecuzione, premere F4 al posto di Invio.

## 7. Avvertenze

WIP agisce su lavori veri, spesso di altre persone. Va usato sapendo che:

- **Non tutti i lavori si possono interrompere.** Il sistema lo vieta per i lavori di sistema, i monitor
  di sottosistema, i lettori e gli scrittori di spool, i lavori in fase di avvio o di chiusura e i lavori
  che si sono dichiarati non interrompibili. In questi casi WIP risponde con un messaggio di errore.
- **La risposta può non arrivare.** WIP aspetta la risposta del lavoro per 10 secondi; se il lavoro non
  arriva a un punto in cui può essere interrotto, compare «nessuna risposta dal lavoro».
- **Il lavoro viene sospeso a metà di quello che stava facendo.** I dati su cui stava lavorando possono
  essere in uno stato intermedio: un comando che li modifica può creare incoerenze. Sugli ambienti di
  produzione è prudente limitarsi a leggere.
- **Ciò che si esegue resta nel joblog del lavoro scelto**, non nel proprio.
- **L'accesso va limitato.** Chi può usare WIP può eseguire comandi dentro i lavori su cui ha il
  controllo: la libreria va resa disponibile solo ai profili che devono usarla.

## 8. Che cosa c'è nella libreria

| Oggetto | Tipo | A che cosa serve |
|---|---|---|
| `WIP` | comando | Avvia WIP. |
| `WIPINIT` | comando | Installazione: registra il programma di servizio e controlla il valore di sistema. |
| `WIPWSR` | programma | L'elenco dei lavori e le opzioni. |
| `WIPWSV` | file video | Le schermate. |
| `WIPCL` | programma | Invia la richiesta al lavoro scelto e ne attende la risposta. |
| `WIPSVR` | programma | Il programma di servizio: è quello che gira dentro il lavoro scelto. |
| `WIPQT`, `WIPQX`, `WIPXQ` | programmi | Creazione della libreria `QTMP...`, copia dalla `QTEMP`, copia verso la `QTEMP`. |
| `WIP_LOG` | file | Registro delle schermate copiate. |

## 9. In arrivo

- Registrazione delle schermate e delle azioni di un lavoro.
- Avvio diretto su un lavoro indicato, senza passare dall'elenco.

## Vedi anche

- [Interrupt applicativo](../../../booster/utility/interrupt-applicativo/) — l'interruzione controllata dei programmi applicativi.
- [Osservare un job IBM i dall'esterno](../../../expert/kb/osservare-job-ibmi/) — leggere memoria e stato di un lavoro in esecuzione.
