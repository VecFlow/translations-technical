# Translation API Implementation Specification

## Overview

The Translation API provides document and text translation capabilities using two translation engines:
1. **DeepL API** - For supported language pairs with direct document translation support
2. **GPT-5** - For unsupported language pairs or when DeepL fails, with streaming capabilities

## How Translations Work

### Translation Flow
1. Client initiates translation request with either a document ID or raw text
2. System creates a translation record with status "created"
3. Background processing begins:
   - For documents: Auto-detects source language if not provided
   - Determines translation engine (DeepL vs GPT-5) based on language support
   - Processes translation (DeepL: bulk upload, GPT-5: streaming)
   - Updates translation record with results
4. Client can poll status or subscribe to real-time updates via Supabase

### Translation Types
- **Document Translation**: Full document translation with preserved formatting (when using DeepL)
- **Text Translation**: Plain text translation (always returns text)

## API Endpoints

### 1. Create Translation
**Endpoint:** `POST /translate`

**Purpose:** Initiates a new translation job

**Request Body:**
```json
{
  "target_language": "ES",        // Required: Target language code (e.g., "ES", "FR", "JA")
  "source_language": "EN",        // Optional: Source language code (auto-detected if omitted)
  "doc_id": "abc-123-def",        // Optional: Document ID (mutually exclusive with text)
  "text": "Hello world",          // Optional: Raw text (mutually exclusive with doc_id)
  "user_email": "user@example.com" // Required: User's email address
}
```

**Response (200 OK):**
```json
{
  "status": "started",
  "translation": {
    "id": "trans-uuid-123",
    "doc_id": "abc-123-def",
    "target_language": "ES",
    "source_language": "EN",
    "text": "",
    "translated_text": null,
    "translated_document_metadata": null,
    "status": "created",
    "type": "document",
    "user_email": "user@example.com",
    "created_at": "2025-01-21T10:00:00Z",
    "updated_at": "2025-01-21T10:00:00Z"
  }
}
```

**Error Responses:**
- `400 Bad Request`: Invalid request (e.g., both doc_id and text provided)
- `500 Internal Server Error`: Failed to create translation

### 2. Get Supported Languages
**Endpoint:** `GET /translate/languages`

**Purpose:** Retrieves list of supported languages for translation

**Query Parameters:**
- `type` (optional): "source" or "target" (defaults to "target")

**Response (200 OK):**
```json
{
  "deepl_available": true,
  "languages": [
    {
      "language": "EN-US",
      "name": "English (American)"
    },
    {
      "language": "ES",
      "name": "Spanish"
    },
    {
      "language": "FR",
      "name": "French"
    },
    {
      "language": "JA",
      "name": "Japanese"
    }
  ],
  "type": "target",
  "note": "Languages supported by DeepL for target language selection."
}
```

### 3. Download Translated Document
**Endpoint:** `GET /translate/download/{translation_id}`

**Purpose:** Gets a presigned URL to download the translated document

**Path Parameters:**
- `translation_id`: The UUID of the translation

**Response (200 OK):**
```json
{
  "download_url": "https://s3.amazonaws.com/bucket/...",  // Presigned S3 URL (valid for 1 hour)
  "filename": "ES_document.pdf",
  "content_type": "application/pdf",
  "expires_in": 3600,
  "translation_id": "trans-uuid-123",
  "source": "deepl_document"  // or "legacy_document" for older translations
}
```

**Error Responses:**
- `404 Not Found`: Translation not found
- `400 Bad Request`: Translation not completed or is text-only (no document available)

## Real-time Updates via Supabase

The frontend can subscribe to real-time updates for translation progress using Supabase's real-time functionality. Translations can be monitored by subscribing to the 'translations' table updates filtered by the translation ID. The status field will update to "processing", "completed", or "failed". For GPT-5 translations, the translated_text field updates incrementally during streaming.

## Translation Status Values

- `created`: Initial state when translation is created
- `processing`: Translation is being processed
- `completed`: Translation finished successfully
- `failed`: Translation failed (check error details)

## Language Support

### DeepL Supported Languages

