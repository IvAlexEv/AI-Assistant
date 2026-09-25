---
name: recon-code
description: >-
  Fase di ricognizione (recon) in modalità white-box su un target web/API in-scope, concentrata sull'analisi statica del CODICE SORGENTE dell'applicazione. Controlla se è presente la cartella `code` con il sorgente e, in tal caso, lo usa direttamente per estrarre funzionalità, architettura, stack, configurazione, endpoint/route, gestione auth e superfici d'attacco, partendo SEMPRE anche dalla cartella RESULT e dalla cronologia Burp per correlare codice e comportamento a runtime.
  Usare quando il tester chiede un recon white-box / analisi del codice / source review iniziale, o quando è disponibile il sorgente dell'applicazione da mappare.
  Produce un file di stato consolidato in RESULT da poter usare nelle altre skill.
---

# Skill: recon_code (white-box)

Ricognizione ordinata e accurata di un target web/API **in-scope** partendo dal **codice sorgente**, per comprendere l'applicazione e tutte le sue funzionalità e per raccogliere le informazioni che serviranno alle skill successive. Al termine produce un **file di stato** in `RESULT`.

Questa skill segue le regole del `CLAUDE.md` del progetto. In particolare: solo target in-scope, tutte le interazioni verso il target passano dai tool MCP di Burp, e se manca un'informazione **si chiede** invece di assumere.

> **Analisi statica.** Questa skill **legge** il codice, non lo esegue e non esegue test attivi sul target. Qualsiasi verifica a runtime (invio richieste, payload) esula da qui e segue le regole di conferma del `CLAUDE.md`.

---

## 0. Prima di iniziare

1. **Verifica scope.** Conferma che il target/applicazione sia tra quelli autorizzati nel `CLAUDE.md`. Se non lo è o è ambiguo → fermati e chiedi.
2. **Controlla la disponibilità del codice.**
   - Verifica se esiste la cartella `code` con il sorgente dell'applicazione.
   - **Se `code` è presente**: è la fonte primaria di questa skill, usala direttamente.
   - **Se `code` è assente o incompleta**: fermati e chiedi al tester dove si trova il sorgente (o se va usato un altro percorso). Non inventare struttura o contenuti.
3. **Leggi lo stato esistente.**
   - Controlla la cartella `RESULT`: se esiste già un file di recon (black-box o white-box) per questo target, **riprendi da lì** e integra, non ricominciare da zero. Se esiste un recon black-box, usalo per **correlare** codice e comportamento reale.
   - Leggi la cronologia Burp (`get_proxy_http_history` / `get_proxy_http_history_regex`, filtrando per host in-scope): utile per collegare route del codice a richieste reali.
4. **Raccogli il contesto mancante dal tester.** Se per analizzare bene servono cose non presenti nel codice, nel `RESULT` né nel `CLAUDE.md`, chiedile prima di partire, ad es.:
   - Quale parte del codice corrisponde al target in-scope (mono-repo, più servizi).
   - Framework/linguaggio se non evidente, come viene fatto il build/deploy.
   - Aree del codice fuori scope o da ignorare (librerie di terze parti, vendor).

---

## 1. Inquadramento del codice (prima passata)

Prima di scendere nel dettaglio, fatti un'idea d'insieme leggendo la struttura:

- **Linguaggi e framework**: dai file manifest e dalla struttura delle cartelle.
- **Punto d'ingresso** dell'applicazione (entrypoint, bootstrap, `main`, `index`, app).
- **Organizzazione**: layout del progetto (MVC, moduli, servizi), cartelle principali.
- **File di dipendenze**: `package.json`, `requirements.txt`, `pom.xml`, `composer.json`, `go.mod`, `Gemfile`, ecc. — per stack e versioni.
- **File di configurazione e deploy**: `.env`(.example), `config/`, `Dockerfile`, `docker-compose`, file del web server, CI/CD.

