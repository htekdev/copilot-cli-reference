---
name: "Weekly Content Update"
description: |
  Automated weekly workflow that researches the latest GitHub Copilot CLI
  information using MCP servers (Exa, YouTube, MS Learn, Perplexity) and
  updates the static HTML reference site with current data.

on:
  schedule: weekly
  workflow_dispatch:

timeout-minutes: 45
strict: false

permissions: read-all

network:
  allowed:
    - defaults
    - node
    - python
    - mcp.exa.ai
    - learn.microsoft.com
    - www.googleapis.com
    - youtube.googleapis.com

tools:
  github:
    toolsets: [repos, issues, pull_requests]
  edit:
  bash:
    - "*"
  web-fetch:
  cache-memory: true

mcp-servers:
  exa:
    url: "https://mcp.exa.ai/mcp?tools=web_search_exa,web_search_advanced_exa,crawling_exa,deep_researcher_start,deep_researcher_check&exaApiKey=${{ secrets.EXA_API_KEY }}"
    allowed: ["*"]
  mslearn:
    url: "https://learn.microsoft.com/api/mcp"
    allowed: ["*"]
  perplexity:
    command: "npx"
    args: ["-y", "perplexity-mcp"]
    env:
      PERPLEXITY_API_KEY: "${{ secrets.PERPLEXITY_API_KEY }}"
    allowed: ["*"]
  youtube:
    command: "npx"
    args: ["-y", "@htekdev/youtube-mcp-server"]
    env:
      YOUTUBE_API_KEY: "${{ secrets.YOUTUBE_API_KEY }}"
    allowed: ["*"]

safe-outputs:
  create-pull-request:
    title-prefix: "[content-update] "
    labels: [copilot-update, automation]
    draft: false
    if-no-changes: "warn"
  noop:
---

# Weekly Content Update Agent

You are a content maintenance agent for the **GitHub Copilot CLI Reference Site** at `${{ github.repository }}`. Your job is to ensure this static HTML site always has the latest information about GitHub Copilot CLI.

## Site Structure

This is a static HTML site (no build system, no framework) deployed via GitHub Pages. All data is hardcoded in HTML and JavaScript. The key files are:

| File | Content | Update Priority |
|------|---------|----------------|
| `release-velocity.html` | Release tracking with 90+ releases, Chart.js charts, searchable table, competitor comparison | **HIGH** — new releases happen frequently |
| `learning-paths.html` | 17+ curated YouTube videos across 4 learning paths with thumbnails, durations, channel badges | **HIGH** — new videos published regularly |
| `references.html` | 50+ curated links to official docs organized by category | **MEDIUM** — links can go stale or new docs added |
| `index.html` | Main reference hub with version number, keyboard shortcuts, slash commands, CLI flags, deprecated commands | **HIGH** — commands change with each release |

## Your Mission

Work through these phases in order. Use cache memory to track what you've already updated in previous runs. Only make changes when you find genuinely new information.

## Phase 1: Check Cache Memory

Load your cache memory to see what was updated in previous runs. This tells you:
- The last version number you recorded
- Which releases were already added
- Which YouTube videos were already included
- Which commands/shortcuts are documented and their badge status
- When the last successful update happened

If this is your first run, the cache will be empty — proceed with all phases.

## Phase 2: Research Latest Copilot CLI Releases

Use **Exa** and **Perplexity** to research the latest GitHub Copilot CLI releases.

### What to search for:
- Search for "GitHub Copilot CLI latest release version changelog 2025 2026"
- Search for "github/copilot releases" on GitHub
- Check the GitHub releases page for @github/copilot npm package
- Look for blog posts announcing new Copilot CLI features

### How to update `release-velocity.html`:
The release data is stored as a JavaScript array in the file. Look for the `releases` array with entries in this format:
```javascript
{v:'0.0.421', d:'2026-03-04', t:'patch', h:'Feature description'}
```

Fields:
- `v` = version string
- `d` = date in YYYY-MM-DD format
- `t` = type: 'major', 'minor', or 'patch'
- `h` = short highlight/description of what changed

