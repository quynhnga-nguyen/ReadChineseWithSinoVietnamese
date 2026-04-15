# Read Chinese With Sino-Vietnamese - Product Requirements Contract (v0.4)

## 1) Purpose

This document defines the product contracts for an app that helps Vietnamese learners read Chinese texts with instant lexical support (Chinese definition, pinyin, examples, Han Viet, and Vietnamese definition) and vocabulary collection.

This contract is intentionally implementation-agnostic and is the baseline agreement.

## 2) Product Goal

The app MUST allow a user to:

1. Import and read a Chinese document.
2. Double-click a word/phrase to show a `Look up` button.
3. Click `Look up` to highlight the word/phrase and view linguistic and dictionary details.
4. Save that word/phrase to a personal "New Word List" with the same details.

## 3) Personas and Primary Use Case

- **Primary user**: Vietnamese speaker learning Chinese by reading native text.
- **Core workflow**:
  1. User logs in/authenticates.
  2. User imports a new Chinese document or opens an existing document.
  3. User reads and double-clicks unknown words/phrases.
  4. App shows a `Look up` button.
  5. User clicks `Look up` and reviews lexical details. App highlights the selected text in yellow.
  6. User adds useful items to New Word List for later study.

## 4) Scope

### In Scope (MVP Contract)

- Chinese document import and display.
- Authentication and user session management.
- Double-click selection on words/phrases.
- `Look up` button flow for explicit lookup execution and highlighting.
- Display of four required content blocks:
  - Chinese definition (Chinese-Chinese dictionary)
  - Pinyin
  - Han Viet
  - Vietnamese definition
- Add selected item to New Word List.
- New Word List stores and displays the same four content blocks + original word/phrase.

### Out of Scope (for this contract version)

- Lookup frequency tracking.
- Audio pronunciation.
- Anki integration
- Collaborative features and cloud sync.
- Spaced repetition scheduling.
- Direct Kindle/AZW import support.

## 5) Functional Contracts

## 5.1 Document Import Contract

### User Story

As a user, I want to import Chinese reading material so I can read and interact with it in the app.

### Required Behavior

- The app MUST support importing `EPUB`, `TXT`, and `HTML` in v1.0.
- Kindle/AZW direct import is explicitly deferred after v1.0.

### Preferred Format Strategy (contractual priority)

1. **Preferred input**: EPUB (common conversion target).
2. **Fallback/secondary**: TXT and/or HTML (easy conversion and rendering).

### Non-Functional Constraints

- Import success for valid files MUST complete with user feedback (success or actionable error message).
- Chinese text MUST render correctly in Unicode.

## 5.2 Reading and Text Interaction Contract

### User Story

As a reader, I want to double-click a word/phrase, then click `Look up` to get lexical details.

### Required Behavior

- Double-clicking text MUST create a selection.
- After selection, the app MUST show a `Look up` button near the selection or in the lookup panel.
- Lookup MUST execute when user clicks `Look up`. On that same click, the app MUST apply the yellow highlight to the selected text.
- **Automatic target selection strategy:** depends on chosen dictionary source in design phase; prefer a source/provider that already supports tokenization for robust default candidate selection.

### Response Contract on Selection

The result panel MUST provide:

1. Chinese definition (from Chinese-Chinese source).
2. Pinyin.
3. Han Viet.
4. Vietnamese definition.

### UX Timing Contract

- `Look up` button visibility SHOULD appear immediately after double-click. Yellow highlight SHOULD appear immediately when the user clicks `Look up` (same moment lookup is triggered).
- Lookup response SHOULD feel immediate for local cache hits.
- For non-cached lookups, app MUST show loading state and eventual success/failure state.

## 5.3 New Word List Contract

### User Story

As a learner, I want to save words/phrases for review.

### Required Behavior

- User MUST be able to add current selected word/phrase to New Word List.
- Each saved item MUST include:
  - `text` (Chinese word/phrase)
  - `chineseDefinition`
  - `pinyin`
  - `hanViet`
  - `vietnameseDefinition`

### Duplicate Handling Contract

- Duplicate interactions in reading flow (double-click, highlight, `Look up`) MUST remain fully allowed.
- Duplicate blocking MUST apply only to add-to-list behavior.
- System MUST prevent exact duplicate New Word List items by `text` + normalized reading.
- When user attempts to add an existing item, app MUST keep the original entry and show a clear "already in list" notification.

### Editing/Deletion

- At minimum, user MUST be able to remove an item from New Word List.
- Editing entries is optional in v0.1 but recommended for v0.2.

## 6) Data Contracts

## 6.1 Lookup Result Entity

```json
{
  "text": "中文词/短语",
  "chineseDefinition": "中文释义",
  "pinyin": "Zhōng wén ...",
  "hanViet": "Hán Việt",
  "vietnameseDefinition": "Nghĩa tiếng Việt"
}
```

## 6.2 New Word List Item Entity

```json
{
  "id": "uuid-or-stable-hash",
  "text": "中文词/短语",
  "chineseDefinition": "中文释义",
  "pinyin": "Pīn yīn",
  "hanViet": "Hán Việt",
  "vietnameseDefinition": "Nghĩa tiếng Việt",
  "createdAt": "ISO-8601 timestamp"
}
```

## 7) Source and Quality Contracts for Lexical Data

- Chinese definition MUST come from a Chinese-Chinese dictionary source.
- Pinyin MUST be displayed natively with tone marks (for example, `Zhōng wén`).
- Han Viet mapping MUST be deterministic for the selected text unit.
- Vietnamese definition MUST be displayed natively with full Vietnamese diacritics.
- Vietnamese definition MUST be present if available from source stack.
- If any field is unavailable, UI MUST explicitly show "Not available" rather than empty silent failure.

## 8) Error Handling Contracts

- Import errors MUST provide user-facing reason (unsupported format, parse failure, empty file).
- Lookup errors MUST not crash reading view.
- Partial lookup results are allowed, but missing fields MUST be labeled clearly.
- Add-to-list failures MUST provide retry path.
- Authentication failures/session expiry MUST show clear action (log in again).

## 8.1 Authentication and Password Handling Contract

- v1.0 authentication MUST use username and password.
- Passwords MUST be handled with secure hashing (never stored or logged in plaintext).
- Password verification MUST use constant-time comparison where applicable.
- The app MUST enforce secure transport for credential exchange when networked.
- Error responses for failed login SHOULD avoid leaking whether username or password was incorrect.

## 9) Acceptance Criteria (MVP)

The MVP is accepted only if all criteria below pass:

1. User can log in/authenticate and access their reading workspace.
2. User can import a supported Chinese document or open an existing one.
3. User reads imported/opened content in-app.
4. User double-clicks at least 100 random words/phrases; each time, `Look up` is shown.
5. User clicks `Look up` and receives a result panel each time (success or explicit no-result state). The selected text is highlighted yellow.
6. For successful lookups, panel displays the four required fields, with pinyin tone marks and Vietnamese diacritics.
7. User adds an item to New Word List and sees it persisted in the list view.
8. New Word List item contains the same data schema as lookup display.
9. Duplicate behavior follows agreed policy consistently.

