Blueprint decode quickstart
- Blueprint string layout: `BLUEPRINT:0,<layout>,<icon1>,<icon2>,<icon3>,<icon4>,<icon5>,0,<timestampTicks>,<gameVersion>,<urlEncoded shortDesc>,<urlEncoded desc>"<base64GzipBinary><checksum>`. The icon slots are always five entries; unused ones are `0`. Description fields are URL-encoded UTF-8.
- Header decoding: split the string on `"` to isolate header/base64/checksum. Then split the header on `,` and run URL decode on the last two fields to read the title/description text.
- Binary payload layout (little-endian, gzip-compressed before base64):
  - Meta: int32 version; int32 cursorOffset.x; int32 cursorOffset.y; int32 cursorTargetArea; int32 dragBoxSize.x; int32 dragBoxSize.y; int32 primaryAreaIdx.
  - Areas: uint8 numAreas, then per area: int8 index; int8 parentIndex; int16 tropicAnchor; int16 areaSegments; int16 anchorLocalOffset.x; int16 anchorLocalOffset.y; int16 size.x; int16 size.y.
  - Buildings: int32 numBuildings, then per building:
    - int32 index; int8 areaIndex; float32 localOffset[2][3] (two vec3s); float32 yaw[2]; int16 itemId; int16 modelIndex; int32 outputObjIdx; int32 inputObjIdx; int8 outputToSlot; int8 inputFromSlot; int8 outputFromSlot; int8 inputToSlot; int8 outputOffset; int8 inputOffset; int16 recipeId; int16 filterId.
    - int16 paramLen, followed by `paramLen` int32s. Parameters are parsed per itemId using the parsers in `Scripts/blueprint.js` (`parameterParsers` map) to interpret belts, sorters, labs, etc.
- Handy Python snippet (requires gzip/base64/struct) to dump a blueprint string:
```python
import base64, gzip, struct, urllib.parse
from collections import Counter
from pathlib import Path

bp = Path("issues.md").read_text()  # or any source
header, b64, checksum = bp.split('"')
parts = [urllib.parse.unquote(p) for p in header.split(",")]
print("header parts:", parts, "checksum:", checksum)

buf = gzip.decompress(base64.b64decode(b64))
pos = 0
def i8():  # signed byte
    global pos
    val = struct.unpack_from("<b", buf, pos)[0]; pos += 1; return val
def u8():
    global pos
    val = struct.unpack_from("<B", buf, pos)[0]; pos += 1; return val
def i16():
    global pos
    val = struct.unpack_from("<h", buf, pos)[0]; pos += 2; return val
def i32():
    global pos
    val = struct.unpack_from("<i", buf, pos)[0]; pos += 4; return val
def f32():
    global pos
    val = struct.unpack_from("<f", buf, pos)[0]; pos += 4; return val

meta = [i32() for _ in range(7)]
areas = [dict(index=i8(), parentIndex=i8(), tropicAnchor=i16(),
              areaSegments=i16(), anchorLocalOffset={"x": i16(), "y": i16()},
              size={"x": i16(), "y": i16()}) for _ in range(u8())]
buildings = []
for _ in range(i32()):
    b = dict(index=i32(), areaIndex=i8(),
             localOffset0=dict(x=f32(), y=f32(), z=f32()),
             localOffset1=dict(x=f32(), y=f32(), z=f32()),
             yaw0=f32(), yaw1=f32(), itemId=i16(), modelIndex=i16(),
             outputObjIdx=i32(), inputObjIdx=i32(),
             outputToSlot=i8(), inputFromSlot=i8(),
             outputFromSlot=i8(), inputToSlot=i8(),
             outputOffset=i8(), inputOffset=i8(),
             recipeId=i16(), filterId=i16())
    param_len = i16()
    b["params"] = struct.unpack_from("<" + "i"*param_len, buf, pos)
    pos += 4*param_len
    buildings.append(b)
print("areas:", areas)
print("buildings:", len(buildings), "items:", Counter(b["itemId"] for b in buildings))
```
- Mapping itemIds to names: `Scripts/blueprint.js` has `buildingMap` with `itemId` per entry. Quick grab: `rg "itemId:" Scripts/blueprint.js` or a regex in Python to build `itemId -> name`. Use that to label counts or recipeIds.
- Common hurdles:
  - Set `PYTHONIOENCODING=utf-8` on Windows to avoid cp1252 errors when printing decoded Chinese text.
  - The payload is not JSON; it is a binary struct compressed with gzip. If JSON parsing fails, go straight to base64+gzip+struct.
  - Icons/layout: five icon slots follow `layout` in the header; unused ones are zeros.
- Useful references in repo: `Scripts/blueprint.js` (binary schema, parameter parsers, building map) and `Scripts/data.js`/`data.json` (recipes/items if you need recipeId meaning).
