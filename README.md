# python-docx-footnotes

An extension for `python-docx` that adds native Microsoft Word footnotes and endnotes support. It bypasses current library limitations by manually handling the underlying Object XML mapping (`lxml`) correctly, ensuring document validity and styling.

## Key Features

- **Native Word Footnotes**: Creates true `<w:footnote>` elements visible and manageable in MS Word.
- **Native Word Endnotes**: Creates `<w:endnote>` elements appended appropriately to the document.
- **Smart Endnote Reusability**: If you re-use the exact same string for an endnote, it automatically inserts a cross-reference field linking back to the original endnote instead of duplicating it.
- **Numbering Styles**: Support for customizing Endnote cross-reference styles (e.g., lower Roman `i, ii` vs Arabic `1, 2`).
- **Robustness**: Fixes document corruption loops in MS Word that typically happen when manually editing OpenXML for footnotes without maintaining relationship ids properly.

## Prerequisites

```bash
pip install python-docx lxml
```

## How It Works

Adding footnotes and endnotes to `.docx` files programmatically is heavily intertwined with MS Word's internal relations map files (`_rels/document.xml.rels`, `footnotes.xml`, etc.). 

Because we edit inside the active zip file, the workflow is:
1. **Initialize**: Generate a clean docx template explicitly pre-styled for footnotes/endnotes (`create_template.py`).
2. **Inject Runs**: Add footnotes via the `FootnoteAdder` class in memory.
3. **Save**: Save your doc using standard `doc.save()`.
4. **Finalize**: Pass the saved file back to `FootnoteAdder` which extracts the zip, maps relations correctly to `footnotes.xml` & `endnotes.xml`, and compiles it back seamlessly.

## Quickstart Guide

### Step 1: Provide a Pre-styled Template

Microsoft Word requires specific internal XML settings for notes to render without corruption. You must **either** use an existing word file that already has had footnotes/endnotes added to it manually at least once, **or** programmatically generate a fresh template.

**Option A (Recommended): Create a new template automatically**
Run this script once to generate `footnote_template.docx` with all mandatory `FootnoteText`/`EndnoteReference` styles predefined:
```bash
python create_template.py
```

**Option B: Manual creation**
If you want to use an existing customized Word document as your base:
1. Open your document in MS Word.
2. Manually add *one* footnote and *one* endnote anywhere in the document.
3. Delete the text of those notes so it appears empty again (the hidden styles will remain).
4. Save the file and use its path in Step 2.

### Step 2: Add Footnotes & Endnotes

Use the `FootnoteAdder` class alongside your standard `python-docx` Document logic.

```python
from docx import Document
from footnote_adder import FootnoteAdder

# 1. Load the pre-configured template
doc = Document("footnote_template.docx")
doc._body.clear_content()

# 2. Instantiate the FootnoteAdder 
# Optionally set endnote_style to 'roman' (default) or 'arabic' 
adder = FootnoteAdder(endnote_style="roman")

# 3. Standard Paragraph creation
p = doc.add_paragraph()

# 4. Add Footnote
# Signature: add_footnote(paragraph_object, text_before_note, note_text)
adder.add_footnote(p, "Here is a standard statement", "This is the text for the footnote at the bottom of the page.")

# 5. Add Endnote (Multiple Possibilities)

# A. Basic Endnote
# Signature: add_endnote(paragraph_object, text_before_note, note_text)
adder.add_endnote(p, " And a conclusion.", "This is the text for endnote #1.")

# B. Add Multiple Distinct Endnotes
adder.add_endnote(p, " Moving on to the next point.", "This is the text for endnote #2.")

# C. Smart Re-use (Cross-referencing) Across Paragraphs
# If you pass the EXACT SAME endnote text, it won't duplicate the endnote.
# Instead, it will create a Word Native Cross-Reference (NOTEREF field) pointing back to the first one!
p2 = doc.add_paragraph()
adder.add_endnote(p2, "Referring back to the first claim...", "This is the text for endnote #1.")

# D. Smart Re-use Inside the Same Paragraph
p3 = doc.add_paragraph()
adder.add_endnote(p3, "We mention point 2", "This is the text for endnote #2.")
adder.add_endnote(p3, " and again we mention point 2 right away.", "This is the text for endnote #2.")

# 6. Save the Document object
output_filename = "my_final_document.docx"
doc.save(output_filename)

# 7. CRITICAL: Initialize XML linking on the saved document
adder.finalize_footnotes(output_filename)

print("Document generated successfully!")
```

## Demonstrations

Two sample test cases are provided: 
1. `python example.py` - Basic standard footnote/endnote layout.
2. `python test_endnotes.py` - Strict testing scenario validating Endnote Sharing/Reference Reusability accross different configurations.
