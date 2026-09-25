---
name: recon
description: >-
  Fase di ricognizione (recon) su un target web/API in-scope. Naviga e ispeziona l'applicazione in modo ordinato e accurato per comprenderne funzionalità, stack, configurazione ed endpoint partendo SEMPRE dalla cronologia Burp e dalla cartella RESULT.
  Usare quando il tester chiede di fare recon/ricognizione/mappatura di un target, di capire come funziona un'applicazione, o di raccogliere endpoint e superfici d'attacco.
  Produce un file di stato consolidato in RESULT da poter usare nelle altre skill.
---
 
# Skill: recon
 
Ricognizione ordinata e accurata di un target web/API **in-scope**, per comprendere l'applicazione e tutte le sue funzionalità e per raccogliere le informazioni che serviranno alle skill successive. Al termine produce un **file di stato** in `RESULT`.
 
Questa skill segue le regole del `CLAUDE.md` del progetto. In particolare: solo target in-scope, tutte le interazioni verso il target passano dai tool MCP di Burp, e se manca un'informazione **si chiede** invece di assumere.
 
---
 
## 0. Prima di iniziare
 
1. **Verifica scope.** Conferma che il target richiesto sia tra quelli autorizzati nel `CLAUDE.md`. Se non lo è o è ambiguo → fermati e chiedi.
2. **Leggi lo stato esistente.**
   - Controlla la cartella `RESULT`: se esiste già un file di recon per questo target, **riprendi da lì** (aggiorna, non ricominciare da zero).
   - Leggi la cronologia Burp (`get_proxy_http_history` / `get_proxy_http_history_regex`, filtrando per host in-scope) e la cronologia WebSocket se presente. Spesso qui c'è già molto materiale: sfruttalo prima di generare traffico nuovo.
3. **Raccogli il contesto mancante dal tester.** Se per navigare bene servono cose che non sono nella history né nel `CLAUDE.md`, chiedile prima di partire, ad esempio:
   - Credenziali di test e ruoli disponibili (utente, admin, ecc.).
   - Punti d'ingresso noti, aree fuori scope o "da non toccare", host di staging vs prod.
   - Flussi sensibili da evitare (pagamenti reali, invii email/notifiche a utenti veri).
 
---
 
## 1. Ricognizione passiva (dalla history)
 
Prima di generare traffico nuovo, estrai il massimo da ciò che è già in Burp:
 
- **Host e sottodomini** coinvolti, schemi (http/https), porte.
- **Endpoint e metodi** già osservati; raggruppa per area funzionale.
- **Parametri** (query, body, path), header notevoli, cookie e token di sessione.
- **Tecnologie** deducibili da header di risposta (`Server`, `X-Powered-By`, framework), cookie, path, estensioni, commenti o qualsiasi altra cosa che possa essere utile.
- **Formati** delle risposte (HTML, JSON, GraphQL, XML) e pattern delle API.
- **Auth**: come viene gestita la sessione/autenticazione (cookie, Bearer/JWT, header custom), dove avviene il login.

Questa è la prima fase per farsi già un'idea dell'applicazione, della sua superficie e delle sue funzionalità.
 
---
 
## 2. Navigazione attiva e ordinata
 
Se la history non basta a coprire l'applicazione, naviga in modo **sistematico** (ovviamente via MCP di Burp, così ogni richiesta resta tracciata).
Procedi con ordine, non a caso:
 
1. **Punti d'ingresso pubblici**: home, `robots.txt`, `sitemap.xml`, pagine di login/ registrazione, documentazione API se esposta.
2. **Mappatura per aree funzionali**: percorri le sezioni dell'app una per una (es. area pubblica → autenticazione → area utente → area privilegiata), seguendo i link e le funzionalità reali.
3. **Risorse client-side**: individua i file JS/bundle e ispezionali per endpoint, route e parametri nascosti (comprese eventuali sourcemap); segnala chiavi/segreti se emergono.
4. **Superfici API**: per REST ricostruisci le operazioni e le collezioni; per GraphQL verifica l'introspection (se lecita nel contesto) e mappa query/mutation.
5. **Copertura per ruolo**: se hai più utenti/ruoli, ripeti la mappatura per ciascuno, annotando le differenze di accesso (utile poi per test di autorizzazione).

Mantieni traccia di cosa hai già visitato per non duplicare. File di lavoro intermedi (liste grezze, appunti, output di parsing dei JS) vanno nella cartella `files`; il risultato consolidato andrà in `RESULT` (§4). Se durante la navigazione qualcosa non è chiaro o ti manca un accesso, **chiedi**.
 
---
 
## 3. Analisi e considerazioni
 
Consolida quanto raccolto e ragiona sull'applicazione:
 
