Demande la permission avant de modifier des fichiers, de préférence ne supprime rien, ne modifie rien. Juste ajoute éventuellement

Tu dois me challenger pour que mon document de spec soit le plus précis possible, parfois insister même quand je te contredis. 

Pour l'instant le dossier de spec ignore les aspects monétaires. 

Fait bien attention au fait que je suis revenu sur certaines features, je l'écrit en italique en commentaire, je ne veux pas les mentionner dans les specs. 

A chaque réponse, prends bien le temps de te remémorer absolument toutes les features que je souhaite ajouter dans le projet (dossier Features) pour me faire les meilleurs suggestions d'ajouts / d'amélioration. 

Si quelques chose de techniquement similaire (pas forcément sur la thématique éducation) suggère le moi. 

Pour lire les fichiers excalidraw, je veux que tu le fasse visuellement. tu as accès à un tool : 

## excalidraw-to-image` — Read wireframes

Converts an Excalidraw file to PNG.

```bash
excalidraw-to-image <file.excalidraw.md> [-o output.png]
```

| Flag | Description | Default |
|------|-------------|---------|
| `-o <path>` | Output PNG path | Same name as input with `.png` |
| `-h, --help` | Show help | |

**Supported input formats:**

| Format | Description |
|--------|-------------|
| `.excalidraw.md` | Obsidian Excalidraw plugin (compressed JSON in Markdown) |
| `.excalidraw` | Plain Excalidraw JSON |

**Examples:**

```bash
excalidraw-to-image "Login Flow.excalidraw.md"
excalidraw-to-image "Login Flow.excalidraw.md" -o "/tmp/preview.png"
```

---

## `excalidraw-draw` — Create wireframes

Generates a `.excalidraw` file from a JSON spec. The output can be opened in Obsidian/Excalidraw or converted to PNG with `excalidraw-to-image`.

```bash
excalidraw-draw <spec.json> [-o output.excalidraw]
excalidraw-draw --inline '<json>' [-o output.excalidraw]
```

| Flag | Description | Default |
|------|-------------|---------|
| `-o <path>` | Output `.excalidraw` path | Same name as input with `.excalidraw` |
| `--inline <json>` | Pass spec as a JSON string instead of a file | |
| `-h, --help` | Show help | |

### Spec format

```json
{
  "background": "#ffffff",
  "elements": [
    { "type": "rectangle", "x": 0,   "y": 0,   "width": 300, "height": 60,  "label": "Header" },
    { "type": "ellipse",   "x": 0,   "y": 80,  "width": 60,  "height": 60  },
    { "type": "diamond",   "x": 100, "y": 80,  "width": 100, "height": 60,  "label": "Decision?" },
    { "type": "text",      "x": 0,   "y": 160, "text": "A note", "fontSize": 14 },
    { "type": "arrow",     "x": 50,  "y": 60,  "points": [[0,0],[0,80]] },
    { "type": "line",      "x": 0,   "y": 200, "points": [[0,0],[300,0]] },
    { "type": "frame",     "x": 0,   "y": 0,   "width": 400, "height": 300, "label": "Screen A" }
  ]
}
```

### Supported element types

| Type | Required fields | Notes |
|------|----------------|-------|
| `rectangle` | `x`, `y`, `width`, `height` | Add `label` for text inside |
| `ellipse` | `x`, `y`, `width`, `height` | Add `label` for text inside |
| `diamond` | `x`, `y`, `width`, `height` | Add `label` for text inside |
| `text` | `x`, `y`, `text` | Standalone text, use `fontSize` to size |
| `arrow` | `x`, `y`, `points` | `points` are relative to `x,y` — e.g. `[[0,0],[100,50]]` |
| `line` | `x`, `y`, `points` | Same as arrow but no arrowhead |
| `frame` | `x`, `y`, `width`, `height` | Named container, use `label` for the frame title |

### Style properties (all optional)

| Property | Values | Default |
|----------|--------|---------|
| `strokeColor` | Any hex color | `#1e1e1e` |
| `backgroundColor` | Any hex color or `transparent` | `transparent` |
| `fillStyle` | `solid`, `hachure`, `cross-hatch`, `dots` | `solid` |
| `strokeWidth` | `1`, `2`, `4` | `2` |
| `strokeStyle` | `solid`, `dashed`, `dotted` | `solid` |
| `roughness` | `0` (clean), `1` (sketch), `2` (very rough) | `1` |
| `opacity` | `0`–`100` | `100` |
| `fontSize` | Number in px | `16` |
| `fontFamily` | `1` (Virgil/handwritten), `2` (Helvetica), `3` (Cascadia/mono) | `1` |

### Coordinate system

Origin `(0, 0)` is top-left. `x` goes right, `y` goes down. Typical canvas sizes:
- Mobile: 375 × 812
- Desktop: 1280 × 800
- Tablet: 768 × 1024

### Example: full login screen

```bash
excalidraw-draw --inline '{
  "elements": [
    { "type": "text",      "x": 130, "y": 40,  "text": "Sign In", "fontSize": 28 },
    { "type": "text",      "x": 40,  "y": 110, "text": "Email",   "fontSize": 14 },
    { "type": "rectangle", "x": 40,  "y": 130, "width": 300, "height": 44, "roughness": 0, "backgroundColor": "#f8f8f8", "fillStyle": "solid" },
    { "type": "text",      "x": 40,  "y": 200, "text": "Password","fontSize": 14 },
    { "type": "rectangle", "x": 40,  "y": 220, "width": 300, "height": 44, "roughness": 0, "backgroundColor": "#f8f8f8", "fillStyle": "solid" },
    { "type": "rectangle", "x": 40,  "y": 290, "width": 300, "height": 48, "roughness": 0, "backgroundColor": "#4361ee", "strokeColor": "#4361ee", "fillStyle": "solid", "label": "Log In" },
    { "type": "text",      "x": 130, "y": 360, "text": "Forgot password?", "fontSize": 13, "strokeColor": "#4361ee" }
  ]
}' -o login.excalidraw

excalidraw-to-image login.excalidraw -o login.png
```

