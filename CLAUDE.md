# mcp-google-workspace

A stdio MCP server exposing Gmail and Google Calendar tools (`src/tools/gmail.ts`, `src/tools/calendar.ts`), wired through a shared OAuth2 client (`src/services/gauth.ts`) and dispatched from `src/server.ts`. Build output goes to `dist/` (gitignored, rebuilt with `npm run build`).

## Tool schema conventions

- Every tool except `gmail_list_accounts` / `calendar_list_accounts` requires `user_id`, the email of one of the accounts in `.accounts.json`. There is no session-level "current account" — each call routes independently.
- Params are `snake_case` (`message_id`, `event_id`, `add_labels`, ...), matching the rest of this schema, not the camelCase used by the underlying `googleapis` client.
- ID-shaped params must be the literal value returned by a prior tool call, never something derived from it:
  - `attachment_id` (used by `gmail_get_attachment`, `gmail_bulk_save_attachments`) is the `attachmentId` string inside the `attachments` map from `gmail_get_email`/`gmail_bulk_get_emails` — not the map's numeric key (`"1"`, `"2"`, ...), which is just the MIME part index. This was a real bug: the field used to be named `part_id`, which read as if the part-index key were correct input. Renamed 2026-07-14 (commit `be69c10`) specifically to stop that misread.
  - `gmail_bulk_modify`'s `add_labels`/`remove_labels` want label **IDs**. System labels double as their own ID (`INBOX`, `UNREAD`, `STARRED`, `TRASH`, `SPAM`, `IMPORTANT`); user labels don't — fetch the ID via `gmail_list_labels` first.
  - `calendar_update_event`/`calendar_delete_event` want `event_id` from `calendar_get_events`, not the event summary.
- When adding a new tool with an ID-shaped param, name it to match what it actually is (`attachment_id`, not `part_id`/`token`/`ref`), and say in the description exactly which prior tool's output it comes from. That's the class of bug this file exists to prevent.

## Batch/bulk semantics

- `gmail_bulk_get_emails` and `gmail_bulk_save_attachments` are client-side fan-out: one Gmail API call per item, no server batch endpoint involved.
- `gmail_bulk_archive` also does one `messages.modify` call per ID.
- `gmail_bulk_modify` is the actual batched write path — `users.messages.batchModify`, up to 1000 IDs per request, auto-chunked (`CHUNK_SIZE = 1000` in `gmail.ts`) for longer lists. Prefer this over `gmail_bulk_archive` for large batches.
- Archiving == removing the `INBOX` label. It never adds `TRASH`. Don't conflate "archive" with "delete" in tool descriptions or behavior.

## Safety guardrail: sending is opt-in

`gmail_reply` and `gmail_create_draft` are stripped from the tool list in `getTools()` unless `process.env.GMAIL_ALLOW_SENDING === 'true'` (see the `.filter(...)` at the end of `GmailTools.getTools()`). This is deliberate — see commit `4e116d3` ("Never allow AI to send draft emails"). Don't remove or weaken this filter without the repo owner explicitly asking for it; it's the only thing standing between an assistant and sending mail as the user.

## Workflow: change → build → reconnect

1. Edit the tool schema/handler in `src/tools/*.ts`.
2. `npm run build` (`tsc`, compiles to `dist/`).
3. Restart or reconnect the MCP client session (in Claude Code: `/mcp`, reconnect this server). Clients cache the tool schema from initial connection — a rebuild alone does not make a running client see the new schema.

## Auth model

Each account in `.accounts.json` authenticates independently via OAuth2; first use opens a browser flow on `http://localhost:4100/code`, then caches to `.oauth2.{email}.json` under `--credentials-dir`. `.gauth.json`, `.accounts.json`, and `.oauth2.*.json` are all gitignored (`/.*.json` in `.gitignore`) — never commit them.
