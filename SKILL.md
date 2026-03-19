---
name: word-footnotes
description: Create Microsoft Word (.docx) documents with native footnotes using Python and python-docx. Solves the limitation that python-docx doesn't support footnotes natively.
tags:
  - python
  - word
  - docx
  - footnotes
  - documents
  - office
---

# Word Documents with Footnotes

This skill enables creating Microsoft Word (.docx) documents with proper native footnotes using Python and the python-docx library.

## Overview

The python-docx library doesn't natively support footnotes. This skill provides a workaround by:
1. Creating a template with footnotes infrastructure
2. Adding footnote references in the document
3. Post-processing the saved document to inject the actual footnotes

## Requirements

```bash
pip install python-docx lxml
```

## Step 1: Create the Template

First, create a template document with footnotes infrastructure. Save this as `create_footnote_template.py`:

```python
"""
Create a Word template with footnotes infrastructure.
Run this ONCE to create the template.
"""

from docx import Document
from lxml import etree
import zipfile
import os
import shutil

def create_template_with_footnotes(template_path):
    """Create a template document with footnotes infrastructure."""

    # First create a basic document
    doc = Document()
    doc.add_paragraph("Template paragraph")

    # Save it temporarily
    temp_path = template_path + ".temp"
    doc.save(temp_path)

    # Extract the docx
    extract_dir = temp_path + "_extracted"
    with zipfile.ZipFile(temp_path, 'r') as zip_ref:
        zip_ref.extractall(extract_dir)

    word_dir = os.path.join(extract_dir, "word")
    w_ns = "http://schemas.openxmlformats.org/wordprocessingml/2006/main"

    # Create footnotes.xml - CRITICAL: declare all namespaces referenced in mc:Ignorable
    footnotes_xml = '''<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<w:footnotes xmlns:wpc="http://schemas.microsoft.com/office/word/2010/wordprocessingCanvas"
             xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
             xmlns:o="urn:schemas-microsoft-com:office:office"
             xmlns:r="http://schemas.openxmlformats.org/officeDocument/2006/relationships"
             xmlns:m="http://schemas.openxmlformats.org/officeDocument/2006/math"
             xmlns:v="urn:schemas-microsoft-com:vml"
             xmlns:wp14="http://schemas.microsoft.com/office/word/2010/wordprocessingDrawing"
             xmlns:wp="http://schemas.openxmlformats.org/drawingml/2006/wordprocessingDrawing"
             xmlns:w10="urn:schemas-microsoft-com:office:word"
             xmlns:w="http://schemas.openxmlformats.org/wordprocessingml/2006/main"
             xmlns:w14="http://schemas.microsoft.com/office/word/2010/wordml"
             xmlns:wne="http://schemas.microsoft.com/office/word/2006/wordml"
             mc:Ignorable="w14 wp14">
    <w:footnote w:type="separator" w:id="-1">
        <w:p><w:pPr><w:spacing w:after="0" w:line="240" w:lineRule="auto"/></w:pPr>
            <w:r><w:separator/></w:r></w:p>
    </w:footnote>
    <w:footnote w:type="continuationSeparator" w:id="0">
        <w:p><w:pPr><w:spacing w:after="0" w:line="240" w:lineRule="auto"/></w:pPr>
            <w:r><w:continuationSeparator/></w:r></w:p>
    </w:footnote>
</w:footnotes>'''
    with open(os.path.join(word_dir, "footnotes.xml"), "w", encoding="utf-8") as f:
        f.write(footnotes_xml)

    # Create endnotes.xml (Word expects this when footnotes exist)
    endnotes_xml = '''<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<w:endnotes xmlns:wpc="http://schemas.microsoft.com/office/word/2010/wordprocessingCanvas"
            xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
            xmlns:o="urn:schemas-microsoft-com:office:office"
            xmlns:r="http://schemas.openxmlformats.org/officeDocument/2006/relationships"
            xmlns:m="http://schemas.openxmlformats.org/officeDocument/2006/math"
            xmlns:v="urn:schemas-microsoft-com:vml"
            xmlns:wp14="http://schemas.microsoft.com/office/word/2010/wordprocessingDrawing"
            xmlns:wp="http://schemas.openxmlformats.org/drawingml/2006/wordprocessingDrawing"
            xmlns:w10="urn:schemas-microsoft-com:office:word"
            xmlns:w="http://schemas.openxmlformats.org/wordprocessingml/2006/main"
            xmlns:w14="http://schemas.microsoft.com/office/word/2010/wordml"
            xmlns:wne="http://schemas.microsoft.com/office/word/2006/wordml"
            mc:Ignorable="w14 wp14">
    <w:endnote w:type="separator" w:id="-1">
        <w:p><w:pPr><w:spacing w:after="0" w:line="240" w:lineRule="auto"/></w:pPr>
            <w:r><w:separator/></w:r></w:p>
    </w:endnote>
    <w:endnote w:type="continuationSeparator" w:id="0">
        <w:p><w:pPr><w:spacing w:after="0" w:line="240" w:lineRule="auto"/></w:pPr>
            <w:r><w:continuationSeparator/></w:r></w:p>
    </w:endnote>
</w:endnotes>'''
    with open(os.path.join(word_dir, "endnotes.xml"), "w", encoding="utf-8") as f:
        f.write(endnotes_xml)

    # Update [Content_Types].xml
    content_types_path = os.path.join(extract_dir, "[Content_Types].xml")
    tree = etree.parse(content_types_path)
    root = tree.getroot()
    ct_ns = "http://schemas.openxmlformats.org/package/2006/content-types"

    # Add footnotes and endnotes overrides
    for part, content_type in [
        ("/word/footnotes.xml", "application/vnd.openxmlformats-officedocument.wordprocessingml.footnotes+xml"),
        ("/word/endnotes.xml", "application/vnd.openxmlformats-officedocument.wordprocessingml.endnotes+xml")
    ]:
        override = etree.SubElement(root, "{%s}Override" % ct_ns)
        override.set("PartName", part)
        override.set("ContentType", content_type)
    tree.write(content_types_path, xml_declaration=True, encoding="UTF-8", standalone="yes")

    # Update styles.xml - add FootnoteReference and FootnoteText styles
    styles_path = os.path.join(word_dir, "styles.xml")
    styles_tree = etree.parse(styles_path)
    styles_root = styles_tree.getroot()

    # Add FootnoteReference style
    fn_ref_style = etree.SubElement(styles_root, "{%s}style" % w_ns)
    fn_ref_style.set("{%s}type" % w_ns, "character")
    fn_ref_style.set("{%s}styleId" % w_ns, "FootnoteReference")
    etree.SubElement(fn_ref_style, "{%s}name" % w_ns).set("{%s}val" % w_ns, "footnote reference")
    etree.SubElement(fn_ref_style, "{%s}basedOn" % w_ns).set("{%s}val" % w_ns, "DefaultParagraphFont")
    fn_ref_rPr = etree.SubElement(fn_ref_style, "{%s}rPr" % w_ns)
    etree.SubElement(fn_ref_rPr, "{%s}vertAlign" % w_ns).set("{%s}val" % w_ns, "superscript")

    # Add FootnoteText style
    fn_text_style = etree.SubElement(styles_root, "{%s}style" % w_ns)
    fn_text_style.set("{%s}type" % w_ns, "paragraph")
    fn_text_style.set("{%s}styleId" % w_ns, "FootnoteText")
    etree.SubElement(fn_text_style, "{%s}name" % w_ns).set("{%s}val" % w_ns, "footnote text")
    etree.SubElement(fn_text_style, "{%s}basedOn" % w_ns).set("{%s}val" % w_ns, "Normal")
    fn_text_pPr = etree.SubElement(fn_text_style, "{%s}pPr" % w_ns)
    spacing = etree.SubElement(fn_text_pPr, "{%s}spacing" % w_ns)
    spacing.set("{%s}after" % w_ns, "0")
    spacing.set("{%s}line" % w_ns, "240")
    spacing.set("{%s}lineRule" % w_ns, "auto")
    fn_text_rPr = etree.SubElement(fn_text_style, "{%s}rPr" % w_ns)
    etree.SubElement(fn_text_rPr, "{%s}sz" % w_ns).set("{%s}val" % w_ns, "20")
    etree.SubElement(fn_text_rPr, "{%s}szCs" % w_ns).set("{%s}val" % w_ns, "20")

    styles_tree.write(styles_path, xml_declaration=True, encoding="UTF-8", standalone="yes")

    # Update document.xml.rels - add footnotes and endnotes relationships
    rels_path = os.path.join(word_dir, "_rels", "document.xml.rels")
    rels_tree = etree.parse(rels_path)
    rels_root = rels_tree.getroot()
    rel_ns = "http://schemas.openxmlformats.org/package/2006/relationships"

    existing_ids = [int(el.get("Id")[3:]) for el in rels_root if el.get("Id", "").startswith("rId")]
    next_id = max(existing_ids) + 1 if existing_ids else 1

    for target, rel_type in [
        ("footnotes.xml", "http://schemas.openxmlformats.org/officeDocument/2006/relationships/footnotes"),
        ("endnotes.xml", "http://schemas.openxmlformats.org/officeDocument/2006/relationships/endnotes")
    ]:
        rel = etree.SubElement(rels_root, "{%s}Relationship" % rel_ns)
        rel.set("Id", f"rId{next_id}")
        rel.set("Type", rel_type)
        rel.set("Target", target)
        next_id += 1

    rels_tree.write(rels_path, xml_declaration=True, encoding="UTF-8", standalone="yes")

    # Update settings.xml - add footnotePr and endnotePr
    settings_path = os.path.join(word_dir, "settings.xml")
    settings_tree = etree.parse(settings_path)
    settings_root = settings_tree.getroot()

    footnotePr = etree.Element("{%s}footnotePr" % w_ns)
    etree.SubElement(footnotePr, "{%s}footnote" % w_ns).set("{%s}id" % w_ns, "-1")
    etree.SubElement(footnotePr, "{%s}footnote" % w_ns).set("{%s}id" % w_ns, "0")

    endnotePr = etree.Element("{%s}endnotePr" % w_ns)
    etree.SubElement(endnotePr, "{%s}endnote" % w_ns).set("{%s}id" % w_ns, "-1")
    etree.SubElement(endnotePr, "{%s}endnote" % w_ns).set("{%s}id" % w_ns, "0")

    # Insert after characterSpacingControl or at beginning
    insert_idx = 0
    for i, elem in enumerate(settings_root):
        if 'characterSpacingControl' in elem.tag:
            insert_idx = i + 1
            break
    settings_root.insert(insert_idx, footnotePr)
    settings_root.insert(insert_idx + 1, endnotePr)

    settings_tree.write(settings_path, xml_declaration=True, encoding="UTF-8", standalone="yes")

    # Repack the docx
    with zipfile.ZipFile(template_path, 'w', zipfile.ZIP_DEFLATED) as zipf:
        for root_dir, dirs, files in os.walk(extract_dir):
            for file in files:
                file_path = os.path.join(root_dir, file)
                arcname = os.path.relpath(file_path, extract_dir)
                zipf.write(file_path, arcname)

    # Clean up
    os.remove(temp_path)
    shutil.rmtree(extract_dir)

    print(f"Template created: {template_path}")

if __name__ == "__main__":
    create_template_with_footnotes("footnote_template.docx")
```

