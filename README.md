# AI Book Translation Workflow

An n8n workflow that translates a long document section by section with Gemini, using the full document as reference for every section, and returns the result as a Google Doc.

## The Problem

People who translate books and long chapters run into the same limits.

- Pasting a whole chapter into a chat model fails or degrades: the model drops detail or changes style partway through.
- Pasting one section at a time loses context. Names, terms, and tone drift between sections.
- Scanned or PDF source files must be turned into text first.
- Tables and normal paragraphs get mixed up when a model rewrites them.
- Manual copy and paste between tools is slow and easy to get wrong on long documents.

## The Solution

1. Someone submits a Google Form: a translation prompt, a link to the source file in Drive, a chapter name, and an optional test limit.
2. The new form row triggers the workflow (checked every minute).
3. The source file is uploaded to a Gemini File Search store, so every later model call can look up the whole document.
4. The same file is sent to Mistral OCR and returned as Markdown.
5. The Markdown is split into sections at the word `SLOT`.
6. For each section, Gemini translates it using your prompt, with the File Search store attached as context.
7. Each translated section is converted to HTML and stored in a temporary n8n data table.
8. All sections are joined into one Google Doc, and a Telegram message sends the link.
9. Temporary data is deleted: the data table, the file in the store, and the store itself.

## How It Helps

- A long chapter is translated in one run, with no copy and paste.
- Every section is translated with the whole document available as reference, not only the section in front of the model.
- Scanned files work, because OCR runs first.
- The "test" field in the form limits the run to the first N sections, so a prompt can be checked cheaply before a full run.
- Telegram messages show when a run starts, each section finishes, and the document is ready.
- A row is added to an execution history sheet for each run.

## What Makes It Different

| Feature | How it works here |
|---|---|
| Whole-document context | Gemini File Search store attached to every section call |
| Input handling | Mistral OCR turns the source file into Markdown |
| Layout rules | Prompt tells the model to keep tables as tables and paragraphs as paragraphs |
| Cost control | Optional section limit from the form |
| Output | Markdown converted to HTML (Gemini 2.5 Flash) and written to a Google Doc |
| Cleanup | Store, uploaded file, and data table are removed after each run |

## How It Works

```
Google Form --> Google Sheet row --> Sheets Trigger
                                        |
                  +---------------------+----------------------+
                  |                                            |
         Gemini File Search store                    Mistral OCR -> Markdown
         (source file uploaded)                                  |
                  |                                  split on word "SLOT"
                  |                                              |
                  |                             loop over each section
                  +------------> Gemini translate (store attached)
                                        |
                           Markdown -> HTML -> data table row
                                        |
                          join rows -> Google Doc -> Telegram link
                                        |
                       log run, delete store/file/data table
```

## Tech Stack

| Part | Tool |
|---|---|
| Orchestration | n8n (Data Tables, Code nodes) |
| Translation | Google Gemini with File Search |
| OCR | Mistral |
| Input and output | Google Forms, Sheets, Drive, Docs |
| Notifications | Telegram |

## Run It Locally

1. Import `workflow.json` into n8n.
2. Create credentials: Google Sheets, Drive, Docs, Gemini, Mistral, Telegram.
3. Create a Google Form with these fields: `Prompt`, `Chapter File` (a Drive link), `chapter name`, `test `. Link it to a Sheet that has a second tab named `execution history`.
4. Set the Sheets Trigger and the Sheets nodes to your sheet. Replace `GOOGLE_FILE_ID`.
5. Replace `TELEGRAM_CHAT_ID` in the five Telegram nodes.
6. In the "delete file from file search store1" node, replace `GEMINI_API_KEY` with a key stored as an n8n credential or environment variable.
7. Activate the workflow and submit the form.

## Warnings

- The source text is sent to Gemini and Mistral. Do not use documents you are not allowed to share with them.
- Both are paid APIs, and cost grows with document length. Use the test limit first.
- The split uses the word `SLOT`, case-insensitive, anywhere in the text. Your source must contain `SLOT` markers between sections, and the word must not appear inside the text.
- The Gemini key in the delete node was originally inline in the workflow. Never keep keys inside workflow JSON.

## Status

Done: form trigger, OCR, section loop, translation with File Search, HTML conversion, Google Doc output, Telegram updates, cleanup.

Next: the prompt asks the model to respect context, but the workflow has no automated check that terms stay consistent, and no human review step. Those are not built.
