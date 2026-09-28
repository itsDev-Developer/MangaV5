# Session 3 — poster quality, file-store bot, auto-publish checks, backup, help, production pass

## 1. High-quality posters — fixed

Root cause: search/listing pages serve small thumbnails. The manga's own
detail page usually has a much better image, but the site modules never
looked for it — two of them (`manga18fx`, `manhwa18`) even had the upgrade
code already, just gated behind `if "poster" not in results`, which never
actually ran since a poster was always already set by the search step.

**Fix:** added `extract_og_image()` (`Webs/utitls.py`) — pulls the
`og:image` / `twitter:image` meta tag from the manga's detail page, which
virtually every site sets to a full-resolution "share image" regardless of
what their listing page shows. Wired into all 9 HTML-scraped site modules
(asurascans, manga18fx, manhwa18, manhuafast, manhuaplus, manhwaclan, mgeko,
templetoons, weebcentral). Comick was already fine — its API returns a
proper CDN cover URL directly — left untouched.

This applies to **every** flow (manual `/newpost`, "📤 Post to Channel",
🔁 Auto-Publish) since they all go through the same `get_chapters()` call.

## 2. File Store Bot integration — how it works

**The constraint:** a Telegram bot's file IDs only work for *that specific
bot* — there's no API to pull a file from another bot's storage and re-send
it as your own. So this integration works the way virtually every public
"file store bot" is designed to be used: as a **direct hand-off**. Your
channel post's "Read Now" button points straight at the *other* bot's own
share link, and that bot delivers the file itself. Your bot never touches
the file.

**Setup:**
1. In `/settings` → 🗄 File Store Bot, turn on **"Use File Store Bot"** and
   set **"File Store Bot username"** (no `@`).
2. When creating a post (`/newpost` or "📤 Post to Channel"), after files are
   gathered you'll now be asked:
   > Upload the same file(s) to @YourFileStoreBot and paste the share link
   > it gives back...
   Upload the file(s) to that bot yourself (outside this bot, using
   whatever upload flow it has), get its share link back, and paste it in.
   Send `/skip` to fall back to this bot's own Dump Channel delivery for
   that post.
3. The post preview shows "🗄 Delivers via the File Store Bot" when a link
   is attached, and the published "Read Now" button uses that link directly.

**Limitations (by design, not bugs):**
- It's per-post, not automatic — there's no way to auto-generate the other
  bot's share link without an API integration with that specific bot's
  codebase, which varies bot-to-bot.
- 🔁 Auto-Publish posts always use this bot's own delivery (nobody's around
  to paste a link into an unattended background job). Only the manual
  wizard and the interactive "Post to Channel" flow ask for it.
- If your file-store bot exposes an HTTP API to fetch a share link
  programmatically, I can wire that in directly instead of the manual paste
  step — tell me which one you're using and I'll take a look.

## 3. Auto-Publish now checks if the manga is already posted

Tapping "🔁 Auto-Publish" now checks the Post Channel for an existing post
with that title (same check used by the duplicate-post guard):
- **Already posted** → sends a message *in the Post Channel*:
  "🔁 Auto-Publish activated for **Title** — new chapters will be posted
  here automatically."
- **Not posted yet** → no channel message; instead every admin gets a DM
  warning that it's being tracked but has no intro post yet, with a nudge
  to use /newpost or "📤 Post to Channel" first.

## 4. Help command overhaul + daily auto-backup

- `/help` is now a categorized menu: a short usage guide plus **👤 User
  Commands** and (for admins) **🛠 Admin Commands** buttons, listing every
  command in the bot — including ones that had no documentation before
  (`/scheduled`, `/autopublish`, `/backup`, `/restore`, `/settings`, and the
  full broadcast/premium/system command set). The admin-only Telegram
  command menu (the `/` autocomplete list) was also updated to include the
  newer commands.
- **Auto-backup**: `/settings` → 💾 Auto-Backup has an on/off switch and an
  "every (days)" interval. When enabled, a full database backup is sent to
  the **Log Channel** automatically on that schedule (checked hourly,
  persisted across restarts via a DB timestamp — no double-sends after a
  restart).

## 5. Production-readiness / optimization pass

- **Dockerfile**: switched `python:3.12` → `python:3.12-slim` (saves ~300MB
  of unused build tools/docs in the image), and the container now runs as
  an unprivileged user instead of root — standard hardening, costs nothing
  functionally since the app only touches its own working directory.
- Added `.dockerignore` (was missing entirely) so `.git`, `__pycache__`,
  local venvs etc. never end up in the build context or image.
- Reduced the scheduled-posts background loop's poll interval from 30s to
  60s — still fine granularity for "in 2h"-style scheduling, halves the
  idle DB reads over a long-running deployment.
- **Dependency pinning**: `requirements.txt` still has no version pins
  (`pyrofork`, `pymongo`, etc.). I didn't guess at exact versions here since
  pinning the *wrong* one could break something I can't test — but for a
  genuinely production-locked deployment, I'd recommend running
  `pip freeze > requirements.txt` on your current working deployment once
  it's confirmed stable, so a future rebuild can't silently pull a newer,
  possibly-breaking version of a dependency.

## Note on testing

Same standing caveat: no live bot/network here, so I can't run this
end-to-end myself — I traced every change by hand and the whole project
compiles cleanly. Please test in staging first, especially:
- The `og:image` poster upgrade (site markup can change without notice).
- The file-store link flow end-to-end with your actual file-store bot.
