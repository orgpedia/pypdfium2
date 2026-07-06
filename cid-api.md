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

The following script demonstrates how to load a PDF, iterate through text items on a page, read their glyph indices, extract the raw font program, compute styles (including synthetic bold), and check bounding boxes:

```python
import ctypes
import hashlib
import tempfile
from pathlib import Path
import pypdfium2 as pdfium

# 1. Load document and page
doc = pdfium.PdfDocument("sample.pdf")
page = doc[0]
textpage = page.get_textpage()

# 2. Iterate over characters and print styles/geometry
print(f"{'Idx':<4} | {'Char':<6} | {'CID/Code':<8} | {'Glyph ID':<8} | {'Font Obj':<8} | {'Size':<5} | {'Bold':<4} | {'Italic':<6} | {'Bbox':<28} | {'Font Name'}")
print("-" * 105)
for idx in range(min(15, textpage.count_items())):
    item = textpage.get_item(idx)
    char = chr(item.unicode) if item.unicode else ""
    char_repr = repr(char)
    
    # 2a. Determine Bold style (Design Bold, ForceBold flag, or Synthetic Stroke Bold)
    is_bold_val = (item.font_weight >= 700 or "bold" in (item.font_name or "").lower() or bool(item.font_flags & 262144))
    if not is_bold_val:
        textobj = textpage.get_textobj(idx)
        if textobj:
            render_mode = pdfium.raw.FPDFTextObj_GetTextRenderMode(textobj)
            line_width = ctypes.c_float()
            pdfium.raw.FPDFPageObj_GetStrokeWidth(textobj, ctypes.byref(line_width))
            if render_mode in (1, 2, 5, 6) and line_width.value > 0:
                is_bold_val = True
    
    # 2b. Determine Italic style (Italic flag, or Italic/Oblique in name)
    is_italic_val = bool(item.font_flags & 64) or any(x in (item.font_name or "").lower() for x in ("italic", "oblique"))
    
    bold_str = "Yes" if is_bold_val else "No"
    italic_str = "Yes" if is_italic_val else "No"
    size_str = f"{item.font_size:.1f}" if item.font_size else "N/A"
    bbox_str = f"({item.bbox[0]:.1f}, {item.bbox[1]:.1f}, {item.bbox[2]:.1f}, {item.bbox[3]:.1f})" if item.bbox else "N/A"
    
    print(f"{idx:<4} | {char_repr:<6} | {item.char_code:<8} | {item.glyph_id:<8} | {item.font_obj_num:<8} | {size_str:<5} | {bold_str:<4} | {italic_str:<6} | {bbox_str:<28} | {item.font_name}")

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