Questa è la prima fase per farsi già un'idea dell'applicazione, della sua architettura e della sua superficie.

---

## 2. Analisi sistematica del sorgente

Analizza il codice in modo **ordinato**, non a caso, coprendo:

1. **Routing / endpoint**: individua dove sono definite le route (file di routing, annotazioni/decoratori dei controller, framework) e ricava l'elenco completo di
   endpoint con metodo, path, handler associato.
2. **Controller / handler e logica**: per ogni endpoint, cosa fa, quali input riceve, quali funzioni/servizi richiama.
3. **Autenticazione e autorizzazione**: middleware, guard, filtri, gestione sessione/JWT, controlli di ruolo/permesso, dove sono applicati e dove **mancano**.
4. **Accesso ai dati**: query verso DB/ORM, chiamate a servizi esterni, gestione file, comandi di sistema, deserializzazione, template — i punti dove entra input.
5. **Configurazione e segreti**: parametri di sicurezza, header, CORS, cookie, feature flag; **credenziali/API key/segreti** hardcoded o nei file di config (segnalali, senza riprodurre valori sensibili più del necessario per identificarli).
6. **Dipendenze**: librerie e versioni, con attenzione a componenti datati o notoriamente vulnerabili (ricerca CVE out-of-band ammessa).

File di lavoro intermedi (mappe grezze, elenchi di route, note di parsing) vanno nella cartella `files`; il risultato consolidato va in `RESULT` (§4). Se qualcosa nel codice non è chiaro o manca un pezzo, **chiedi** invece di assumere.

---

## 3. Analisi e considerazioni

Consolida quanto raccolto e ragiona sull'applicazione:

- A cosa serve, flussi principali e funzionalità chiave, così come emergono dal codice.
- Come è strutturata e (dai file di config/deploy) come sembra essere configurata.
- Dove si concentrano le superfici d'attacco più interessanti a livello di codice (da approfondire con altre skill, senza test attivi qui).
- Correlazione con il comportamento reale (history Burp / recon black-box) dove possibile.
- Cosa resta scoperto o incerto (parti non lette, codice offuscato/minificato, dubbi).

---

## 4. Output: file di stato del recon (white-box)

Al termine produci **un unico file accurato** nella cartella `RESULT`, che rappresenta la memoria e lo stato attuale della fase di recon white-box e sarà la base per le skill successive.

- **Cartella**: `RESULT`
- **Nome suggerito**: `recon_code_<host-o-target>.md`
- Se un file di recon white-box per lo stesso target esiste già, **aggiornalo** mantenendo lo storico delle sezioni, invece di crearne uno nuovo scollegato.

Il file è diviso nei capitoli seguenti, **sempre in questo ordine e con questi titoli**:

1. **Data ultima modifica** — data (e ora) dell'ultimo aggiornamento del file.
2. **Descrizione applicazione** — cosa fa e a cosa serve, cosa contiene (funzionalità, sezioni, flussi principali) come emerge dal codice, ed eventuali considerazioni utili.
3. **Architettura e struttura del codice** — organizzazione del progetto, componenti/moduli/servizi principali, entrypoint, come sono divise le responsabilità.
4. **Autenticazione e autorizzazione** — meccanismo (sessione/JWT/…), dove è implementato, come sono applicati i controlli di ruolo/permesso e dove risultano assenti; credenziali di test note.
5. **Stack tecnologico e dipendenze** — il più completo possibile, ricavato da manifest, config e codice. Presentato in una **tabella riassuntiva** con, per ogni voce, versione (se nota) e *come* è stato dedotto; segnala versioni datate/vulnerabili.
6. **Endpoint / route** — **numero totale** e lista organizzata per area funzionale, in una **tabella chiara** con metodo HTTP, path, handler/file, scopo, parametri principali e se richiede autenticazione/ruolo.
7. **Configurazione e segreti** — parametri di sicurezza (header, CORS, cookie), impostazioni d'ambiente, e presenza di segreti/credenziali hardcoded (identificati, non esposti oltre il necessario).
8. **Altre informazioni emerse** — elenco puntato di qualsiasi altra cosa rilevata durante l'analisi che non rientri nei capitoli precedenti, così **nessuna informazione va persa** (commenti nel codice, TODO/FIXME, endpoint di debug, file dimenticati, comportamenti particolari, indizi, ecc.).
9. **Superfici d'attacco / punti di interesse** — elenco ragionato delle aree da approfondire con le altre skill, con una **prima considerazione** per ciascuna (con riferimento al file/funzione del codice). In questa fase **non** si eseguono test attivi: qui si annota soltanto e si fa un primo ragionamento in termini di sicurezza e superficie d'attacco.
10. **Stato del recon** — copertura raggiunta, cosa è stato analizzato, cosa resta da coprire, domande aperte e assunzioni dichiarate.

