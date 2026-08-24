# MCP Odoo

Questo progetto include la configurazione del server MCP
[`mcp-server-odoo`](https://github.com/ivnvxd/mcp-server-odoo) (v0.7.1), che
permette a Claude Code di leggere e scrivere sui record di un'istanza Odoo.

## Requisiti

- [`uv`](https://docs.astral.sh/uv/) installato in locale (fornisce `uvx`):
  `curl -LsSf https://astral.sh/uv/install.sh | sh`
- Un'istanza Odoo raggiungibile con XML-RPC abilitato.
- Consigliato in produzione: il modulo
  [`mcp_server`](https://apps.odoo.com/apps/modules/19.0/mcp_server) installato
  su Odoo (Odoo 16+), che aggiunge il controllo dei permessi per modello.
  Senza il modulo si puo' usare `ODOO_YOLO=read` (sola lettura, solo per test).

## Setup

1. Copia il file di esempio e compila i valori:

   ```bash
   cp .env.example .env
   ```

2. Genera una API key su Odoo: **Impostazioni > Il mio profilo > Sicurezza
   account > Chiavi API**, e incollala in `ODOO_API_KEY`.

3. Esporta le variabili nella shell da cui lanci Claude Code (il server MCP
   legge le variabili d'ambiente del processo che lo avvia):

   ```bash
   set -a && source .env && set +a
   claude
   ```

4. Alla prima apertura del progetto, Claude Code chiede di approvare il server
   MCP definito in `.mcp.json`. Verifica con:

   ```bash
   claude mcp list
   ```

## Configurazione

Le credenziali **non** stanno in `.mcp.json`: il file usa la sostituzione
`${VAR}` di Claude Code e legge i valori dall'ambiente. `.env` e' in
`.gitignore`.

| Variabile | Obbligatoria | Descrizione |
|---|---|---|
| `ODOO_URL` | si | URL dell'istanza Odoo |
| `ODOO_DB` | quasi sempre | Nome del database (obbligatorio se la lista database e' disabilitata, come su Odoo Online) |
| `ODOO_API_KEY` | si* | API key Odoo |
| `ODOO_USER` / `ODOO_PASSWORD` | si* | Alternativa alla API key |

*Serve `ODOO_API_KEY` **oppure** la coppia `ODOO_USER` + `ODOO_PASSWORD`.

La lista completa delle opzioni (limiti di paginazione, log, locale, transport
HTTP) e' disponibile con:

```bash
uvx mcp-server-odoo@0.7.1 --help
```

## Aggiornare la versione

La versione e' pinnata in `.mcp.json` (`mcp-server-odoo@0.7.1`) per avere build
riproducibili. Per aggiornare, modifica il numero di versione in quel file.
