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
coppia `KEY=value` per riga) aggiungi:

```
ODOO_URL=https://tuazienda.odoo.com
ODOO_DB=tuazienda
ODOO_API_KEY=la-tua-api-key
```

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
Nelle impostazioni dell'ambiente scegli **Custom** e:

- aggiungi il dominio Odoo, es. `tuazienda.odoo.com` (oppure
  `*.tuazienda.com` per tutti i sottodomini);
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

---

## B) Uso da terminale (locale)

1. Installa [`uv`](https://docs.astral.sh/uv/) (fornisce `uvx`):

   ```bash
   curl -LsSf https://astral.sh/uv/install.sh | sh
   ```

2. Copia il file di esempio e compila i valori:

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
  Chiavi API.
- Consigliato in produzione: il modulo
  [`mcp_server`](https://apps.odoo.com/apps/modules/19.0/mcp_server) installato
  su Odoo (Odoo 16+), che aggiunge il controllo dei permessi per modello.
  Senza il modulo si puo' usare `ODOO_YOLO=read` (sola lettura, solo per test).

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