**Add new releases** to the beginning of the array (newest first). Also update:
- The total release count in the stats section
- The Chart.js data arrays for cumulative and monthly charts
- The hero version number if there's a newer version

Also update the **competitor comparison** data if you find new release information for:
- Claude Code (Anthropic)
- Cursor
- OpenAI Codex CLI

### How to update `index.html`:
- Update the version number in the hero section (look for the version badge/tag)
- Update any feature descriptions if new capabilities were announced

## Phase 3: Research New YouTube Content

Use the **YouTube MCP** to search for new relevant videos about GitHub Copilot CLI.

### What to search for:
- "GitHub Copilot CLI" (published in last 30 days)
- "GitHub Copilot terminal" (published in last 30 days)
- "Copilot CLI tutorial" (published in last 30 days)
- "GitHub Copilot agent mode CLI" (published in last 30 days)

### Quality criteria for including a video:
- From reputable channels (GitHub official, Microsoft, well-known tech educators)
- At least 1,000 views (or from official channels regardless of views)
- Directly about Copilot CLI (not just mentioning it briefly)
- Not a duplicate of existing videos in the list

### How to update `learning-paths.html`:
Videos are organized in learning path sections. Each video card contains:
- YouTube thumbnail image
- Video title
- Channel name with badge
- Duration
- Difficulty level chip (Beginner, Intermediate, Advanced)

Add new videos to the appropriate learning path section based on content:
1. **Getting Started** — Installation, setup, first use
2. **Core Features** — Agent mode, MCP, slash commands
3. **Advanced Workflows** — Automation, CI/CD integration, custom agents
4. **Real-World Projects** — Full project builds, demos

Match the existing HTML card structure when adding new videos.

## Phase 4: Audit Commands, Shortcuts & CLI Flags Against Actual Releases

This is the **most critical phase**. The cheat sheet in `index.html` must reflect the actual current state of the CLI. You must go through **real release notes** to discover what was added, changed, or removed — not just search for generic lists.

### Step 1: Read the actual changelog

Use **web-fetch** and **GitHub tools** to read the primary sources of truth:

1. **Crawl the changelog**: Fetch `https://github.com/github/copilot-cli/blob/main/changelog.md` — this is the definitive source. Read every release entry from the current version shown on the cheat sheet through the latest release. Look for:
   - Lines starting with "Add" or "New" → these are **new commands, flags, shortcuts, or features**
   - Lines mentioning "Rename" → a command/flag name changed
   - Lines mentioning "Remove", "Drop", or "Deprecate" → a feature was removed
   - Lines mentioning keyboard shortcuts, keybindings, Ctrl+, Shift+, or hotkeys
   - Lines mentioning new slash commands (e.g., `/something`)
   - Lines mentioning new CLI flags (e.g., `--something`)

2. **Check the releases page**: Fetch `https://github.com/github/copilot-cli/releases` — this has structured "Added", "Improved", "Fixed" sections for each release. Read through every release since the version currently on the cheat sheet.

3. **Check the npm package**: Fetch `https://www.npmjs.com/package/@github/copilot` to confirm the latest published version number.

4. **Check the GitHub blog changelog**: Use Exa to search `site:github.blog/changelog copilot CLI` for feature announcement posts that describe new commands/modes.

5. **Check official docs**: Use MS Learn to search `GitHub Copilot CLI commands reference` and `GitHub Copilot CLI flags` for the latest documented command set.

### Step 2: Build a diff of changes

From the release notes, build three change lists:

**New items to add to the cheat sheet:**
- New slash commands (e.g., a release note says "Add `/newcmd` command for X")
- New CLI flags (e.g., "Add `--new-flag` to enable Y")
- New keyboard shortcuts (e.g., "Press Ctrl+R to search command history")
- New input shortcuts (e.g., new `@`, `!`, `&` syntax)
- New modes or configuration options

**Items to update:**
- Renamed commands/flags (e.g., "Rename `--old-flag` to `--new-flag`")
- Changed descriptions (e.g., a command now does something different)
- Changed default model (for the hero version line)

**Items to deprecate/remove:**
- Removed commands or flags
- Deprecated features

