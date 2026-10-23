---
name: "vault-ingest"
description: "Process Scott's Software Engineering Obsidian vault Inbox into Source and Atlas notes per the Vault Charter, or capture a URL, text or screenshot into it. Use for \"process my inbox\", \"ingest this\", \"add this to my vault\"."
---

# vault-ingest

Turns raw captures in `00 Inbox` into finished notes: **what the source said** becomes a Source note in `30 Sources`, **what is now known** becomes a new or extended Atlas note in `20 Atlas`. Everything follows the vault's own templates and prompts, which are the single source of truth. Read them every run; if this file and the vault disagree, the vault wins.

## Modes

| Mode | Trigger | Behavior |
|------|---------|----------|
| **Process Inbox** | "process my inbox", "run vault-ingest", a scheduled run | Every eligible Inbox item, oldest first |
| **Quick capture** | "add this to my vault" + URL, pasted text or an image | Write the capture into `00 Inbox` as a new item, then process just that item |
| **Unattended** | Started by a scheduled task, or nobody is there to answer | Same as Process Inbox, but skip the approval step and write a run log (step 8) |

## Step 0: Locate the vault and load conventions

The vault root is the folder containing both `Vault Charter.md` and `00 Inbox/`. On Scott's PC it is `D:\Google Drive\Scott\ObsidianVaults\Software Engineering`. It lives in Google Drive, so it syncs both ways between the PC and Drive.

- **Cowork with a linked computer:** do all file work with `device_bash`, editing in place. Never stage a text file just to read or edit it. Find the mount with:
  ```bash
  for d in "$HOME"/mnt/*/; do [ -f "$d/Vault Charter.md" ] && [ -d "$d/00 Inbox" ] && echo "$d"; done
  ```
  If nothing is found, ask for folder access to the path above.
