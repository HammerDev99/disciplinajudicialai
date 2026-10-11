# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

DisciplinaJudicial AI is a Streamlit-based AI assistant specialized in **disciplinary guidance for Colombian judicial servants**. The application provides normative orientation, document processing via OCR, and conversational support for judges, magistrates, court employees, and lawyers regarding the disciplinary regime of the Judicial Branch (Comisión Nacional de Disciplina Judicial, Comisiones Seccionales, Código General Disciplinario).

### Core Technologies
- **Framework**: Streamlit (web UI)
- **AI Services**: OpenAI Responses API (SDK `openai>=3`), Mistral OCR API
- **Document Processing**: PyPDF2, Pillow, FPDF2, ReportLab
- **Python Version**: 3.8+

## Application Architecture

### Single-File Application Structure

The entire application logic resides in `app.py` (~2100 lines), organized into these logical sections:

1. **APP_IDENTITY Dictionary (lines 22-155)**: Centralized configuration for all UI text, branding, URLs, and developer information. Modify this dictionary to change application identity, messages, or links.

2. **Logging & Configuration (lines 158-173)**: Multi-destination logging setup with daily log rotation in `logs/` directory.

3. **Analytics Injection (lines 176-195)**: Umami analytics integration via `ANALYTICS_SCRIPT` secret, injected into parent document.

4. **Error Handling Framework (lines 211-247)**: `@handle_error` decorator with retry logic for all critical functions.

5. **Document Processing Pipeline** (lines 732-1321):
   - File validation and format detection (PDF, Image, Text)
   - Mistral OCR integration with retry logic
   - Image optimization for better OCR results
   - Multi-strategy text extraction from OCR responses

6. **PDF Export System** (lines 329-703):
   - Primary: FPDF with markdown processing
   - Secondary: ReportLab fallback
   - Tertiary: Simple FPDF fallback
   - Final fallback: Markdown export

7. **OpenAI Integration** (lines 1363-1667):
   - Thread-based conversation management
   - Document context injection into prompts
   - Message processing with citation handling
   - Streaming status updates

### State Management

Key `st.session_state` variables:
- `thread_id`: OpenAI conversation thread
- `messages`: Chat history array
- `uploaded_files`: List of processed filenames
- `document_contents`: Dict mapping filenames to OCR-extracted content
- `file_metadata`: Dict mapping OpenAI file IDs to metadata

### Document Processing Flow

1. User uploads file via chat input
2. `validate_file_format()` checks extension and file signature
3. `process_document_with_mistral_ocr()` sends to Mistral API
4. `extract_text_from_ocr_response()` parses response (supports markdown, text, elements formats)
5. Content stored in `st.session_state.document_contents`
6. `send_message_with_document_context()` injects document text into OpenAI prompt

## Configuration

### Required Secrets (.streamlit/secrets.toml or environment variables)

```toml
OPENAI_API_KEY = "sk-..."
MISTRAL_API_KEY = "..."
VECTOR_STORE_ID = "vs_..."              # Optional: knowledge base for file_search (replaces ASSISTANT_ID)
ANALYTICS_SCRIPT = "<script defer src='...' data-website-id='...'></script>"  # Optional
```

### Deployment Environments

The app detects Streamlit Cloud via `is_streamlit_cloud()` checking multiple indicators:
- `STREAMLIT_SHARING_MODE` env var
- `STREAMLIT_SERVER_BASE_URL_IS_SET`
- `IS_STREAMLIT_CLOUD`
- `STREAMLIT_RUNTIME`
- Presence of `/.streamlit/config.toml`

## Development Commands

### Running Locally

```bash
# Install dependencies
pip install -r requirements.txt

# Set up secrets (create .streamlit/secrets.toml)
mkdir -p .streamlit
# Add your API keys to .streamlit/secrets.toml

# Run the application
streamlit run app.py
```

### Testing Document Processing

The OCR system supports:
- **PDF**: `.pdf` files (validated via `%PDF` signature)
- **Images**: `.jpg`, `.jpeg`, `.png` (validated via PIL)
- **Text**: `.txt` files (with multiple encoding fallbacks: utf-8, latin-1, cp1252, iso-8859-1)

Debug files are saved to: `{tempfile.gettempdir()}/disciplinajudicial_debug/debug_{job_id}_{filename}`

## Key Implementation Patterns

### Multi-Strategy Fallback Pattern

Critical operations use cascading fallback strategies:

