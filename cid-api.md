# Custom CID & Glyph API Reference

This document provides an overview of the custom APIs added to the **`orgpedia`** forks of PDFium and `pypdfium2`. These extensions enable low-level character-to-glyph index mapping and raw embedded font extraction, designed specifically for Indic script layout analysis and rendering pipelines.

---

## 1. C/C++ API Additions (`orgpedia/pdfium`)

### `FPDF_TEXT_ITEM` Struct Extension
Exposes the resolved character-to-glyph index within the font file.
* **Header**: `public/fpdf_text.h`
* **Definition**:
  ```cpp
  typedef struct _FPDF_TEXT_ITEM {
    unsigned int char_code;  // Original character code (CID)
    unsigned int glyph_id;   // Resolved FreeType glyph index <-- ADDED
    float origin_x;
    float origin_y;
    // ... remaining geometry fields
  } FPDF_TEXT_ITEM;
  ```

### `FPDFFont_GetObjNum` Function
Retrieves the indirect PDF object number for a font dictionary.
* **Header**: `public/fpdf_edit.h`
* **Declaration**:
  ```cpp
  FPDF_EXPORT unsigned long FPDF_CALLCONV FPDFFont_GetObjNum(FPDF_FONT font);
  ```
* **Returns**: The PDF indirect object ID (integer `> 0`), or `0` if the font is a standard non-embedded font.

---

## 2. Python API Reference (`orgpedia/pypdfium2`)

### `PdfTextItem` Class (Data Class)
* **File**: `src/pypdfium2/_helpers/textpage.py`
* **New Attribute**:
  * `glyph_id: int`: The resolved FreeType glyph index within the font program.

### `PdfFont` Class (Helper Extensions)
* **File**: `src/pypdfium2/_helpers/pageobjects.py`
* **New Methods**:
  * `get_obj_num(self) -> int`: 
    Returns the indirect PDF object number of the font.
  * `get_data(self) -> bytes | None`: 
    Extracts the raw embedded font program bytes (e.g., TrueType/CFF data). Returns `None` if no embedded data is available (or returns system substitution bytes if resolved).

---

## 3. Usage Example

The following script demonstrates how to load a PDF, iterate through text items on a page, read their glyph indices, extract the raw font program, and compute the exact same SHA-256 hash as used in `cid_pdf.py`:

```python
import hashlib
import tempfile
from pathlib import Path
import pypdfium2 as pdfium

# 1. Load document and page
doc = pdfium.PdfDocument("sample.pdf")
page = doc[0]
textpage = page.get_textpage()

# 2. Iterate over characters
print(f"Index | Char | CID/Code | Glyph ID | Font Obj | Font Name")
print("-" * 65)
for idx in range(min(10, textpage.count_items())):
    item = textpage.get_item(idx)
    char = chr(item.unicode) if item.unicode else ""
    print(f"{idx:<5} | {repr(char):<4} | {item.char_code:<8} | {item.glyph_id:<8} | {item.font_obj_num:<8} | {item.font_name}")

# 3. Extract Font Bytes & Object ID from a character's text object
textobj = textpage.get_textobj(0)
if textobj:
    font = textobj.get_font()
    if font:
        print(f"\nFont Base Name: {font.get_base_name()}")
        print(f"PDF Object ID : {font.get_obj_num()}")
        
        # Get raw font bytes
        font_bytes = font.get_data()
        if font_bytes:
            print(f"Extracted Size: {len(font_bytes)} bytes")
            print(f"SHA-256 Hash  : {hashlib.sha256(font_bytes).hexdigest()}")
```
