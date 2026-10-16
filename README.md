# move-usings-to-global

A Claude Code skill that consolidates per-file C# `using` statements into a single `GlobalUsings.cs` file per project.

It wraps the `move_usings_to_global.py` script (a Python port of `Move-UsingsToGlobal.ps1`, cross-platform).

---

## What it does

For every `.csproj` file found under the root path:

1. Creates `<ProjectDir>/GlobalUsings.cs` if it does not exist yet.
2. Scans every `.cs` file in that project (skips `bin`, `obj`, `.git`, `.vs`, and `GlobalUsings.cs` itself).
3. Reads only the **header block** of each file: the run of blank lines, comments, `#` directives, and `using` statements at the top. Scanning stops at the first other line (`namespace`, `class`, and so on), so usings placed inside a namespace block are left untouched.
4. **Never moves**:
   - Alias usings, for example `using Entity = My.Entities.Product;`
   - `using static X;` directives
   - Lines that are already `global using ...;`
5. Removes every other `using` line from the file and adds it to `GlobalUsings.cs` as `global using X;` (deduplicated and sorted).
6. Cleans up spacing before the first line after the header (for example `namespace Foo;`): removes dangling blank lines, then inserts exactly one blank line if any header content (alias usings, static usings, comments, directives) remains above it.

Re-running is safe. Existing entries in the `GlobalUsings.cs` file are read first and are never duplicated. The BOM of each `.cs` file is preserved.

---

## Why a skill (and not a command, subagent, hook, or MCP server)

| Option | Fit | Reason |
|---|---|---|
| **Skill** | Best | Bundles the `.py` file next to its instructions. Works as a `/slash-command`. |
| Command only | OK | A single markdown file cannot cleanly carry the script. |
| Subagent | Overkill | Meant for tasks that need their own context or judgment. This script is deterministic. |
| Hook / MCP server | Overkill | Too heavy for a one-off refactoring script. |

---

## Folder structure

```
.claude/skills/move-usings-to-global/
├── SKILL.md
├── README.md
└── move_usings_to_global.py
```

- Put it in `.claude/skills/` inside your repo to use it in that project only.
- Put it in `~/.claude/skills/` to use it in all your projects.

---

## Installation

1. Create the folder: `.claude/skills/move-usings-to-global/`
2. Copy `move_usings_to_global.py` into the folder.
3. Create the `SKILL.md` file in the same folder (content below).
4. Restart Claude Code.
5. Type `/` and check that `move-usings-to-global` appears in the list.

### SKILL.md file

```markdown
---
name: move-usings-to-global
description: Consolidates per-file C# using statements into a GlobalUsings.cs file per project. Use when the user asks to move usings to global usings.
disable-model-invocation: true
argument-hint: [root-path]
allowed-tools: Bash(python3 *), Bash(python *)
---

# Move usings to GlobalUsings.cs

Root path: `$ARGUMENTS` (use the current directory if empty).

## Steps

1. Run a preview first. Do not skip this step:
   `python3 ${CLAUDE_SKILL_DIR}/move_usings_to_global.py --root-path "<root>" --whatif`
2. Summarize the preview: projects found, number of files to update, number of global usings per project.
3. Ask the user to confirm before making changes.
4. After confirmation, run the same command without `--whatif`.
5. Run `dotnet build` and report any errors.

## Rules

- Do not edit the .cs files manually. The script handles all changes.
- Alias usings and `using static` directives are intentionally not moved.
```

### About `disable-model-invocation: true`

The script changes many files, so only you can trigger it by typing the slash command. Claude will not run it on its own, and it will not load the skill description into its context.

---

## Usage

### In Claude Code

```
/move-usings-to-global C:\path\to\solution
```

or, for the current directory:

```
/move-usings-to-global
```

Claude will:

1. Run the preview (`--whatif`) and summarize it.
2. Ask for your confirmation.
3. Run the real change.
4. Run `dotnet build` and report any errors.

### Running the script directly (without Claude)

```bash
# Preview only, no files are written
python move_usings_to_global.py --root-path C:\path\to\solution --whatif

# Apply the changes
python move_usings_to_global.py --root-path /path/to/solution
```

### Script options

| Option | Default | Description |
|---|---|---|
| `--root-path` | `.` | Root folder to search for `.csproj` files. |
| `--exclude-dirs` | `bin obj .git .vs` | Folder names to skip anywhere in the path. |
| `--whatif` | off | Preview changes without writing any files. |

---

## Example

**Before** (`Services/OrderService.cs` file):

```csharp
using System;
using System.Collections.Generic;
using Entity = My.Entities.Product;
using static System.Math;

namespace My.Services;
```

**After** (`Services/OrderService.cs` file):

```csharp
using Entity = My.Entities.Product;
using static System.Math;

namespace My.Services;
```

**`GlobalUsings.cs` file:**

```csharp
global using System;
global using System.Collections.Generic;
```

---

## CLAUDE.md file

You do **not** need to reference this skill in the `CLAUDE.md` file. Claude Code finds skills automatically from the `.claude/skills/` folder (project) or the `~/.claude/skills/` folder (personal).

A `CLAUDE.md` file entry is only useful if you want Claude to recommend the skill. In that case:

```markdown
## Skills
- `/move-usings-to-global`: consolidates C# usings into GlobalUsings.cs. Suggest it when a project has many per-file usings.
```

If you add this, also remove `disable-model-invocation: true` from the `SKILL.md` file, otherwise the setting still blocks Claude from running the skill by itself. Keeping it manual-only is the safer choice, so skipping the `CLAUDE.md` file entry is recommended.

---

## Tips

- Commit or stash your changes first, so you can review the result with `git diff`.
- Always run with `--whatif` first (the skill does this for you).
- On Windows, `python3` may not exist. If so, change the command in the `SKILL.md` file to `python`.
- If `${CLAUDE_SKILL_DIR}` does not resolve in your version of Claude Code, use the relative path `.claude/skills/move-usings-to-global/move_usings_to_global.py` instead.
- After the run, build the solution. A moved using can expose naming conflicts that were hidden before.

---

## Using it in claude.ai (not Claude Code)

Skills also work in claude.ai:

1. Zip the `move-usings-to-global` folder.
2. Upload it in **Settings** under **Capabilities**.

Claude can then run the script in its sandbox, but it only sees files you upload. For a real solution folder, Claude Code is the better fit.

---

## Known limits

- Only the top header block of each file is scanned. Usings inside a namespace block stay as they are.
- Multi-line `/* ... */` comments in the header are not treated as pass-through, so scanning stops at them.
- Alias usings and `using static` directives are never moved.
