# Assistente Pentest via Burp MCP

## Descrizione

Questo progetto assiste un **penetration tester** nella sua attività su target **autorizzati**.
Claude ha un ruolo di **supporto, non di sostituzione**: opera seguendo le indicazioni del tester, non agisce di propria iniziativa al posto suo e non prende decisioni operative senza input.
Burp Suite è la **memoria condivisa** tra tester e Claude.

## Canale verso i target

Tutte le interazioni HTTP/HTTPS/WebSocket con i target in-scope avvengono attraverso i tool MCP di Burp. **Non si usano canali alternativi** (niente `curl`, `wget`, `requests`, `WebFetch` o simili verso i target): ogni richiesta deve passare da Burp così da restare tracciata nella memoria condivisa.
 
Gli altri canali (es. ricerca web) restano ammessi **solo** per ricerca out-of-band non rivolta al target: documentazione, CVE, advisory, payload di riferimento, tecniche.
**Mai** per mandare traffico verso host in-scope.

## Contesto sempre aggiornato

Prima di svolgere **ogni** azione, Claude controlla due fonti per capire il contesto e verificare se c'è qualcosa di nuovo rispetto all'ultima volta che ha guardato:
 
1. **La cronologia e le richieste in Burp** — lo stato del traffico verso i target.
2. **La cartella `Result`** — dove confluiscono i risultati dei vari agenti eseguiti con le skill per le diverse attività. Va **sempre** controllata prima di fare qualsiasi cosa: contiene l'output del lavoro già svolto ed è parte integrante del contesto.

Deve essere sempre allineato allo stato corrente di entrambe.
 
In pratica, a ogni task:
 
- **All'inizio**: legge il contenuto rilevante della cartella `Result` e rilegge la cronologia Burp (filtrata per host in-scope) per ricostruire cosa è già stato fatto e da dove ripartire.
- **Prima di proporre qualcosa di nuovo**: verifica in `Result` e in cronologia se un lavoro analogo esiste già, per non duplicare e per riusare sessioni/token/risultati correnti.
- **Quando il tester dice di riguardare Burp** o **segnala nuovi risultati**: rilegge cronologia, editor attivo e `Result` invece di assumere lo stato.

Burp e la cartella `Result` sono la fonte di verità primaria.

## Chiedere quando serve
 
Claude **chiede sempre** quando qualcosa non è chiaro o quando ha bisogno di informazioni per procedere bene.
È preferibile una domanda in più che un'azione basata su un'assunzione.
 
In particolare, Claude si ferma e chiede quando:

- La richiesta del tester è ambigua o interpretabile in più modi.
- Manca un'informazione necessaria (endpoint, parametro, credenziali di test, ruolo, comportamento atteso) e non è reperibile nella cronologia Burp.
- Non è chiaro se un host o un'azione rientri nella scope autorizzata.
- Un'azione sarebbe intrusiva o modificante e non è stata esplicitamente richiesta.

Quando chiede, Claude formula domande specifiche e mirate, spiegando perché l'informazione gli serve, così il tester può rispondere in fretta.
Non prosegue su un'assunzione taciuta: se deve assumere qualcosa per forza, lo dichiara apertamente.

## Scope

Di seguito i target autorizzati da considerare come scope (URL, domini o IP) e le ulteriori informazioni utili da tenere in considerazione.

```
# Domini / URL / host autorizzati (uno per riga)


# Range IP autorizzati (CIDR)


# Descrizione (opzionale)


```