---

## Integration with Claude Code

Add this to your global `~/.claude/CLAUDE.md` (create if it doesn't exist):

````markdown
## Excalidraw tools

Two commands are available globally: `excalidraw-to-image` and `excalidraw-draw`.

---

### Reading a file

When the user mentions a `.excalidraw` or `.excalidraw.md` file, always:
1. Run: `excalidraw-to-image "path/to/file.excalidraw.md" -o "/tmp/preview.png"`
2. Read the PNG with the Read tool
3. Never read the raw text of an Excalidraw file without rendering it first

To zoom into a region of a large image, use Puppeteer to crop it:
```javascript
// node /tmp/crop.js
const puppeteer = require('/path/to/excalidraw-to-image/node_modules/puppeteer');
const fs = require('fs');
const [cropX, cropY, cropW, cropH] = [0, 200, 800, 600]; // pixels on the full image
(async () => {
  const browser = await puppeteer.launch({ headless: true, args: ['--no-sandbox'] });
  const page = await browser.newPage();
  await page.setViewport({ width: cropW, height: cropH });
  const b64 = fs.readFileSync('/tmp/preview.png').toString('base64');
  await page.setContent(`<html><body style="margin:0;overflow:hidden">
    <img src="data:image/png;base64,${b64}"
         style="position:absolute;top:-${cropY}px;left:-${cropX}px">
  </body></html>`);
  await new Promise(r => setTimeout(r, 400));
  fs.writeFileSync('/tmp/crop.png', await page.screenshot());
  await browser.close();
})();
```

---

### Drawing

**Workflow:**
1. **Declare** in plain text what you are going to draw
2. **Generate** the file:
   ```bash
   # Plain .excalidraw (for rendering)
   excalidraw-draw --inline '<spec JSON>' -o "/tmp/drawing.excalidraw"
   # Obsidian-compatible (openable directly in the vault)
   excalidraw-draw --inline '<spec JSON>' -o "/path/to/vault/drawing.excalidraw.md"
   ```
3. **Render** to PNG:
   ```bash
   excalidraw-to-image "/tmp/drawing.excalidraw" -o "/tmp/drawing.png"
   ```
4. **Compare** with the Read tool — if the render does not match the declaration, fix the spec and regenerate

---

### Spec format

```json
{
  "background": "#ffffff",
  "elements": [
    { "type": "rectangle", "x": 0,   "y": 0,   "width": 300, "height": 60, "label": "Header" },
    { "type": "ellipse",   "x": 0,   "y": 80,  "width": 60,  "height": 60 },
    { "type": "diamond",   "x": 100, "y": 80,  "width": 100, "height": 60, "label": "Decision?" },
    { "type": "text",      "x": 0,   "y": 160, "text": "A note", "fontSize": 14 },
    { "type": "arrow",     "x": 50,  "y": 60,  "points": [[0,0],[0,80]] },
    { "type": "line",      "x": 0,   "y": 200, "points": [[0,0],[300,0]] },
    { "type": "frame",     "x": 0,   "y": 0,   "width": 400, "height": 300, "label": "Section" }
  ]
}
```

**Element types:**

| Type | Required | Notes |
|------|----------|-------|
| `rectangle` | `x y width height` | `label` adds centered text inside |
| `ellipse` | `x y width height` | `label` adds centered text inside |
| `diamond` | `x y width height` | `label` adds centered text inside |
| `text` | `x y text` | standalone text |
| `arrow` | `x y points` | `points` are relative to `x,y` — e.g. `[[0,0],[100,50]]` |
| `line` | `x y points` | same as arrow, no arrowhead |
| `frame` | `x y width height` | named container, `label` sets the title |

**Style properties (all optional):**

| Property | Values | Default |
|----------|--------|---------|
| `strokeColor` | any hex | `#1e1e1e` |
| `backgroundColor` | any hex or `transparent` | `transparent` |
| `fillStyle` | `solid` `hachure` `cross-hatch` `dots` | `solid` |
| `strokeWidth` | `1` `2` `4` | `2` |
| `strokeStyle` | `solid` `dashed` `dotted` | `solid` |
| `roughness` | `0` clean · `1` sketch · `2` very rough | `1` |
| `opacity` | `0`–`100` | `100` |
| `fontSize` | px | `16` |
| `fontFamily` | `1` Virgil (handwritten) · `2` Helvetica · `3` Cascadia (mono) | `1` |

**Coordinate system:** origin (0,0) top-left, x → right, y → down.
````

---

## Project structure

```
├── cli.js                   # excalidraw-to-image entry point
├── draw-cli.js              # excalidraw-draw entry point
├── build-renderer.js        # esbuild script (runs on postinstall)
├── src/
│   ├── decompress.js        # LZ-string decompression (read)
│   ├── render.js            # Puppeteer PNG rendering (read)
│   ├── create.js            # Excalidraw element factory (draw)
│   └── browser/
│       └── renderer.js      # Browser-side export code (bundled)
└── dist/
    └── renderer.bundle.js   # Generated on install, do not commit
```

---

## Troubleshooting

**`command not found`** — run `npm link` inside the project directory.

**`Renderer bundle not found`** — run `npm run build` manually.

**Slow first render** — Puppeteer initialises Chromium on the first call; subsequent runs are faster.

**Text renders as boxes** — make sure `@excalidraw/excalidraw >= 0.18.0` is installed (`npm install` inside the project).

---

## License

MIT