**Target Languages (can translate TO):**
```
AR (Arabic), BG (Bulgarian), CS (Czech), DA (Danish), DE (German),
EL (Greek), EN-GB (British English), EN-US (American English),
ES (Spanish), ET (Estonian), FI (Finnish), FR (French), HU (Hungarian),
ID (Indonesian), IT (Italian), JA (Japanese), KO (Korean), LT (Lithuanian),
LV (Latvian), NB (Norwegian), NL (Dutch), PL (Polish), PT-BR (Brazilian Portuguese),
PT-PT (European Portuguese), RO (Romanian), RU (Russian), SK (Slovak),
SL (Slovenian), SV (Swedish), TR (Turkish), UK (Ukrainian),
ZH/ZH-HANS (Simplified Chinese), ZH-HANT (Traditional Chinese)
```

**Source Languages (can translate FROM):**
```
AR, BG, CS, DA, DE, EL, EN, ES, ET, FI, FR, HU, ID, IT, JA, KO,
LT, LV, NB, NL, PL, PT, RO, RU, SK, SL, SV, TR, UK, ZH
```

### GPT-5 Fallback
For language pairs not supported by DeepL, the system automatically falls back to GPT-5 translation, which supports most world languages.

## Example Workflows

### 1. Translate a Document

**Step 1: Create Translation**
- Send a POST request to `/translate` with doc_id, target_language, and user_email
- Optionally include source_language (will auto-detect if not provided)
- Store the returned translation ID for tracking

**Step 2: Monitor Progress**
- Subscribe to Supabase real-time updates using the translation ID
- Monitor the status field for "processing", "completed", or "failed"
- For document translations, the process may take several minutes

**Step 3: Download Translated Document**
- Once status is "completed", call GET `/translate/download/{translation_id}`
- Use the returned presigned URL to download the translated document
- Note: Download URLs expire after 1 hour

### 2. Translate Text with Streaming Updates

**Step 1: Create Translation**
- Send a POST request to `/translate` with text, target_language, and user_email
- Store the returned translation ID

**Step 2: Subscribe to Streaming Updates**
- For GPT-5 translations, subscribe to Supabase real-time updates
- The translated_text field will update incrementally as translation progresses
- Update your UI progressively as new text arrives
- Monitor status field for final "completed" state

### 3. Check Available Languages

**Implementation Requirements:**
- Call GET `/translate/languages?type=target` for target language options
- Call GET `/translate/languages?type=source` for source language options
- Populate language selection dropdowns with the returned language arrays
- Each language object contains 'language' (code) and 'name' (display name)

## Important Notes

1. **Document Format Support:** DeepL supports: docx, doc, pptx, xlsx, pdf, htm, html, txt, xlf, xliff, srt
2. **Text Length Limits:** GPT-5 translations are limited to 100,000 tokens (approximately 75,000 words)
3. **Auto-detection:** If no source language is provided for documents, the system will auto-detect using AI
4. **Presigned URLs:** Download URLs expire after 1 hour for security
5. **Real-time Updates:** For GPT-5 translations, the `translated_text` field updates incrementally during streaming
6. **Language Codes:** Use uppercase ISO language codes (e.g., "EN", "ES", "FR")
7. **English Variants:** For English targets, specify variant: "EN-US" or "EN-GB"
8. **Portuguese Variants:** For Portuguese targets, specify: "PT-BR" or "PT-PT"
9. **Chinese Variants:** For Chinese, use "ZH-HANS" (Simplified) or "ZH-HANT" (Traditional)

## Error Handling

Common error scenarios:
- **Invalid Language Code:** Check against supported languages endpoint
- **Document Not Found:** Verify doc_id exists
- **Translation Timeout:** DeepL translations timeout after 15 minutes
- **Unsupported Format:** Only specific document formats supported for DeepL
- **Text Too Long:** GPT-5 has token limits

## Database Schema

The translations table includes:
- `id` (UUID): Primary key
- `doc_id` (string): Reference to document (null for text translations)
- `text` (text): Original text input
- `target_language` (string): Target language code
- `source_language` (string): Source language code (nullable)
- `translated_text` (text): Translation result (for text/GPT-5)
- `translated_document_metadata` (JSONB): Document metadata (for DeepL documents)
- `status` (string): Current status
- `type` (string): "document" or "text"
- `user_email` (string): Requesting user
- `created_at` (timestamp): Creation time
- `updated_at` (timestamp): Last update time
