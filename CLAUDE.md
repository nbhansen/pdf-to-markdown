# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Python-based PDF to Markdown converter designed specifically for Retrieval Augmented Generation (RAG) and other NLP tasks. The tool extracts and preserves markdown-formatted content including text, tables, images, links, code blocks, lists, and formatting from PDF files.

## Development Setup

### Installation

```bash
# Create and activate virtual environment
python -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt
```

### System Dependencies

Tesseract OCR must be installed separately:
- Ubuntu: `sudo apt-get install tesseract-ocr`
- macOS: `brew install tesseract`
- Windows: Download from https://github.com/UB-Mannheim/tesseract/wiki

### Running the Extractor

```bash
# Basic usage - single output file
python extract.py --pdf_path path/to/your/file.pdf

# Split into multiple files (e.g., 10 pages per file)
python extract.py --pdf_path path/to/your/file.pdf --pages-per-file 10

# Split into individual page files
python extract.py --pdf_path path/to/your/file.pdf --split-pages
```

Output will be saved to `outputs/` directory as markdown files with the same name as the input PDF.

## Architecture

### Core Components

The codebase uses an object-oriented architecture with two main classes:

1. **PDFExtractor (Abstract Base Class)**: Located at extract.py:25-51
   - Defines the interface for PDF extraction
   - Handles logging setup
   - Abstract `extract()` method must be implemented by subclasses

2. **MarkdownPDFExtractor**: Located at extract.py:54-648
   - Concrete implementation that converts PDFs to markdown
   - Main extraction pipeline in `extract_markdown()` method (extract.py:98-169)

### Extraction Pipeline Flow

The extraction process follows this sequence:

1. **Table Extraction** (extract.py:171-186): Uses pdfplumber to extract all tables from the PDF first
2. **Page-by-Page Processing** (extract.py:112-163):
   - Iterates through each page using PyMuPDF (fitz)
   - Extracts text blocks and image blocks
   - Processes links from the page
   - Inserts tables at their approximate positions
3. **Block Processing**:
   - **Text blocks** (extract.py:411-545): Handles formatting, headers, lists, code blocks, links
   - **Image blocks** (extract.py:547-578): Extracts images, generates captions via OCR or AI model
4. **Post-processing** (extract.py:597-633): Cleans up excessive newlines, page numbers, duplicate horizontal lines

### Key Features Implementation

- **Text Formatting** (extract.py:277-303): Applies markdown syntax for bold, italic, monospace, superscript, subscript based on PDF font flags
- **Code Block Detection** (extract.py:340-409): Pattern-based detection for Python, JavaScript, HTML, Shell, Bash, C++, Java, JSON with language-specific regex patterns
- **Image Captioning** (extract.py:242-269): Two-tier approach:
  1. First attempts OCR using pytesseract
  2. Falls back to AI-generated captions using VisionEncoderDecoder model (nlpconnect/vit-gpt2-image-captioning)
- **Link Preservation** (extract.py:327-338): Extracts URI links from PDF and converts to markdown link syntax
- **Header Detection** (extract.py:580-595): Font size-based header level detection (24+ = H1, 20+ = H2, etc.)

### Configuration

Configuration is managed via `config/config.yaml`:
- `PAGE_DELIMITER`: Unique delimiter separating pages in output (default: `<||WXb23TXrUn3Rxz00yNNr89HV||>`)
- `OUTPUT_DIR`: Directory for output files (default: `outputs`)

### Logging

Logs are written to `logs/extract.log` with both file and console handlers. Logging is configured in extract.py:32-46.

## Important Implementation Details

### Header and Footer Skipping

The extractor skips content in the top 50 and bottom 50 points of each page to avoid headers/footers (extract.py:425-426).

### Image Size Limits

- Pages with more than 128 images are processed differently to avoid performance issues (extract.py:119)
- Images smaller than 20x20 pixels are skipped (extract.py:557-558)
- Tables are limited to 128 per page (extract.py:178-179)

### AI Model Usage

The extractor uses GPU if available (`torch.device("cuda" if torch.cuda.is_available() else "cpu")`) for the image captioning model (extract.py:77-78).

### Code Block State Management

Code block detection maintains state across lines using `in_code_block`, `code_block_content`, and `code_block_lang` variables to properly fence multi-line code blocks.

## Testing Strategy

When adding features or fixing bugs:
- Test with various PDF layouts (single column, multi-column, mixed content)
- Verify table extraction accuracy with complex table structures
- Test image captioning with both text-heavy images (triggers OCR) and graphic images (triggers AI model)
- Ensure code block detection works across all supported languages
- Validate that markdown output is properly formatted for RAG ingestion

## Performance Considerations

- Average processing time: 30-60 seconds for a 10-page PDF with mixed content
- Large PDFs (100+ pages) may require significant processing time
- Image captioning model loading happens once during initialization
- PyMuPDF (fitz) is used for fast text extraction; pdfplumber specifically for tables