The template is ready to use immediately after creation.

## Step 2: FootnoteAdder Class

Use this class in your document generation scripts:

```python
from docx import Document
from docx.oxml.ns import qn
from docx.oxml import OxmlElement
from lxml import etree
import zipfile
import tempfile
import shutil
import os
import re

class FootnoteAdder:
    """Helper class to add real Word footnotes to documents."""

    def __init__(self):
        self.footnote_id = 0
        self.footnotes_to_add = []

    def add_footnote(self, paragraph, text, footnote_text):
        """Add a footnote reference to a paragraph.

        Args:
            paragraph: The python-docx paragraph object
            text: Text to add before the footnote reference (can be empty string)
            footnote_text: The footnote content
        """
        self.footnote_id += 1

        if text:
            paragraph.add_run(text)

        # Create the footnote reference run
        footnote_run = paragraph.add_run()
        r = footnote_run._r

        # Add run properties with FootnoteReference style
        rPr = OxmlElement('w:rPr')
        rStyle = OxmlElement('w:rStyle')
        rStyle.set(qn('w:val'), 'FootnoteReference')
        rPr.append(rStyle)
        r.insert(0, rPr)

        # Add the footnote reference element
        footnote_ref = OxmlElement('w:footnoteReference')
        footnote_ref.set(qn('w:id'), str(self.footnote_id))
        r.append(footnote_ref)

        self.footnotes_to_add.append((self.footnote_id, footnote_text))
        return footnote_run

    def finalize_footnotes(self, docx_path):
        """Add all queued footnotes to the document. Call after doc.save()."""
        if not self.footnotes_to_add:
            return

        extract_dir = tempfile.mkdtemp()

        try:
            with zipfile.ZipFile(docx_path, 'r') as zip_ref:
                zip_ref.extractall(extract_dir)

            # Update footnotes.xml
            footnotes_path = os.path.join(extract_dir, "word", "footnotes.xml")
            tree = etree.parse(footnotes_path)
            root = tree.getroot()
            w_ns = "http://schemas.openxmlformats.org/wordprocessingml/2006/main"

            for fn_id, fn_text in self.footnotes_to_add:
                footnote = etree.SubElement(root, "{%s}footnote" % w_ns)
                footnote.set("{%s}id" % w_ns, str(fn_id))

                p = etree.SubElement(footnote, "{%s}p" % w_ns)
                pPr = etree.SubElement(p, "{%s}pPr" % w_ns)
                pStyle = etree.SubElement(pPr, "{%s}pStyle" % w_ns)
                pStyle.set("{%s}val" % w_ns, "FootnoteText")

                # Footnote reference mark
                r1 = etree.SubElement(p, "{%s}r" % w_ns)
                rPr1 = etree.SubElement(r1, "{%s}rPr" % w_ns)
                rStyle1 = etree.SubElement(rPr1, "{%s}rStyle" % w_ns)
                rStyle1.set("{%s}val" % w_ns, "FootnoteReference")
                etree.SubElement(r1, "{%s}footnoteRef" % w_ns)

                # Space after reference
                r2 = etree.SubElement(p, "{%s}r" % w_ns)
                t2 = etree.SubElement(r2, "{%s}t" % w_ns)
                t2.set("{http://www.w3.org/XML/1998/namespace}space", "preserve")
                t2.text = " "

                # Footnote text
                r3 = etree.SubElement(p, "{%s}r" % w_ns)
                t3 = etree.SubElement(r3, "{%s}t" % w_ns)
                t3.text = fn_text

            tree.write(footnotes_path, xml_declaration=True, encoding="UTF-8", standalone="yes")

            # Clean up Mac-specific content and fix XML
            self._cleanup_docx(extract_dir)

            # Repack with proper file order
            self._repack_docx(extract_dir, docx_path)

        finally:
            shutil.rmtree(extract_dir)

    def _cleanup_docx(self, extract_dir):
        """Remove Mac-specific elements and fix XML formatting."""

        for root_dir, dirs, files in os.walk(extract_dir):
            for file in files:
                if file.endswith('.xml') or file.endswith('.rels'):
                    file_path = os.path.join(root_dir, file)
                    with open(file_path, 'r', encoding='utf-8') as f:
                        content = f.read()

                    modified = False

                    # Remove Mac namespaces
                    if 'xmlns:mo=' in content or 'xmlns:mv=' in content:
                        content = re.sub(r'\s*xmlns:mo="[^"]*"', '', content)
                        content = re.sub(r'\s*xmlns:mv="[^"]*"', '', content)
                        modified = True

                    # Fix single-quote XML declarations
                    if "<?xml version='1.0'" in content:
                        content = content.replace(
                            "<?xml version='1.0' encoding='UTF-8' standalone='yes'?>",
                            '<?xml version="1.0" encoding="UTF-8" standalone="yes"?>'
                        )
                        modified = True

                    if modified:
                        with open(file_path, 'w', encoding='utf-8') as f:
                            f.write(content)

        # Fix webSettings.xml
        ws_path = os.path.join(extract_dir, "word", "webSettings.xml")
        if os.path.exists(ws_path):
            with open(ws_path, 'r', encoding='utf-8') as f:
                content = f.read()
            content = re.sub(r'\s*<w:doNotSaveAsSingleFile/>', '', content)
            with open(ws_path, 'w', encoding='utf-8') as f:
                f.write(content)

        # Fix settings.xml
        settings_path = os.path.join(extract_dir, "word", "settings.xml")
        if os.path.exists(settings_path):
            with open(settings_path, 'r', encoding='utf-8') as f:
                content = f.read()
            content = re.sub(r'<w:zoom w:val="bestFit"/>', '<w:zoom w:percent="100"/>', content)
            with open(settings_path, 'w', encoding='utf-8') as f:
                f.write(content)

        # Fix docProps/app.xml
        app_path = os.path.join(extract_dir, "docProps", "app.xml")
        if os.path.exists(app_path):
            with open(app_path, 'r', encoding='utf-8') as f:
                content = f.read()
            content = content.replace('Microsoft Macintosh Word', 'Microsoft Office Word')
            content = content.replace('<Manager/>', '<Manager></Manager>')
            content = content.replace('<Company/>', '<Company></Company>')
            content = content.replace('<HyperlinkBase/>', '<HyperlinkBase></HyperlinkBase>')
            with open(app_path, 'w', encoding='utf-8') as f:
                f.write(content)

    def _repack_docx(self, extract_dir, docx_path):
        """Repack the docx with proper OOXML file order."""
        all_files = []
        for root_dir, dirs, files in os.walk(extract_dir):
            for file in files:
                file_path = os.path.join(root_dir, file)
                arcname = os.path.relpath(file_path, extract_dir).replace('\\', '/')
                all_files.append((file_path, arcname))

        # OOXML requires specific file order
        priority_order = [
            '[Content_Types].xml',
            '_rels/.rels',
            'word/_rels/document.xml.rels',
            'word/document.xml',
            'word/footnotes.xml',
            'word/endnotes.xml',
        ]

        def sort_key(item):
            try:
                return (priority_order.index(item[1]), item[1])
            except ValueError:
                return (len(priority_order), item[1])

        all_files.sort(key=sort_key)

        temp_docx = docx_path + ".tmp"
        with zipfile.ZipFile(temp_docx, 'w', zipfile.ZIP_DEFLATED) as zipf:
            for file_path, arcname in all_files:
                zipf.write(file_path, arcname)

        os.replace(temp_docx, docx_path)
```

