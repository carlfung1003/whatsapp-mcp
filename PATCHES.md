# Local patches on top of `lharries/whatsapp-mcp`

This fork applies four patches on top of [PR #245](https://github.com/lharries/whatsapp-mcp/pull/245) (the whatsmeow + `context.Context` API update). Built and tested against `go.mau.fi/whatsmeow v0.0.0-20260511155711-eb05d94dea7d`.

All four are upstreamable — they're self-contained, don't change existing behavior for callers, and address concrete gaps. They were built for use with [carlfung1003/whatsapp-viewer](https://github.com/carlfung1003/whatsapp-viewer) but help anyone using the bridge.

## 1. Capture reaction events into a `reactions` table

**Why:** stock `handleMessage` calls `extractTextContent()` which only handles `Conversation` and `ExtendedTextMessage`. Reaction events arrive as `Message.ReactionMessage` and were silently dropped, so the bridge had no way to surface "who reacted with what to which message" — table-stakes for any workflow that uses reactions semantically (e.g., business groups marking items "claimed" with an emoji).

**What changed:**

- New table `reactions(target_id, target_chat_jid, reactor, emoji, timestamp)` with `PRIMARY KEY (target_id, target_chat_jid, reactor)`. Schema migration is idempotent (`CREATE TABLE IF NOT EXISTS`).
- New `MessageStore.StoreReaction(targetID, chatJID, reactor, emoji, timestamp)`. Empty emoji = delete the reactor's reaction (matches WhatsApp's "remove reaction" semantics).
- `handleMessage` checks `msg.Message.GetReactionMessage()` at the top of the function and short-circuits to `StoreReaction` after extracting the target's message ID and the reactor's JID.
- Same branch added in the history-sync loop so reactions captured during initial sync land in the table too.

## 2. Capture quoted-reply context into a `quoted_message_id` column

**Why:** WhatsApp reply messages carry a `ContextInfo` that includes `StanzaId` — the ID of the message being quoted. The stock bridge calls `extendedText.GetText()` and throws away the context. Without the linkage, you can't tell whether "I want it" is replying to image #1 or image #50 in a multi-image drop.

**What changed:**

- New column `messages.quoted_message_id TEXT`. Migration uses a `pragma_table_info()` check + `ALTER TABLE ADD COLUMN` so existing DBs upgrade in place without data loss.
- New helper `extractQuotedMessageID(msg)` that walks the standard message types (ExtendedText, Image, Video, Document, Audio, Sticker) and returns `ContextInfo.GetStanzaID()` for whichever one is set, or `""` if not a reply.
- `StoreMessage` signature gained a trailing `quotedMessageID string` parameter. Both call sites (live messages and history sync) updated to pass it.

## 3. Make media filenames unique per message ID

**Why:** `extractMediaInfo` originally generated filenames from `time.Now().Format("20060102_150405")`. During history sync where thousands of messages arrive within the same second, the second-resolution timestamp collides massively — in one observed case, **1,665 distinct messages** got the filename `image_20260522_182021.jpg`. The bridge writes downloaded media to `store/<chat_jid>/<filename>` so later downloads of *different* messages overwrite or shadow each other on disk. Anything that trusts the filename for a cache check serves the wrong file.

**What changed:**

- `extractMediaInfo` signature now takes a trailing `msgID string`. Image/video/audio filenames are built as `image_<msgID>.jpg` / `video_<msgID>.mp4` / `audio_<msgID>.ogg`. Documents get a `<msgID>_<original-filename>` prefix so user-supplied names also can't collide.
- Both call sites updated. For the history-sync loop the message ID was previously computed *after* `extractMediaInfo` ran — moved up so it's available when needed.
- Run this one-time migration on an existing DB after deploying the patch:

  ```sql
  UPDATE messages SET filename = 'image_' || id || '.jpg' WHERE media_type = 'image';
  UPDATE messages SET filename = 'video_' || id || '.mp4' WHERE media_type = 'video';
  UPDATE messages SET filename = 'audio_' || id || '.ogg' WHERE media_type = 'audio';
  ```

  Then delete the `store/<chat_jid>/` directories — the stale cached files no longer match the DB and will re-download as needed.

## 4. QR-pairing helper: also write the raw code to `/tmp/whatsapp-qr.txt`

**Why:** stock bridge renders the QR with `qrterminal.GenerateHalfBlock` (Unicode half-block characters). Some terminals don't render these cleanly — squished, mis-aspect-ratio, or just blank.

**What changed:**

- One extra line in the QR-loop in `main.go` that also writes `evt.Code` to `/tmp/whatsapp-qr.txt`. From there you can generate a scannable PNG with any QR library:

  ```bash
  uv run --with "qrcode[pil]" python -c "
  import qrcode
  qrcode.make(open('/tmp/whatsapp-qr.txt').read().strip(), box_size=12, border=2).save('/tmp/wa-qr.png')
  " && open /tmp/wa-qr.png
  ```

  Optional, doesn't affect anything else, leaves the terminal QR in place too.

## Upstreaming

The patches are commit-ready against upstream `main`. If you want to PR them separately:

1. Fork from upstream `main`, not from the PR #245 branch this lives on.
2. Cherry-pick the relevant subset of files per patch:
   - Reactions + quoted-message: `whatsapp-bridge/main.go` reaction branch, `extractQuotedMessageID`, `StoreReaction`, schema additions
   - Filename collision: `extractMediaInfo` signature change + the two call sites
   - QR PNG: the single `os.WriteFile` line
3. The Python MCP server changes (`Reaction` dataclass, `quoted_message_id` field, inline reaction display, `list_reactions` tool) are paired with patches 1+2 — submit together so the surface area stays consistent.
