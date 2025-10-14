# Track-Deleted-Telegram-Messages
This python script is used to track deleted telegram messages in a group chat in real time. Host in your RDP for long term use
# pip install telethon
import os
from datetime import datetime, timezone
from telethon import TelegramClient, events
from pathlib import Path

# ---------- CONFIG ----------
API_ID = int(os.getenv("API ID", "0"))          # set env: API ID
API_HASH = os.getenv("API HASH", "")            # set env: API HASH
SESSION = os.getenv("TG_SESSION", "my_session")    # optional: session file name
LOG_DIR = Path(os.getenv("TG_LOG_DIR", "logs"))    # where to put the txt file
LOG_DIR.mkdir(parents=True, exist_ok=True)

# Create a NEW txt file each run
start_stamp = datetime.now().strftime("%Y%m%d_%H%M%S")
LOG_PATH = LOG_DIR / f"telegram_deleted_watch_{start_stamp}.txt"

if not API_ID or not API_HASH:
    raise RuntimeError("Set TG_API_ID and TG_API_HASH environment variables.")

client = TelegramClient(SESSION, API_ID, API_HASH)

# Minimal in-memory cache so deletions can include content
# { chat_id: { message_id: (sender_id, sender_name, text, sent_iso, has_media) } }
store = {}

# Open the log file once, line-buffered; flush on every write for "real-time"
_log_fp = open(LOG_PATH, "a", encoding="utf-8", buffering=1)

def utc_iso(dt) -> str:
    if dt is None:
        return datetime.now(timezone.utc).isoformat(timespec="seconds").replace("+00:00", "Z")
    if dt.tzinfo is None:
        dt = dt.replace(tzinfo=timezone.utc)
    else:
        dt = dt.astimezone(timezone.utc)
    return dt.isoformat(timespec="seconds").replace("+00:00", "Z")

def log_line(s: str):
    """Append one line to the txt file immediately."""
    _log_fp.write(s.rstrip("\n") + "\n")
    _log_fp.flush()  # ensure written to disk in real time

async def get_sender_friendly(event):
    try:
        s = await event.get_sender()
    except Exception:
        s = None
    if not s:
        return None, "Unknown"
    sid = getattr(s, "id", None)
    uname = getattr(s, "username", None)
    if uname:
        return sid, uname
    fn = getattr(s, "first_name", "") or ""
    ln = getattr(s, "last_name", "") or ""
    name = (fn + " " + ln).strip() or f"id:{sid}"
    return sid, name

@client.on(events.NewMessage())
async def on_new_message(event: events.NewMessage.Event):
    # Optional: skip our own outgoing messages to reduce noise
    if event.out:
        return

    msg = event.message
    chat_id = event.chat_id
    if chat_id is None:
        return

    sender_id, sender_name = await get_sender_friendly(event)
    text = msg.message or ""
    has_media = 1 if msg.media else 0
    sent_iso = utc_iso(msg.date)

    # Cache for deletion lookup
    store.setdefault(chat_id, {})[msg.id] = (sender_id, sender_name, text, sent_iso, has_media)

    # Real-time human-readable log entry
    content = text if text else ("<media / no text>" if has_media else "<no text>")
    log_line(
        f"[NEW] {sent_iso} | chat={chat_id} | msg_id={msg.id} | "
        f"from={sender_name}({sender_id}) | {content}"
    )

@client.on(events.MessageEdited())
async def on_message_edited(event: events.MessageEdited.Event):
    msg = event.message
    chat_id = event.chat_id
    if chat_id is None:
        return

    text = msg.message or ""
    has_media = 1 if msg.media else 0
    edit_iso = utc_iso(datetime.now(timezone.utc))

    # Update cache (if present)
    rec = store.get(chat_id, {}).get(msg.id)
    if rec:
        sender_id, sender_name, _, sent_iso, _ = rec
        store[chat_id][msg.id] = (sender_id, sender_name, text, sent_iso, has_media)
    else:
        # We might have missed the original; try to reconstruct minimal info
        sender_id, sender_name = await get_sender_friendly(event)
        sent_iso = utc_iso(msg.date)
        store.setdefault(chat_id, {})[msg.id] = (sender_id, sender_name, text, sent_iso, has_media)

    content = text if text else ("<media / no text>" if has_media else "<no text>")
    log_line(
        f"[EDIT] {edit_iso} | chat={chat_id} | msg_id={msg.id} | "
        f"content_now={content}"
    )

@client.on(events.MessageDeleted())
async def on_deleted(event: events.MessageDeleted.Event):
    deleted_iso = utc_iso(datetime.now(timezone.utc))
    chat_id = event.chat_id  # can be None sometimes

    for mid in event.deleted_ids:
        if chat_id is not None:
            rec = store.get(chat_id, {}).pop(mid, None)
        else:
            # fallback: scan all chats for this msg id
            rec = None
            for c_id, msgs in list(store.items()):
                if mid in msgs:
                    chat_id = c_id
                    rec = msgs.pop(mid, None)
                    break

        if rec:
            sender_id, sender_name, text, sent_iso, has_media = rec
            content = text if text else ("<media / no text captured>" if has_media else "<no text>")
            log_line(
                "[DELETE] "
                f"{deleted_iso} | chat={chat_id} | msg_id={mid} | "
                f"sender={sender_name}({sender_id}) | sent={sent_iso} | content={content}"
            )
        else:
            log_line(
                f"[DELETE] {deleted_iso} | chat={chat_id if chat_id is not None else '<unknown>'} "
                f"| msg_id={mid} | content=<no archived copy>"
            )

if __name__ == "__main__":
    print(f"Logging to: {LOG_PATH}")
    client.start()  # first run: interactive login
    try:
        client.run_until_disconnected()
    finally:
        try:
            _log_fp.close()
        except Exception:
            pass