### Step 3: Cross-reference with index.html

Read the current tables in `index.html` under `#quickstart`:

1. **Essential Shortcuts** table (class `mini-table` inside "Essential Shortcuts" block)
2. **Key Slash Commands** table (class `mini-table` inside "Key Slash Commands" block)
3. **CLI Flags** table (class `mini-table` inside "CLI Flags (Command Line)" block)
4. **Deprecated Commands** table (class `mini-table` inside "Deprecated Commands" block)

For each item in your change lists:
- If a new item doesn't exist on the cheat sheet → **add it** with `<span class="badge-new">New</span>`
- If a renamed item exists under its old name → **update** the name and add a note
- If a deprecated item still appears as active → **move it** to the Deprecated section

Also check for items currently on the cheat sheet that may have been removed without you noticing — cross-reference each existing entry against the changelog.

### Step 4: Apply changes to index.html

Follow these exact HTML patterns:

**Normal command:**
```html
<tr><td><code>/command</code></td><td class="desc">Description of what it does</td></tr>
```

**New command (added in a recent release):**
```html
<tr><td><code>/command</code></td><td class="desc">Description <span class="badge-new">New</span></td></tr>
```

**Deprecated command:**
```html
<tr class="deprecated"><td><code>/command</code></td><td class="desc">Removed [date] — replacement info <span class="badge-deprecated">Deprecated</span></td></tr>
```

### Step 5: Manage badge lifecycle

- **New badge**: Add to commands from releases within the last 3 months. Remove from commands older than 3 months.
- **Deprecated badge**: Permanent. Move deprecated entries to the "Deprecated Commands" table.

### Step 6: Update hero version

Update the hero version line if the latest release is newer:
```html
<p class="hero-version">v0.0.XXX · [default model] · [Month Year]</p>
```

Also update the version in the ecosystem section and configuration section if referenced.

## Phase 5: Validate and Update References

Use **MS Learn MCP** and **web-fetch** to validate external links in `references.html`.

### What to check:
- Use web-fetch to test key reference URLs for HTTP status
- Search MS Learn for any new official Copilot CLI documentation pages
- Check if any existing links have moved or been deprecated

### How to update `references.html`:
- Fix any broken links (find the new URL or remove if content no longer exists)
- Add new official documentation links in the appropriate category
- Update link descriptions if page titles have changed

## Phase 6: Update Timestamps

After making any changes:
- Update "Last updated" date references in page footers to the current month/year
- Ensure consistency across all pages

## Phase 7: Create Pull Request or Noop

### If changes were made:
Use the `create-pull-request` safe output with:
- Title: "Update site content — [current date]"
- Body including:
  - Summary of what was updated
  - New releases added (if any)
  - New videos added (if any)
  - Commands/shortcuts added, updated, or deprecated (if any)
  - Links fixed/added (if any)
  - Data sources used

### If no changes needed:
Use the `noop` safe output with a message like:
- "All content is up to date as of [date]. No new releases, videos, or link changes found."

## Phase 8: Update Cache Memory

Save to cache memory:
- The latest version number recorded
- List of release versions now in the data
- List of YouTube video IDs now included
- List of all commands/shortcuts currently documented (for diff comparison next run)
- Which commands have "New" badges and when they were added (for 3-month lifecycle)
- Timestamp of this run
- Summary of changes made (or "no changes")

## Important Guidelines

- **Accuracy over quantity** — Only add information you can verify from multiple sources
- **Preserve existing data** — Never remove existing entries unless they are confirmed incorrect
- **Match existing style** — New HTML should match the existing design patterns exactly
- **No placeholder data** — Every entry must have real, verified information
- **Be conservative** — If uncertain about a data point, skip it rather than add incorrect information
- **Respect rate limits** — Don't make excessive API calls; be efficient with MCP tool usage
- **Badge lifecycle** — "New" badges expire after 3 months; "Deprecated" badges are permanent
- **CLI-only content** — YouTube videos must be about the standalone Copilot CLI, not VS Code, Visual Studio, or the Coding Agent
- **Trusted sources only** — Only cite official GitHub, Microsoft, or verified community sources for command documentation
