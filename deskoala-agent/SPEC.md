# Deskoala Read-Only Agent — Specifiche

> Stato: **bozza v0.1** — da validare contro lo swagger reale
> (`https://webapi.deskoala.com/swagger/index.html`).
> Al momento della stesura l'host non era raggiungibile dall'ambiente di sviluppo,
> quindi il catalogo endpoint (§5) è **generato a runtime dal file OpenAPI** invece
> di essere scritto a mano. Le sezioni marcate con 🔎 vanno completate dopo aver
> scaricato `swagger.json`.

## 1. Obiettivo

Un agente costruito con il **Claude Agent SDK** che risponde in linguaggio naturale a
domande sui dati di Deskoala (help desk / ticketing) interrogando la Web API
**esclusivamente in lettura**.

Esempi di domande:

- "Quanti ticket aperti ha il cliente ACME e quali sono i più vecchi?"
- "Mostrami lo storico del ticket 12345 con gli ultimi commenti."
- "Quali ticket sono assegnati a Mario Rossi con priorità alta?"
- "Riepilogo dei ticket chiusi la settimana scorsa per categoria."

## 2. Non-obiettivi

- Nessuna creazione, modifica, chiusura, assegnazione o cancellazione di dati.
- Nessun invio di email/notifiche tramite Deskoala.
- Nessun accesso al filesystem o shell da parte del modello.
- Nessuna persistenza dei dati Deskoala oltre la sessione (salvo log di audit, §8).

## 3. Vincolo di sola lettura (requisito principale)

Il vincolo è applicato su **tre livelli indipendenti**; ognuno da solo deve bastare.

