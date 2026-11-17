# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Panoramica del progetto

App web statica per **B1PIU'8 di Biffi Davide** — brand italiano di abbigliamento sportivo personalizzato. Nessun build system, nessun package manager, nessun framework. Tutto il codice è HTML/CSS/JavaScript puro, servito direttamente (ospitato su GitHub Pages: `https://b1piu8.github.io/b1piu8-modelli/`).

## Sviluppo

Aprire i file direttamente nel browser oppure avviare un server statico locale:

```bash
# Con Python
python3 -m http.server 8080

# Con Node
npx serve .
```

Non esistono comandi di build, lint o test.

## Architettura

### Due file HTML principali

**`index.html`** — Form ordini + Dashboard Admin (SPA con due tab):
- Tab clienti: form multi-prodotto con caricamento immagini e integrazione configuratore 3D
- Tab admin: dashboard protetta da password. La password NON è nel codice: è salvata cifrata nel database (`privato.impostazioni`) e verificata dalle funzioni `admin_*`. Si cambia dal SQL Editor di Supabase con `select privato.imposta_password('nuova-password');`
- Il form clienti usa la chiave anon solo per: inserire l'ordine (`POST /rest/v1/ordini`), caricare immagini nello storage, chiedere il numero con `rpc/prossimo_numero_ordine`
- L'admin usa solo funzioni RPC protette da password: `admin_login`, `admin_ordini`, `admin_stato`, `admin_elimina`, `admin_archivia_inizia` / `admin_archivia_fine` / `admin_archivia_annulla`
- Pulsante "Archivia immagini dei completati": scarica in .zip le immagini degli ordini completati, chiede conferma, poi le cancella dallo storage (policy `archiviazione_ordini`: permette di cancellare solo i file degli ordini con `in_archiviazione = true`) e segna `immagini_archiviate_il`
- `localStorage` è usato come cache secondaria per lo stato degli ordini (`b1p8_orders`, `b1p8_counter`)

**`configuratore.html`** — Configuratore 3D abbigliamento:
- Usa Three.js (caricato via CDN) per renderizzare modelli `.glb` compressi con Draco
- Sidebar sinistra: selettore modello + color picker per zona con palette colori
- Sidebar destra: editor layer testo e uploader logo/texture (due modalità: decal overlay o texture UV)
- Funziona sia standalone che come iframe incorporato in `index.html`

### Comunicazione iframe / postMessage

`index.html` incorpora `configuratore.html` in un modal iframe. Comunicano via `window.postMessage`:

| Tipo messaggio | Direzione | Scopo |
|---|---|---|
| `CONFIGURATORE_PRONTO` | configuratore → index | Il configuratore è pronto a ricevere dati |
| `CARICA_CONFIGURAZIONE` | index → configuratore | Ripristina una configurazione salvata in precedenza |
| `CONFIGURAZIONE_SALVATA` | configuratore → index | L'utente ha cliccato Salva; porta l'oggetto config completo |

In `index.html` esistono due handler `addEventListener('message', ...)` separati — uno per il configuratore del form ordini, uno per il viewer 3D admin — non vanno uniti.

### Backend: Supabase

- URL del progetto e chiave anon sono hardcoded in `index.html` (righe ~584–585)
- Tabella: `ordini` (colonne: `numero`, `data`, `nome`, `email`, `telefono`, `canale`, `ditta`, `attne`, `via`, `cap`, `citta`, `provincia`, `articoli` (stringa JSON), `note`, `fattura` (stringa JSON), `stato`, `created_at`, `configurazione`, `immagini_archiviate_il`, `in_archiviazione`). RLS attivo: la chiave anon può solo inserire.
- Bucket storage: `immagini-ordini` — file salvati al percorso `{orderId}/prod{index}_{fileIndex}_{timestamp}.{ext}`
- Le configurazioni 3D dei prodotti sono serializzate dentro il campo JSON `articoli`, non in una colonna separata

### Modelli 3D

Tutti i file `.draco.glb` nella root sono i modelli di abbigliamento. Corrispondono 1:1 alle opzioni `<select>` in `configuratore.html`. Il configuratore 3D non è più nel modulo clienti: resta solo nell'admin per gli ordini vecchi.

### Catalogo prodotti

I prodotti del modulo sono nell'array `CATALOGO` in `index.html`, diviso per sport: ogni voce è `[nome completo, nome file]`. Le immagini sono in `prodotti/<nome file>.jpg` (850×1075, JPG RGB). Il cliente li sceglie dalla finestra "Seleziona articolo" (anche più insieme, pulsante CARICA). Per aggiungere un prodotto: mettere l'immagine in `prodotti/` (convertita da CMYK a RGB) e aggiungere la voce nella sezione giusta di `CATALOGO`.

### Struttura dati di una configurazione salvata

Quando l'utente salva dal configuratore, il payload del messaggio `CONFIGURAZIONE_SALVATA` contiene:
```js
{
  type: 'CONFIGURAZIONE_SALVATA',
  configId, configUrl, model, modelLabel,
  screenshot,      // JPEG base64 della vista 3D
  zones: [{ name, color }],
  texts: [{ text, font, weight, color, strokeColor, strokeWidth, ... }],
  logos: [{ name, imgSrc, ... }],
  textures: [{ zoneId, imgSrc, ... }]
}
```
`window._cfgRegistry` in `index.html` è una cache runtime degli oggetti config, indicizzata per `"{orderId}_{prodIndex}"`, usata per evitare problemi di escape HTML quando le config vengono passate come argomenti inline negli onclick.