- A cosa serve, quali sono i flussi principali e le funzionalità chiave.
- Come sembra essere strutturata e configurata l'infrastruttura.
- Dove si concentrano le superfici d'attacco più interessanti (da approfondire con altre
  skill, senza eseguire test attivi qui).
- Cosa resta scoperto o incerto (aree non raggiunte, ruoli non testati, dubbi aperti).

---
 
## 4. Output: file di stato del recon
 
Al termine produci **un unico file accurato** nella cartella `RESULT`, che rappresenta la memoria e lo stato attuale della fase di recon e sarà la base per le skill successive.
 
- **Cartella**: `RESULT`
- **Nome suggerito**: `recon_<host-o-target>.md`
- Se un file di recon per lo stesso target esiste già, **aggiornalo** mantenendo lo storico delle sezioni, invece di crearne uno nuovo scollegato.

Il file è diviso nei capitoli seguenti, **sempre in questo ordine e con questi titoli**:
 
1. **Data ultima modifica** — data (e ora) dell'ultimo aggiornamento del file.
2. **Descrizione applicazione** — cosa fa e a cosa serve, cosa contiene (funzionalità, sezioni, flussi principali) ed eventuali considerazioni utili da tenere presenti.
3. **Autenticazione** — meccanismo di auth se individuabile, credenziali di test usate, ruoli/utenti ottenuti e differenze di accesso; se non ricavabile, riporta comunque una considerazione su come sembra gestita.
4. **Stack tecnologico** — il più completo possibile, dedotto da tutte le risorse utili (header di risposta, cookie, path, bundle JS, messaggi d'errore…). Presentato in una **tabella riassuntiva** con, per ogni voce, *come* è stato dedotto.
5. **Endpoint** — **numero totale** e lista organizzata per area funzionale, in una **tabella chiara** con metodo HTTP, path, scopo, parametri principali e se richiede autenticazione/ruolo.
6. **Altre informazioni emerse** — elenco puntato di qualsiasi altra cosa rilevata durante il recon che non rientri nei capitoli precedenti, così **nessuna informazione va persa** (dettagli di configurazione, comportamenti particolari, file/percorsi notati, commenti nel codice, messaggi, indizi, ecc.).
7. **Superfici d'attacco / punti di interesse** — elenco ragionato delle aree da approfondire con le altre skill, con una **prima considerazione** per ciascuna. In questa fase **non** si eseguono test attivi: qui si annota soltanto e si fa un primo ragionamento in termini di sicurezza e superficie di attacco.
8. **Stato del recon** — copertura raggiunta, cosa è stato visitato, cosa resta da coprire, domande aperte e assunzioni dichiarate.

Mantieni il contenuto pulito, schematizzato, coerente e preciso.
Al termine comunica al tester dove hai salvato il file e una descrizione sul processo e sul suo andamento, se ha avuto problemi o se è stato eseguito correttamente.
 
> **Nota importante, completezza.** Il file deve essere il **più completo possibile**, con tutte le informazioni raccolte: l'obiettivo è non perdere nulla di ciò che è emerso durante il recon. Può quindi essere **lungo quanto serve, senza limiti di lunghezza**. In caso di dubbio, includi l'informazione piuttosto che ometterla.
 
### Template della struttura (da seguire identico a ogni output)
 
```markdown
# Recon — <target>
 
## 1. Data ultima modifica
<AAAA-MM-GG HH:MM>
 
## 2. Descrizione applicazione
- **Obiettivo:** <...>
- **Sezioni e funzionalità:**
  - <sezione/funzionalità> — <descrizione>
- **Considerazioni:** <...>
 
## 3. Autenticazione
- **Meccanismo:** <cookie / JWT / header custom / non determinato>
- **Flusso di login:** <...>
- **Credenziali usate:** <utente/ruolo → dove/come>
- **Ruoli individuati:** <...>
- **Considerazioni:** <...>
 
## 4. Stack tecnologico
| Componente | Tecnologia e Versione | Come è stato dedotto |
|------------|---------------------|----------------------|
| <es. Web server> | <...> | <es. header Server> |
| <es. Framework>  | <...> | <...> |
| <es. Frontend>   | <...> | <...> |
 
## 5. Endpoint
**Numero totale:** <N>
 
### <Area funzionale>
| Metodo | Path | Scopo | Parametri principali | Auth/Ruolo |
|--------|------|-------|----------------------|------------|
| <GET>  | <...> | <...> | <...> | <no / sì: ruolo> |
 
## 6. Altre informazioni emerse
- <qualsiasi altra info rilevata che non rientra sopra>
 
## 7. Superfici d'attacco / punti di interesse
- **<Area/endpoint>** — <perché è interessante> — *prima considerazione:* <...>
 
## 8. Stato del recon
- **Copertura:** <cosa è stato coperto>
- **Da coprire:** <cosa manca>
- **Domande aperte:** <...>
- **Assunzioni:** <...>
```
 