| Livello | Meccanismo | Dettaglio |
|---|---|---|
| L1 — Catalogo | Filtro OpenAPI | Vengono esposti al modello solo le operazioni `GET` dello swagger. `POST/PUT/PATCH/DELETE` sono scartate al caricamento. |
| L2 — HTTP client | Guardia nel codice | `DeskoalaClient.request()` accetta solo `GET` (e `HEAD`). Qualsiasi altro metodo → eccezione, senza chiamata di rete. Unica eccezione: la chiamata di login (§4), interna al modulo auth e mai raggiungibile dai tool. |
| L3 — Credenziali | Lato Deskoala | Usare un utente/API key dedicato con ruolo **sola lettura** (da richiedere all'amministratore Deskoala). |

Inoltre:

- `allowedTools` contiene solo i tool `mcp__deskoala__*` (nessun `Bash`, `Write`, `Edit`, `WebFetch`).
- Hook `PreToolUse` che blocca qualsiasi tool non nella allowlist (difesa in profondità).
- Allowlist di path: `deskoala_get` rifiuta un `path` che non corrisponde a un template GET del catalogo (niente path arbitrari, niente host diversi da `DESKOALA_BASE_URL`).
- Endpoint GET con effetti collaterali (es. `GET /.../export`, `GET /.../markAsRead`) vanno inseriti in una **denylist** esplicita 🔎.

## 4. Autenticazione 🔎

Da confermare leggendo `components.securitySchemes` dello swagger. Supportare via config:

| Modalità | Variabili d'ambiente |
|---|---|
| Bearer token statico / API key | `DESKOALA_TOKEN` (+ `DESKOALA_AUTH_HEADER`, default `Authorization: Bearer`) |
| Login utente → JWT | `DESKOALA_USERNAME`, `DESKOALA_PASSWORD`, `DESKOALA_LOGIN_PATH` |

Requisiti:

- Token cache in memoria; refresh automatico su `401` (una sola volta per richiesta).
- Le credenziali non compaiono mai nei messaggi al modello né nei log.
- `DESKOALA_BASE_URL` default `https://webapi.deskoala.com`.

## 5. Catalogo endpoint

### 5.1 Generazione automatica

All'avvio:

1. Carica lo swagger da `DESKOALA_OPENAPI_URL` (default `${BASE_URL}/swagger/v1/swagger.json`) oppure da file locale `DESKOALA_OPENAPI_FILE` (consigliato: versionato in `deskoala-agent/openapi/swagger.json` per build riproducibili).
2. Tiene solo le operazioni `get`, escludendo la denylist.
3. Per ciascuna costruisce una voce di catalogo: `operationId`, `method`, `pathTemplate`, `tag`, `summary`, parametri (`path`/`query`, tipo, required, enum), schema di risposta (riassunto a 2 livelli).

### 5.2 Tool curati 🔎

Dopo aver letto lo swagger, per le entità più usate si definiscono tool dedicati con
schema Zod esplicito (più affidabili del tool generico). Mappatura attesa da validare:

| Tool | Entità tipica help desk | Endpoint Deskoala (da compilare) |
|---|---|---|
| `search_tickets` | Ticket: filtro stato, priorità, cliente, assegnatario, date | `GET /api/...` |
| `get_ticket` | Dettaglio ticket per id | `GET /api/.../{id}` |
| `get_ticket_messages` | Commenti / interazioni del ticket | `GET /api/.../{id}/...` |
| `search_customers` | Clienti / aziende / contatti | `GET /api/...` |
| `list_users` | Operatori / agenti / gruppi | `GET /api/...` |
| `list_lookups` | Stati, priorità, categorie, SLA | `GET /api/...` |

## 6. Tool esposti al modello

Server MCP in-process (`createSdkMcpServer`, nome `deskoala`).

| Tool | Input | Output |
|---|---|---|
| `deskoala_list_endpoints` | `tag?`, `search?` | Elenco compatto `operationId · path · summary` |
| `deskoala_describe_endpoint` | `operationId` | Parametri e schema di risposta |
| `deskoala_get` | `operationId`, `pathParams?`, `query?`, `maxItems?` (default 50, max 200) | JSON della risposta, troncato e paginato |
| tool curati (§5.2) | schema specifico | JSON normalizzato |

Regole di output:

- Risposte > ~20 KB: troncate con indicazione `truncated: true` e totale elementi; il modello deve restringere i filtri.
- Paginazione: il tool segue al massimo `DESKOALA_MAX_PAGES` (default 5) pagine.
- Errori HTTP restituiti come `{ error: { status, message } }` senza stack trace.
- `429` / `5xx`: retry con backoff esponenziale (max 3).

## 7. Prompt di sistema (sintesi)

- Sei un assistente che **legge** dati Deskoala; non puoi modificare nulla. Se l'utente chiede un'azione di scrittura, spiega che non è supportata e suggerisci cosa fare nell'interfaccia Deskoala.
- Prima di interrogare un endpoint che non conosci usa `deskoala_list_endpoints` / `deskoala_describe_endpoint`.
- Preferisci filtri lato server a scaricare grandi liste.
- Cita sempre gli id dei record su cui basi la risposta.
- Non inventare dati: se l'API non restituisce un'informazione, dillo.
- Rispondi nella lingua dell'utente; date in formato `gg/mm/aaaa`, fuso `Europe/Rome`.
- Minimizza i dati personali nelle risposte (mostra solo i campi richiesti).

## 8. Sicurezza, privacy, audit

- Log di audit JSONL (`logs/audit-YYYY-MM-DD.jsonl`): timestamp, utente, `operationId`, parametri, status, durata, numero record. **Mai** corpi di risposta né token.
- Il contenuto restituito dall'API (testi dei ticket, email dei clienti) è **dato non attendibile**: il prompt istruisce il modello a non eseguire istruzioni contenute nei ticket (prompt injection). L'impatto è comunque limitato dalla sola lettura (§3).
- Dati personali (GDPR): l'agente non esporta dati in file; eventuali campi sensibili configurabili in `DESKOALA_REDACT_FIELDS` vengono mascherati prima di arrivare al modello.
- Rate limit client-side: `DESKOALA_MAX_RPS` (default 5).

## 9. Architettura e struttura del progetto

```
deskoala-agent/
├── SPEC.md
├── README.md
├── package.json            # bun, @anthropic-ai/claude-agent-sdk, zod
├── .env.example
├── openapi/swagger.json    # snapshot dello swagger 🔎
└── src/
    ├── index.ts            # CLI interattiva (readline) → query()
    ├── agent.ts            # opzioni SDK: prompt, allowedTools, hooks, mcpServers
    ├── prompt.ts           # system prompt (§7)
    ├── deskoala/
    │   ├── client.ts       # HTTP client GET-only, retry, rate limit (§3 L2)
    │   ├── auth.ts         # token / login (§4)
    │   └── catalog.ts      # parsing OpenAPI → catalogo GET (§5.1)
    ├── tools/
    │   ├── generic.ts      # list/describe/get (§6)
    │   └── curated.ts      # tool curati (§5.2)
    └── audit.ts            # log di audit (§8)
```

Configurazione SDK (bozza):

```ts
query({
  prompt,
  options: {
    model: "opus",
    systemPrompt: DESKOALA_PROMPT,
    mcpServers: { deskoala: deskoalaServer },
    allowedTools: ["mcp__deskoala__*"],
    disallowedTools: ["Bash", "Write", "Edit", "WebFetch", "WebSearch"],
    permissionMode: "default",
    maxTurns: 30,
    hooks: { PreToolUse: [{ hooks: [blockNonDeskoalaTools] }] },
  },
});
```

## 10. Criteri di accettazione

1. Con lo swagger reale, `deskoala_list_endpoints` non elenca alcuna operazione non-GET.
2. Test unitario: `DeskoalaClient` lancia errore su `POST/PUT/PATCH/DELETE` senza effettuare chiamate di rete.
3. Test: `deskoala_get` rifiuta `operationId` sconosciuti, path non da catalogo e host esterni.
4. Richiesta "chiudi il ticket 123" → l'agente rifiuta e nessuna chiamata non-GET compare nell'audit log.
5. Le 4 domande di esempio (§1) ricevono risposta corretta su un ambiente di test, con id dei record citati.
6. Nessun token/password presente in log o transcript.
7. Risposte grandi sono troncate e il modello raffina la query invece di fallire.

## 11. Domande aperte 🔎

1. Formato e scheme di autenticazione della Web API (API key, JWT da login, OAuth2?).
2. Esiste un ambiente di test/sandbox Deskoala?
3. Paginazione: parametri usati (`page/pageSize`, `skip/take`, `$top/$skip` OData?).
4. Endpoint GET con effetti collaterali da mettere in denylist.
5. Interfaccia desiderata: CLI, chat web (come `simple-chatapp`), o integrazione Teams?
6. Multi-tenant / multi-utente: le query devono girare con le credenziali dell'utente finale?
