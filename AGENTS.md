# Repository Guidelines
This is a repo to generate blueprints for the game 'dyson sphere program'.

## Project Structure & Module Organization
- `index.html` is the entry point; the page pulls scripts and styles from `Scripts/` and uses static assets from `img/`.
- `Scripts/blueprint.js` and `Scripts/data.js` contain the main logic and datasets; keep library order in `index.html` (jQuery -> Vue -> helpers -> data -> blueprint).
- `quote/` holds standalone HTML info pages (advertise, explanation, update notes) linked from the main site.
- GitHub Pages deploys via `.github/workflows/jekyll-gh-pages.yml`, emitting the built site to `_site` during CI.

## Build, Test, and Development Commands
- Open locally by serving the root directory: `python -m http.server 8000` then visit `http://localhost:8000/index.html`.
- Optional Pages parity if Jekyll is installed: `bundle exec jekyll build` (source `./`, output `_site`). No Gemfile is checked in, so rely on your system Jekyll.
- For quick edits you can also open `index.html` directly in a browser, but prefer a local server for correct relative paths.

## Coding Style & Naming Conventions
- Plain HTML/JS/CSS; no bundler or transpiler. Match existing 2-space indentation and avoid auto-formatting large data files.
- Use `const`/`let` and camelCase for variables and data keys, mirroring `itemMap` in `Scripts/blueprint.js`.
- Keep asset references relative to the repo root and avoid adding heavy libraries unless necessary.

## Testing Guidelines
- Manual smoke test after changes: load the page locally, exercise dropdowns/buttons, and confirm no console errors.
- When touching data sources (`Scripts/data.js` or `Scripts/data.json`), verify counts/rendering still look correct and that pako decompression succeeds.

## Commit & Pull Request Guidelines
- Write concise, imperative commits (e.g., `Update blueprint data`, `Tweak dropdown labels`). Group related changes only.
- PRs should describe the intent, list key files touched, and link any related issue. Attach screenshots/GIFs for UI updates.
- Ensure manual checks above are done before requesting review; keep PRs small and focused.

## Security & Assets
- Do not commit secrets or API keys; this site is fully static. Place new images in `img/` and optimize them before adding.
- Large data updates should stay compressed/structured similarly to existing files to keep load time reasonable.

---

## Blueprint Decoding Cheat Sheet
- Blueprint string layout: `BLUEPRINT:0,<layout>,<icon1..5>,0,<timestampTicks>,<gameVersion>,<urlEncoded shortDesc>,<urlEncoded desc>"<base64GzipBinary><checksum>`. Split on `"` to isolate header/base64/checksum; URL-decode the last two header fields for title/description.
- Binary payload (little-endian, gzip then base64): meta (version; cursorOffset x/y; cursorTargetArea; dragBoxSize x/y; primaryAreaIdx), `numAreas` (u8) + per-area {index, parentIndex, tropicAnchor, areaSegments, anchorLocalOffset x/y, size x/y}, `numBuildings` (i32) + per-building {index; areaIndex; two vec3 localOffset; two yaw floats; itemId; modelIndex; outputObjIdx; inputObjIdx; slot bytes; recipeId; filterId; paramLen followed by paramLen int32 params}. Parameter decoding logic is in `Scripts/blueprint.js` (`parameterParsers`).
- Quick Python dump (set `PYTHONIOENCODING=utf-8` on Windows):
  ```python
  import base64, gzip, struct, urllib.parse
  from collections import Counter
  from pathlib import Path
  bp = Path("issues.md").read_text()  # or any source
  header, b64, checksum = bp.split('"')
  parts = [urllib.parse.unquote(p) for p in header.split(",")]
  buf = gzip.decompress(base64.b64decode(b64))
  pos=0
  def i32(): global pos; v=struct.unpack_from("<i", buf, pos)[0]; pos+=4; return v
  def i16(): global pos; v=struct.unpack_from("<h", buf, pos)[0]; pos+=2; return v
  def i8():  global pos; v=struct.unpack_from("<b", buf, pos)[0]; pos+=1; return v
  def u8():  global pos; v=struct.unpack_from("<B", buf, pos)[0]; pos+=1; return v
  def f32(): global pos; v=struct.unpack_from("<f", buf, pos)[0]; pos+=4; return v
  meta=[i32() for _ in range(7)]
  areas=[dict(index=i8(), parentIndex=i8(), tropicAnchor=i16(), areaSegments=i16(),
              anchorLocalOffset={"x": i16(), "y": i16()}, size={"x": i16(), "y": i16()})
         for _ in range(u8())]
  buildings=[]
  for _ in range(i32()):
      pos+=4+1+4*6+4*2
      itemId=i16(); model=i16(); pos+=4*2
      pos+=6
      recipe=i16(); filterId=i16()
      param_len=i16(); pos+=param_len*4
      buildings.append(itemId)
  print("header:", parts, "areas:", areas, "buildings:", len(buildings), "counts:", Counter(buildings))
  ```
- Map `itemId -> name` via `buildingMap` in `Scripts/blueprint.js` (e.g., regex `([A-Za-z0-9_]+):\\s*\\{[^}]*?itemId:\\s*(\\d+)` in Python) to label counts.
- If JSON parsing fails, remember payload is binary gzip/base64, not JSON. For the full breakdown, see `blueprint-decode-cheatsheet.md`.

---
