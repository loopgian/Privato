# MCP Odoo

Questo progetto include la configurazione del server MCP
[`mcp-server-odoo`](https://github.com/ivnvxd/mcp-server-odoo) (v0.7.1), che
permette a Claude Code di leggere e scrivere sui record di un'istanza Odoo.

La configurazione sta in `.mcp.json` (scope "project"), quindi vale sia per
Claude Code da terminale sia per **Claude Code sul web** (claude.ai/code): le
sessioni cloud clonano la repo e caricano il `.mcp.json` che trovano dentro.

---

## A) Uso nella versione web (claude.ai/code)

Le sessioni cloud caricano i server MCP di progetto **senza chiedere
approvazione** (il prompt interattivo non esiste in quel contesto). Servono
solo due cose configurate lato ambiente cloud.

### 1. Variabili d'ambiente

Su [claude.ai/code](https://claude.ai/code) apri le impostazioni del **cloud
environment** e nel campo delle variabili d'ambiente (formato `.env`, una
coppia `KEY=value` per riga) aggiungi **solo la API key**:

```
ODOO_API_KEY=la-tua-api-key
```

URL, database, utente e modalita' YOLO hanno gia' il default corretto dentro
`.mcp.json`, quindi non serve ripeterli. Impostali solo per puntare a
un'istanza diversa. Attenzione in particolare a `ODOO_YOLO`, che per default
vale `true`: la scrittura e' abilitata (vedi [Modalita' YOLO](#modalita-yolo)).

Le variabili vengono copiate nella sessione **all'avvio**: se le modifichi,
le sessioni gia' in corso mantengono i vecchi valori: va aperta una sessione
nuova.

> ⚠️ **Attenzione alle credenziali.** I cloud environment **non hanno un
> secrets store**: chiunque usi quell'ambiente puo' leggerne le variabili, e
> la documentazione Anthropic sconsiglia esplicitamente di metterci API key.
> Se procedi comunque, fallo con questa consapevolezza e limita il danno
> possibile: crea un **utente Odoo dedicato** con permessi minimi (solo i
> modelli che ti servono), usa la sua API key e non quella di un
> amministratore, e revocala quando non serve piu'.

### 2. Accesso di rete

Il livello di rete di default e' **Trusted**, che consente solo i domini in
allowlist (registri di pacchetti, GitHub): l'host Odoo **non** e' raggiungibile.
Verificato: da una sessione cloud con impostazioni di default, una richiesta a
`loopgroup.odoo.com` viene rifiutata dal proxy con `CONNECT tunnel failed,
response 403`.

Nelle impostazioni dell'ambiente scegli quindi **Custom** e:

- aggiungi il dominio Odoo: `loopgroup.odoo.com`;
- lascia spuntato **"Also include default list of common package managers"**,
  altrimenti `uvx` non riesce a scaricare il pacchetto da PyPI.

In alternativa si puo' usare il livello **Full**, che consente qualsiasi
dominio.

> L'istanza Odoo deve essere **raggiungibile da internet**. Un Odoo on-premise
> senza esposizione pubblica non e' contattabile dalla VM cloud.

### 3. Verifica

`uv`/`uvx` sono gia' preinstallati nelle VM cloud, non serve nessun setup
script. In una sessione nuova, controlla con:

```bash
claude mcp list
```

Il server deve comparire come `✓ Connected`. Se resta pendente o assente, la
causa quasi certa e' la `ODOO_API_KEY` mancante: le variabili vengono lette
solo all'avvio, quindi dopo averla aggiunta serve una sessione **nuova**.

#### Stato verificato

Configurazione collaudata end-to-end il 2026-08-24 da una sessione cloud:

| Controllo | Esito |
|---|---|
| Rete verso `loopgroup.odoo.com` | HTTP 200 — allowlist dell'ambiente gia' corretta |
| Endpoint XML-RPC | `server_version` `19.0+e` |
| `common.authenticate()` con la API key | `uid = 2` (`info@loop-group.it`) |
| Lettura reale (`res.partner` / `search_count`) | 119 record |
| Avvio di `uvx mcp-server-odoo@0.7.1` | `odoo-mcp-server` 1.29.0 |
| Tool esposti | 9 |

Con `ODOO_YOLO=true` i tool disponibili sono `search_records`, `get_record`,
`list_models`, `list_resource_templates` e `aggregate_records` in lettura, piu'
`create_record`, `update_record`, `delete_record` e `post_message` in scrittura.

---

## B) Uso da terminale (locale)

1. Installa [`uv`](https://docs.astral.sh/uv/) (fornisce `uvx`):

   ```bash
   curl -LsSf https://astral.sh/uv/install.sh | sh
   ```

2. Copia il file di esempio e inserisci la API key (il resto ha gia' il
   default corretto):

   ```bash
   cp .env.example .env
   ```

3. Esporta le variabili nella shell da cui lanci Claude Code (il server MCP
   legge le variabili d'ambiente del processo che lo avvia):

   ```bash
   set -a && source .env && set +a
   claude
   ```

Il file `.claude/settings.json` contiene `enabledMcpjsonServers: ["odoo"]`, che
pre-approva il server ed evita il prompt di conferma. Nota: questa
pre-approvazione committata viene applicata solo **dopo** che hai accettato il
dialog di fiducia della cartella (workspace trust) al primo avvio di `claude`
nella repo; finche' la cartella non e' fidata il server resta in
`⏸ Pending approval`.

---

## Odoo: cosa serve lato server

- Un'istanza Odoo con XML-RPC abilitato.
- **API key**: Odoo > Impostazioni > Il mio profilo > Sicurezza account >
  Chiavi API. E' l'unico segreto della configurazione.

### Modalita' YOLO

Il server MCP ha due modi di operare:

- **Modalita' standard** (`ODOO_YOLO=off`): richiede il modulo
  [`mcp_server`](https://apps.odoo.com/apps/modules/19.0/mcp_server)
  installato su Odoo, che aggiunge il controllo dei permessi **per modello**
  (decidi tu quali modelli sono leggibili e scrivibili via MCP).
- **Modalita' YOLO**: parla direttamente con XML-RPC, senza il modulo.
  - `read` — sola lettura;
  - `true` — lettura **e scrittura**, senza nessun controllo per modello:
    Claude puo' creare, modificare ed eliminare record ovunque i permessi
    dell'utente Odoo lo consentano.

Su **Odoo Online** (`*.odoo.com`, il nostro caso) non e' possibile installare
moduli di terze parti, quindi la modalita' standard non e' disponibile e YOLO
e' l'unica strada.

**Modalita' scelta per questo progetto: `true`**, cioe' lettura e scrittura.
Coincide con il default di `.mcp.json`, quindi non serve impostare `ODOO_YOLO`
nell'ambiente. Se in futuro servisse il solo accesso in consultazione,
`ODOO_YOLO=read` limita il server ai cinque tool di lettura.

Cosa comporta questa scelta, detto chiaramente: la API key in uso e' quella di
`info@loop-group.it`, l'utente amministratore (`uid 2`), quindi Claude puo'
creare, modificare ed eliminare record su **tutto** il database, senza alcun
controllo per modello. La protezione vera resta lato Odoo: per restringere il
raggio d'azione, crea un utente dedicato con i soli permessi necessari e usa
la sua API key al posto di questa.

## Configurazione

In `.mcp.json` stanno solo valori non sensibili (URL, database, utente), come
default sovrascrivibili. La **API key non e' nel file**: viene letta
dall'ambiente tramite la sostituzione `${VAR}` di Claude Code. `.env` e' in
`.gitignore`.

| Variabile | Obbligatoria | Default in `.mcp.json` |
|---|---|---|
| `ODOO_API_KEY` | **si** | nessuno — va fornita dall'ambiente |
| `ODOO_URL` | no | `https://loopgroup.odoo.com` |
| `ODOO_DB` | no | `loopgroup` |
| `ODOO_USER` | no | `info@loop-group.it` |
| `ODOO_YOLO` | no | `true` (vedi [Modalita' YOLO](#modalita-yolo)) |

In alternativa alla API key si puo' usare `ODOO_USER` + `ODOO_PASSWORD`, ma
la API key e' preferibile: e' revocabile singolarmente.

La lista completa delle opzioni (limiti di paginazione, log, locale, transport
HTTP) e' disponibile con:

```bash
uvx mcp-server-odoo@0.7.1 --help
```

## Aggiornare la versione

La versione e' pinnata in `.mcp.json` (`mcp-server-odoo@0.7.1`) per avere build
riproducibili. Per aggiornare, modifica il numero di versione in quel file.
