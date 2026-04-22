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
- Tab admin: dashboard protetta da password (`916sps`) che legge da Supabase e permette aggiornamento stato ordini, export CSV/PDF, download immagini
- Tutte le chiamate Supabase sono `fetch()` REST dirette dal browser usando la chiave anon
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
- Tabella: `ordini` (colonne: `numero`, `data`, `nome`, `email`, `telefono`, `canale`, `ditta`, `attne`, `via`, `cap`, `citta`, `provincia`, `articoli` (stringa JSON), `note`, `fattura` (stringa JSON), `stato`, `created_at`)
- Bucket storage: `immagini-ordini` — file salvati al percorso `{orderId}/prod{index}_{fileIndex}_{timestamp}.{ext}`
- Le configurazioni 3D dei prodotti sono serializzate dentro il campo JSON `articoli`, non in una colonna separata

### Modelli 3D

Tutti i file `.draco.glb` nella root sono i modelli di abbigliamento. Corrispondono 1:1 alle opzioni `<select>` in `configuratore.html`. Per aggiungere un nuovo modello: copiare il file `.glb` nella root, aggiungere un `<option>` a `#sel-model` in `configuratore.html`, e aggiungere il nome prodotto all'array `CATALOGUE` in `index.html`.

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