## Step 3: Usage Example

```python
from docx import Document

# Load the template
doc = Document("footnote_template.docx")

# Clear template content
doc._body.clear_content()

# Create footnote adder
footnote_adder = FootnoteAdder()

# Add content with footnotes
doc.add_heading('My Document', 0)

p = doc.add_paragraph()
p.add_run("This is some text that needs a citation")
footnote_adder.add_footnote(p, "", "Author, Book Title (Publisher, 2024), p. 42.")

p2 = doc.add_paragraph()
p2.add_run("Here is another statement")
footnote_adder.add_footnote(p2, "", "Another Author, Another Book (Publisher, 2023).")

# Save and finalize
output_path = "my_document.docx"
doc.save(output_path)
footnote_adder.finalize_footnotes(output_path)

print(f"Document saved to {output_path}")
```

## Key Points

1. **Call finalize_footnotes() after save()**: The footnotes are injected into the saved file, so you must call `finalize_footnotes()` after `doc.save()`.

2. **Namespace declarations are critical**: The `footnotes.xml` and `endnotes.xml` files must declare all namespaces referenced in `mc:Ignorable`. Missing `xmlns:w14` or `xmlns:wp14` will cause "unreadable content" errors.

## Troubleshooting

If Word shows "unreadable content" errors:
- Verify `footnotes.xml` and `endnotes.xml` declare `xmlns:w14` and `xmlns:wp14` (required for `mc:Ignorable="w14 wp14"`)
- Check that all Mac-specific namespaces (xmlns:mo, xmlns:mv) are being removed
- Verify the file order in the zip is correct (Content_Types first, then _rels/.rels)
- Ensure XML declarations use double quotes, not single quotes