**Example: PDF Export** (lines 329-356)
1. Try FPDF with full markdown processing
2. Fall back to ReportLab with paragraph styles
3. Fall back to simple FPDF with minimal formatting
4. Ultimate fallback: Return markdown file

**Example: App Restart** (lines 251-278)
1. Try `st.rerun()`
2. Show manual reload message
3. Inject JavaScript window.location.reload()

### Error Handling Pattern

```python
@handle_error(max_retries=2)
def critical_function():
    # Function implementation
    # Automatically retries up to 2 times with 1s delay
```

### Document Context Injection

When sending messages to OpenAI (lines 1461-1667):
1. Combine `current_doc_contents` with `st.session_state.document_contents`
2. Extract text from OCR results (limit to 5000 chars per document)
3. Build context string with document names and content
4. Prepend to user prompt before sending to API

## Important Constraints

### File Format Validation

- **Strict validation** via `validate_file_format()` checks both extension AND content signature
- Invalid files are rejected with clear error messages
- Supported formats defined in `ALLOWED_FILE_FORMATS` dict (line 722)

### Context Size Management

- Document content limited to 5000 characters per file in prompts (line 1497)
- OCR responses saved with full content to `document_contents` for reference
- Thread timeout: 120 seconds (line 1579)

### Logging Behavior

- Daily log rotation: `logs/disciplinajudicial_YYYYMMDD.log`
- Dual output: stdout + file
- Debug artifacts saved to temp directory for OCR troubleshooting

## Modifying the Application

### Changing Application Identity

Edit the `APP_IDENTITY` dictionary (lines 27-155) to customize:
- Application name, icon, tagline
- Developer information and social links
- UI messages (welcome, errors, placeholders)
- Footer content and documentation links

### Adding New Document Formats

1. Add to `ALLOWED_FILE_FORMATS` (line 722)
2. Update `validate_file_format()` with signature validation logic
3. Add processing branch in `process_document_with_mistral_ocr()` (lines 1066-1149)

### Customizing OCR Processing

- Image optimization: `prepare_image_for_ocr()` (lines 858-908)
- Response parsing: `extract_text_from_ocr_response()` (lines 912-971)
- Mistral API endpoint: Line 1184 (`https://api.mistral.ai/v1/ocr`)

## Judicial Disciplinary Domain Context

This application is specifically designed for **disciplinary processes in the Colombian Judicial Branch**. Key domain concepts:

- **Código General Disciplinario**: Ley 1952 de 2019 modified by Ley 2094 de 2021
- **Comisión Nacional de Disciplina Judicial (CNDJ)**: Second instance for disciplinary cases against judicial officials
- **Comisiones Seccionales de Disciplina Judicial**: First instance disciplinary authority
- **Funcionarios judiciales**: Magistrates, Judges, Prosecutors (art. 125, Ley 270/1996)
- **Empleados judiciales**: Other judicial branch staff, disciplined by their hierarchical superior (art. 115, Ley 270/1996)
- **Faltas disciplinarias**: Classified as gravísimas, graves, leves
- **Proceso disciplinario**: Indagación previa, investigación, pliego de cargos, descargos, alegatos, fallo

The assistant's knowledge base includes normative files stored in `/normas/`:
- Ley 1952 de 2019 (Código General Disciplinario)
- Ley 2094 de 2021 (Reform to CGD)
- Ley 270 de 1996 (Ley Estatutaria)
- Acto Legislativo 02 de 2015
- Ley 2430 de 2024 (Reform to Ley Estatutaria)
- Acuerdo PCSJA20-11593 (MECI)
- Decreto 815 de 2018
- Guía Disciplinaria PGN

### Sidebar Links

The sidebar provides links to official normative sources and institutional websites (Comisión Nacional de Disciplina Judicial, Consejo Superior de la Judicatura, Procuraduría General de la Nación).

## Deployment Considerations

- **Streamlit Cloud**: Automatically detected; uses secrets management
- **Local Development**: Requires `.streamlit/secrets.toml`
- **Analytics**: Optional Umami script injection via secrets
- **File Upload**: Uses Streamlit's native chat input file attachment (line 1975)
- **Session Cleanup**: Manual via sidebar button (lines 1827-1840)

## Testing Notes

- No automated tests currently implemented
- Manual testing should verify:
  - OCR processing for each file type
  - PDF export fallback chain
  - Thread initialization and message streaming
  - Document context injection into prompts
  - Session cleanup and state reset
