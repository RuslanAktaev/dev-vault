---
name: obsidian
description: Work with an Obsidian vault through the official Obsidian CLI — read, create, edit, move and delete notes, manage frontmatter properties, tags, tasks, daily notes, templates, bases, links/backlinks, search, plugins and history. Use whenever the user mentions Obsidian, a vault, notes/заметки, daily note/ежедневная заметка, wikilinks, backlinks, frontmatter properties, .base files, or asks to find/organize/rename/link notes — also when the working directory contains a `.obsidian/` folder.
---

# Obsidian CLI

The Obsidian CLI (`obsidian`, docs: https://obsidian.md/cli) talks to the **running Obsidian app**, so every change goes through Obsidian itself: renames and moves update links, and the metadata cache (links, tags, tasks, properties) stays accurate. Use the CLI instead of editing `.md` files directly whenever the change touches links, file names or frontmatter.

## Always call it through the wrapper

```bash
OBS=.claude/skills/obsidian/scripts/obs   # path relative to the vault root
$OBS <command> [key=value ...] [flags]
```

The raw `obsidian` binary has quirks that the wrapper fixes:
- with an outdated installer it prints noise lines ("Loading updated app package…", "installer is out of date…") to **stdout** before every result, which breaks JSON parsing. The wrapper strips them (since installer 1.13.7 they no longer appear).
- it **always exits 0**, even on `Error: File "x" not found.` The wrapper exits 1 when the output starts with `Error:`.
- a call can hang (e.g. `obsidian help` with an outdated installer). The wrapper stops it after `OBS_TIMEOUT` seconds (default 30) and exits 124.

### Modal dialogs silently block file operations

Obsidian may open a dialog inside the app, most often **"Update links?" / «Обновить ссылки?»** after a `rename`/`move` of a note that other notes link to. This happens when *Settings → Files & links → Automatically update internal links* is off, which is the default. While the dialog is open, every later `move`/`rename`/`delete` **prints nothing, exits 0 and does nothing**. The operations are queued and run all at once after the dialog closes. After any rename or move, and whenever a write "succeeded" but the file didn't change, check for a dialog:

```bash
$OBS dev:dom selector=".modal-container" all text     # "No elements found." = no dialog open
```

If one is open, tell the user what it asks and let them decide. Don't close it on your own. If the user wants it answered, list the buttons and click the one they chose:

```bash
$OBS eval code="[...document.querySelectorAll('.modal-container button')].map(b=>b.textContent).join('|')"
$OBS eval code="document.querySelectorAll('.modal-container button')[1].click()"   # [1] = "Только сейчас / Update now"
```

To avoid the dialog, suggest that the user turn on automatic link updating in Obsidian's settings. Don't change the setting yourself.

If a call times out or says it can't connect, Obsidian is probably not running: run `open -a Obsidian`, wait a few seconds, then retry.

## Syntax

- Arguments are `key=value`. Flags are bare words: `total`, `overwrite`, `permanent`, `open`.
- Quote values that contain spaces: `name="My Note"`, `content="$text"`.
- `file=<name>` resolves the way a wikilink does (name only, no `.md`, searched across the vault). `path=<folder/note.md>` is an exact path relative to the vault root. **Use `path=` for writes and for anything ambiguous.** Base files also need `path=` (`path="Board.base"`).
- If you omit `file`/`path`, most commands act on the **file that is currently active** in Obsidian. Don't rely on that unless the user means "the current note".
- In `content=`, `\n` becomes a newline and `\t` becomes a tab, so the note body can be passed as one argument. A literal backslash-n in the text (for example inside a code block) will also turn into a newline. For content like that, write the file with the Write tool instead.
- Target a different vault with `vault=<name>` before the command. List vaults with `$OBS vaults verbose`.
- Prefer `format=json` wherever it exists (search, tasks, tags, backlinks, bookmarks, base:query, plugins, unresolved…) and parse it with `jq`.

## Common recipes

```bash
# Orientation
$OBS vault                       # name, path, file/folder counts
$OBS files folder="Projects"     # list files (ext=md, total)
$OBS folders
$OBS recents

# Read
$OBS read path="Projects/Alpha.md"
$OBS outline path="Projects/Alpha.md" format=md
$OBS file path="Projects/Alpha.md"          # metadata: size, ctime, mtime

# Create and edit
$OBS create path="Inbox/Idea.md" content="# Idea\n\nText"   # add overwrite to replace an existing file
$OBS create path="Meetings/2026-09-19.md" template="Meeting"
$OBS append  path="Inbox/Idea.md" content="- another point"
$OBS prepend path="Inbox/Idea.md" content="> summary"
# For an edit in the middle of a note: read the note, then use the Edit tool on
# <vault path>/<path>. Obsidian picks up changes made on disk automatically.

# Rename and move: links are updated only if the setting is on or the user confirms the dialog (see above)
$OBS rename path="Inbox/Idea.md" name="Better idea"   # prints nothing on success
$OBS move   path="Inbox/Idea.md" to="Projects/"       # the target folder must exist
$OBS dev:dom selector=".modal-container" total        # then check for a blocking dialog

# Delete (moves to trash by default; permanent skips the trash). Confirm with the user first.
$OBS delete path="Inbox/Idea.md"

# Properties (frontmatter)
$OBS properties path="Projects/Alpha.md" format=json
$OBS property:read   path="Projects/Alpha.md" name=status
$OBS property:set    path="Projects/Alpha.md" name=status value=active
$OBS property:set    path="Projects/Alpha.md" name=tags value="work, q3" type=list
$OBS property:set    path="Projects/Alpha.md" name=due value=2026-10-01 type=date
$OBS property:remove path="Projects/Alpha.md" name=draft
$OBS properties counts sort=count            # every property in the vault

# Search
$OBS search query="kubernetes" format=json limit=20       # returns matching file paths
$OBS search:context query="kubernetes" path="Projects"    # returns matching lines

# Links and graph
$OBS links path="Projects/Alpha.md"
$OBS backlinks path="Projects/Alpha.md" counts format=json
$OBS unresolved verbose format=json     # broken wikilinks and the files that contain them
$OBS orphans                            # notes nothing links to
$OBS deadends                           # notes that link to nothing

# Tags
$OBS tags counts sort=count format=json
$OBS tag name=project verbose

# Tasks
$OBS tasks todo format=json                  # [{status, text, file, line}]
$OBS tasks path="Projects/Alpha.md" verbose
$OBS task ref="Projects/Alpha.md:12" done    # also: toggle | todo | status="/"
$OBS tasks daily todo

# Daily notes
$OBS daily:path
$OBS daily:read
$OBS daily:append content="- [ ] Call Bob"

# Templates and bases
$OBS templates
$OBS template:read name="Meeting" resolve title="Sync"
$OBS bases
$OBS base:query path="Board.base" format=json   # also view=<name>; format=md|csv|paths
$OBS base:create path="Board.base" name="New item" content="..."

# App-level
$OBS open path="Projects/Alpha.md" newtab
$OBS commands filter=editor   # command IDs, to run with: $OBS command id=<id>
$OBS plugins:enabled format=json
$OBS history path="X.md"  /  $OBS history:read path="X.md" version=2
```

The full command reference, with every option, is in `commands.txt` next to this file. Read it (or run `$OBS help <command>`) before you use a command that isn't listed above.

## Working rules

1. **Look before you write.** Check that a note exists before you create it, so you don't overwrite a note or end up with duplicates. Before appending or editing, read the note so the new text matches its structure and style.
2. **Renames and moves go through the CLI**, never `mv`, so wikilinks stay correct.
3. **Destructive actions need confirmation**: `delete`, `create … overwrite`, `history:restore`, `sync:restore`, `plugin:uninstall`, `theme:uninstall`, and `eval`/`dev:*`. Prefer trash over `permanent`.
4. **Write in Obsidian conventions**: `[[wikilinks]]` (`[[Note|alias]]`, `[[Note#Heading]]`), `#tags` or a `tags:` list property, `- [ ]` tasks, callouts (`> [!note]`). Before inventing new conventions, check what the vault already uses (`properties counts`, `tags counts`, a couple of existing notes).
5. **Batch reads with json + jq** rather than calling the CLI once per file when you only need metadata, for example `$OBS tasks todo format=json | jq 'group_by(.file)'`.
6. When the CLI can't do something (for example a regex replace across many notes), edit the files directly in the vault directory (`$OBS vault info=path`). Then run `$OBS unresolved` to confirm no links broke.