- **Claude Code / local shell:** use the path directly.
- **No linked computer** (the remote-devices tools are absent, or they can't reach the PC) **but the Google Drive connector is available:** use the **Google Drive fallback** section at the end of this file. It changes Steps 1–2 and 5–8; say so in the Step 4 plan.

In the snippets below, `$VAULT` is that root; set it at the start of every shell call (`VAULT="…"`), because each call starts fresh. Record the start time with `date +%s`; Step 6 needs it.

Then read these files (they're short):

- `90 Templates/Source.md`, `Concept.md`, `Pattern.md`, `Reference.md`, `Term.md`
- `99 System/Prompts/Process Inbox Item.md`, `Screenshot to Text.md`, `Summarize Video.md`
- `00 Inbox/_Inbox Guide.md`

## Step 1: Inventory the Inbox

Eligible items are the files in `00 Inbox/` except those whose names start with `_` (guides) and `Unlinked Images.md` (a Phase 2 backlog; process it only when Scott asks). Order them oldest first by the `created` property, then by file modification time. If the Inbox is empty, say so and stop.

## Step 2: Build the vault index (for duplicate detection)

```bash
cd "$VAULT" && python3 - <<'EOF'
import os, re
def fm(t):
    return t.split('---', 2)[1] if t.startswith('---') and t.count('---') >= 2 else ''
def field(f, k):
    m = re.search(rf'^{k}:[ \t]*(.*?)(?=\n[^\s-]|\Z)', f, re.S | re.M)
    if not m: return ''
    v = m.group(1)
    items = [i.strip().strip('\'"') for i in re.findall(r'^\s*-\s*(.+)$', v, re.M)]
    return '; '.join(items) if items else v.strip().strip('\'"')
for root in ('20 Atlas', '30 Sources', '40 Toolkit'):
    for r, d, fs in os.walk(root):
        for f in fs:
            if f.endswith('.md'):
                h = fm(open(os.path.join(r, f), encoding='utf-8', errors='ignore').read())
                print(f"{os.path.join(r, f)}\t{field(h, 'type')}\t{field(h, 'aliases')}\t{field(h, 'summary')[:140]}")
EOF
```

Keep the output in mind for the rest of the run. It's how "merge before you create" is enforced.

## Step 3: Read and classify each item

For each item:

1. **Get the content.**
   - Markdown: read it.
   - A URL with little or no body: fetch the page with WebFetch.
   - Embedded or loose images: follow *Screenshot to Text*.
     - Images are in `99 System/Attachments`. To see one, stage just that image into the container and Read it.
     - If staging is refused because the file is hardlinked, `cp` it to `99 System/.ingest-tmp/` (a hidden folder Obsidian ignores), stage the copy, and reuse that folder on later runs.
2. **Set `source_type`:** `article` · `post` · `video` · `course` · `book` · `ai`.
3. **Identify the 1–3 distinct concepts** the item teaches, each with a title that is the term someone would search for ("Circuit Breaker", not "LinkedIn post on resilience").
4. **Check each concept against the index.**
   - Match on title and aliases (case-insensitive, acronyms included), then on summary.
   - When unsure, grep the Atlas bodies for the key terms.
   - Decide **extend** (an existing note covers it) or **create**.
5. **Pick for each note:**
   - `type`: `concept`, `pattern` (with `category`), `reference` or `term`.
   - `domain`: one or more from Architecture, Design Patterns, System Design, DotNet & CSharp, Languages, Data, Cloud & Infrastructure, Security, Observability, Testing, AI & Agents, Engineering Practices, Leadership, Glossary.

## Step 4: Approve the plan (interactive runs only)

Show one table, then wait for Scott's OK or edits:

| Inbox item | Source note → | Atlas: create | Atlas: extend | Flags |
|---|---|---|---|---|

- **Flags** are things like `to_verify`, "image transcribed" or "URL fetched".
- Keep the table short. Scott is an experienced architect and doesn't need the conventions explained back to him.
- In **Unattended** mode, skip this step.

## Step 5: Write the notes

**Formatting rules for every note**

- Frontmatter follows the matching template exactly.
- Dates are `YYYY-MM-DD` (today for `created`/`updated` on new notes; bump `updated` on extended ones).
- Wikilinks inside YAML are quoted: `- '[[Architecture]]'`.
- Tags come only from the controlled list: `interview` `cheatsheet` `anti_pattern` `snippet` `decision` `trade_off` `to_verify` `needs_text` `vault_system`. A topic word is a `domain` or a link, never a tag.
- File names contain none of `# ^ [ ] | \ / :` and don't start with `.`. Spell out ".NET" as `DotNet` and "C#" as `CSharp` in file names; put the original spelling in `aliases`.

**Source note** (the Inbox item becomes this note)

1. Rewrite the file in place using `90 Templates/Source.md`. Set `status: draft`.
2. Fill in the Summary (3–5 sentences in plain words) and the Key takeaways.
3. Put the original clipped text under `## Notes`, condensed if it's long.
4. List every Atlas note it created or extended under `## Distilled into`.
5. Move it with `mv -n` to the folder for its type:

| `source_type` | Folder |
|---|---|
| `course` | `30 Sources/Courses/<course>/` |
| `video` | `30 Sources/Videos/<series>/` (use `30 Sources/Videos/` for one-offs) |
| `article` or `post` | `30 Sources/Articles & Posts/` |
| `book` | `30 Sources/Books/` |
| `ai` | `30 Sources/AI Conversations/` |

Create a folder if it's missing. If a file with the same name exists, add a distinguishing suffix; never overwrite.

**New Atlas note**

1. Start from the matching template and save it to `20 Atlas/<primary domain>/<Title>.md`. Glossary terms go to `20 Atlas/Glossary/`.
2. Write the TL;DR callout and a one-sentence `summary`. Add acronyms and synonyms to `aliases`.
3. Fill `## Related` with at least one existing note, chosen from the index.
4. Fill `## Sources` with a link to the Source note.
5. Include `.NET example` only when you're confident the code is correct and idiomatic for current .NET. Otherwise delete the section. Never leave an empty placeholder.
6. If the item was an image, move the image to `99 System/Attachments/` (if it's not there already) and embed it at the bottom under `## Original` as `![[file.png|600]]`.

**Extending an existing Atlas note**

1. Add only. Don't delete or reword existing text.
2. Put the new material in the section it belongs to (a new bullet, a new table row, or a new `###` subsection).
3. Add the new source to `## Sources`, add any new aliases, and bump `updated`.
4. If the new material contradicts what's already there, don't pick a side. Add a `> [!warning] Conflicting sources` callout that states both claims with their sources, and tag the note `to_verify`.

**Guardrails (every mode)**

- Never set `status: evergreen`. Only Scott promotes notes.
- Tag `to_verify` when:
  - the content comes from an AI conversation, or
  - you add a claim that isn't in the source (including your own .NET example), or
  - the transcription was uncertain.
- Never touch `.obsidian/`, `90 Templates/`, `10 Maps/` or `99 System/Bases/`. Domain maps pick up notes automatically through `domain`.
- Never delete anything. If an item isn't worth keeping, say so in the report and let Scott delete it.
- If an item can't be processed (a paywalled URL, an unreadable image), leave it in the Inbox, add `to_verify` to its tags, and report why.

## Step 6: Verify

```bash
cd "$VAULT" && python3 - "$START" <<'EOF'
import os, re, sys
since = float(sys.argv[1]) if len(sys.argv) > 1 else 0
def walk():
    for r, d, fs in os.walk('.'):
        d[:] = [x for x in d if not x.startswith('.')]
        for f in fs: yield r, f
names, notes, bad = set(), set(), []
for r, f in walk():
    names.add(f)
    if f.endswith('.md'): notes.add(f[:-3])
for r, f in walk():
    p = os.path.join(r, f)
    if not f.endswith('.md') or os.path.getmtime(p) < since: continue
    t = open(p, encoding='utf-8', errors='ignore').read()
    if not t.startswith('---') or not re.search(r'^type:', t.split('---')[1], re.M):
        bad.append((p, 'missing frontmatter or type'))
    body = re.sub(r'```.*?```', '', t, flags=re.S)            # skip code fences
    body = re.sub(r'`[^`\n]*`', '', body).replace("''", "'")  # skip inline code; unescape YAML quotes
    for m in re.findall(r'!?\[\[([^\]|#]+)', body):
        tgt = os.path.basename(m.strip().rstrip('\\'))
        if tgt and tgt not in notes and tgt not in names:
            bad.append((p, f'unresolved link: {tgt}'))
print('\n'.join(f'{a}: {b}' for a, b in bad) or 'OK')
EOF
```

`START` is the epoch time (`date +%s`) recorded in Step 0, before any writes. Each shell call is a fresh shell, so paste the number in literally. The check covers only files changed since then; the 47 broken O'Reilly image embeds that predate this skill show up only if one of those notes is touched, and can be ignored. Fix anything else before reporting, and confirm each processed item has left `00 Inbox`.

## Step 7: Report

Reply with a short summary:

- items processed
- notes created and notes extended (as `[[wikilinks]]`)
- anything flagged `to_verify` or left in the Inbox, and why

Skip the step-by-step narration.

## Step 8: Run log (unattended runs, or when Scott asks)

Append to `99 System/Ingest Log.md`. If the file doesn't exist, create it with frontmatter `type: reference`, `status: evergreen`, `tags: [vault_system]` and a one-line summary. Each entry:

```markdown
## YYYY-MM-DD HH:mm
| Inbox item | Source note | Created | Extended | Flags |
|---|---|---|---|---|
```

Scott reviews new `draft` / `to_verify` notes weekly (Phase 3 routine in the Vault Charter).

---

## Google Drive fallback (no linked computer)

Use this path when no computer is linked to the session but the Google Drive connector is available. Everything above still applies (templates, conventions, guardrails) except where this section overrides it. Changes sync down to the PC through Google Drive for Desktop.

### What the Drive tools can and can't do

| Can | Can't |
|---|---|
| Search and list files (`search_files`, by `title` or `parentId`) | Edit a file's content in place |
| Read files (`download_file_content` returns base64) | Append to a file |
| Create files and folders (`create_file`) | Delete anything |
| Rename or move (`update_file` with `title` / `parentId`) | Run the Step 2 / Step 6 scripts over the vault |

`read_file_content` doesn't support `text/markdown`; use `download_file_content` and decode the base64 in the cloud shell (`base64 -d`).

### Step 0 (Drive)

- Find the root: `search_files` with `title = 'Vault Charter.md'`. Its `parentId` is the vault root.
- List the top-level folders with `parentId = '<root>'`, then drill down the same way to `90 Templates`, `99 System/Prompts`, `00 Inbox`, `20 Atlas/<domain>`, `30 Sources/<type>` and `99 System/Attachments`. Keep the folder IDs; the writes need them.
- Read the templates and prompts listed in Step 0 with `download_file_content`.

### Step 1 (Drive)

List `00 Inbox` with `parentId = '<inbox id>'`; apply the same eligibility rules and ordering (`createdTime`, then `modifiedTime`).

### Step 2 (Drive): index

There's no recursive walk. Instead:

1. Search `mimeType = 'text/markdown'` with `title contains` clauses for the key terms and acronyms of each candidate concept (OR them together in one query).
2. List the target domain folder(s) under `20 Atlas` by `parentId`.
3. Download the frontmatter of the likely matches to check `aliases` and `summary`.

Also confirm the domain MOC notes you plan to link exist (they sit together in one folder; `Architecture.md`, `System Design.md`, etc.).

### Step 3 (Drive): images

- **Image already in Drive:** `download_file_content`, decode to the scratchpad, then Read it.
- **Image in the user's open browser tab (quick capture):** page JavaScript can't hand back base64 (the tool blocks it), so capture it from the screen:
  1. Pin it at natural size: `img.style.cssText='position:fixed;left:0;top:0;width:<naturalWidth>px;height:<naturalHeight>px;margin:0;z-index:9999'`.
  2. Hover the mouse far from the image so the cursor isn't in the capture.
  3. Take a small screenshot to learn the coordinate frame. It can differ from CSS pixels (device pixel ratio or browser zoom), so read the image's edges off the screenshot rather than computing them.
  4. `zoom` that region with `save_to_disk: true` and **no `scale`** (a scale shrinks the saved file too).
  5. Trim any white edge with PIL and save as JPEG (quality 90) in the scratchpad, named after the Atlas note.
  6. Reload the tab afterwards to undo the style change.

  The result is a screen capture, not the original file; mention that in the report.

### Step 4 (Drive)

Add a line under the plan table saying the run uses the Drive fallback and listing what that changes for this run (for example, "written straight to final folders" or "extend X is manual").

### Step 5 (Drive): writing

- **New notes:** `create_file` with `parentId` = the target folder ID, `contentMimeType: text/markdown` and `disableConversionToGoogleType: true`. Check first that the title doesn't already exist in that folder; add a distinguishing suffix if it does.
- **Source notes:** since a file can't be rewritten in place, create the finished Source note directly in its `30 Sources/...` folder.
  - *Quick capture:* skip writing to `00 Inbox` first; it would only leave a stale copy.
  - *Process Inbox:* after the Source note is created, move the original Inbox item out with `update_file` (`parentId` = `99 System/.ingest-tmp`, created if missing). That takes it out of the Inbox without deleting it. List it in the report so Scott can delete it.
- **Extending an existing Atlas note isn't possible.** Don't create a near-duplicate note instead. Put the exact additions (section, text, new source link, new aliases) in the report as ready-to-paste blocks, and mark the item "extend pending" in the Step 4 plan.
- **Images:** `create_file` with base64 is impractical for anything but tiny images (the base64 has to pass through the tool call). Upload through the Drive web UI instead:
  1. Open `https://drive.google.com/drive/folders/<Attachments id>` in a new tab.
  2. Inject a file input with JavaScript (`input type=file`, fixed position, with an id), `find` it, and `file_upload` the scratchpad image to it.
  3. With JavaScript, build a `DataTransfer` holding `input.files[0]` and dispatch `dragenter`, `dragover` and `drop` (bubbling, cancelable, with that `dataTransfer`) on the element under the file list (`document.elementFromPoint(...)`). Then remove the input.
  4. Wait a few seconds; confirm with `search_files` (`title` + `parentId`) that the file exists and its size matches.
  5. Close the tab.

### Step 6 (Drive): verify

There's no script. For every created note:

- download it once and check the frontmatter parses and has `type`;
- confirm each `[[wikilink]]` and `![[embed]]` target exists with a `search_files` on `title = '<name>.md'` (or the image file name).

In the report, say the Step 6 script didn't run and the links were checked by hand.

### Step 8 (Drive): run log

The log can't be appended to through Drive. Put the log entry (same table format) in the report and say `Ingest Log.md` wasn't updated. In an unattended run, send it with `SendUserMessage` so it isn't lost.