Mantieni il contenuto pulito, schematizzato, coerente e preciso.
Al termine comunica al tester dove hai salvato il file e una descrizione sul processo e sul suo andamento, se ha avuto problemi o se è stato eseguito correttamente.

> **Nota importante, completezza.** Il file deve essere il **più completo possibile**, con
> tutte le informazioni raccolte: l'obiettivo è non perdere nulla di ciò che è emerso durante
> l'analisi del codice. Può quindi essere **lungo quanto serve, senza limiti di lunghezza**.
> In caso di dubbio, includi l'informazione piuttosto che ometterla.

### Template della struttura (da seguire identico a ogni output)

```markdown
# Recon white-box — <target>

## 1. Data ultima modifica
<AAAA-MM-GG HH:MM>

## 2. Descrizione applicazione
- **Obiettivo:** <...>
- **Sezioni e funzionalità:**
  - <sezione/funzionalità> — <descrizione>
- **Considerazioni:** <...>

## 3. Architettura e struttura del codice
- **Linguaggi/framework:** <...>
- **Entrypoint:** <...>
- **Organizzazione:** <layout, moduli/servizi principali>
- **Componenti:**
  - <componente> — <ruolo> — <path>

## 4. Autenticazione e autorizzazione
- **Meccanismo:** <sessione / JWT / header custom / non determinato>
- **Dove è implementato:** <file/middleware>
- **Controlli di ruolo/permesso:** <come e dove; dove mancano>
- **Credenziali note:** <utente/ruolo → dove/come>
- **Considerazioni:** <...>

## 5. Stack tecnologico e dipendenze
| Componente | Tecnologia e Versione | Come è stato dedotto | Note (datato/vulnerabile) |
|------------|-----------------------|----------------------|---------------------------|
| <es. Framework> | <...> | <es. package.json> | <...> |
| <es. DB/ORM>    | <...> | <...> | <...> |
| <es. Libreria>  | <...> | <...> | <...> |

## 6. Endpoint / route
**Numero totale:** <N>

### <Area funzionale>
| Metodo | Path | Handler/File | Scopo | Parametri principali | Auth/Ruolo |
|--------|------|--------------|-------|----------------------|------------|
| <GET>  | <...> | <...> | <...> | <...> | <no / sì: ruolo> |

## 7. Configurazione e segreti
- **Header di sicurezza / CORS / cookie:** <...>
- **Config d'ambiente:** <...>
- **Segreti/credenziali hardcoded:** <identificati, dove; senza esporre i valori>

## 8. Altre informazioni emerse
- <qualsiasi altra info rilevata che non rientra sopra>

## 9. Superfici d'attacco / punti di interesse
- **<Area/funzione>** (`<file>`) — <perché è interessante> — *prima considerazione:* <...>

## 10. Stato del recon
- **Copertura:** <cosa è stato analizzato>
- **Da coprire:** <cosa manca>
- **Domande aperte:** <...>
- **Assunzioni:** <...>
```