
# Imply Docs — Copy Markdown

https://github.com/mdtalhachy/imply-docs-copy-markdown/raw/main/imply-copy-markdown-demo.mp4

A Chrome extension that adds a **Copy Markdown** button to every page on [docs.imply.io](https://docs.imply.io), extracting article content as clean, LLM-optimized markdown.

## What It Does

Click the **Copy MD** button next to any page title to copy the full article as markdown, including:

- **YAML frontmatter** with title, source URL, product name, and breadcrumb path
- **Table of Contents** with navigable anchor links (H2-only, for structural overview)
- **All tab content** — cURL, Python, Java, etc. are all captured with labeled sections, not just the active tab
- **ATX headings** preserving the document hierarchy
- **Fenced code blocks** with language annotations (SQL, JSON, etc.)
- **GitHub-style callouts** for admonitions (`> [!NOTE]`, `> [!WARNING]`, etc.)
- **Collapsible sections** preserved as `<details>` / `<summary>` HTML
- **GFM tables** for tabular content
- **Image placeholders** (`*[Image: alt text]*`) since LLMs can't fetch URLs
- **Absolute URLs** for all internal links

### Example Output

```markdown
---
title: Quickstart
source: "https://docs.imply.io/polaris/quickstart/"
product: Imply Polaris
path: "Getting started > Quickstart"
---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Sign up for an account](#sign-up-for-an-account)
- [Load data](#load-data)
- [Query data](#query-data)

## Prerequisites

You must use one of the following supported browsers:

- Last three versions of Chrome, Firefox, Edge (Chromium version), Opera
- Firefox ESR
- Safari (desktop) 16 or above

> [!NOTE]
> Polaris doesn't accept free email addresses for new accounts.
```

On pages with tab groups (e.g., API references), all tabs are captured:

```markdown
**cURL:**

curl -X POST https://api.imply.io/v1/jobs \
  -H "Authorization: Bearer $API_TOKEN"

**Python:**

requests.post(
    "https://api.imply.io/v1/jobs",
    headers={"Authorization": f"Bearer {API_TOKEN}"}
)
```

## Install

1. Clone or download this repository
2. Open `chrome://extensions` in Chrome
3. Enable **Developer mode** (top right toggle)
4. Click **Load unpacked** and select this project folder
5. Navigate to any page on [docs.imply.io](https://docs.imply.io) — the button appears next to the page title

## Design Decisions

### Why convert rendered HTML instead of scraping raw MDX?

We can't access the Docusaurus source files from a browser extension. Instead, we convert the rendered DOM to markdown using [Turndown.js](https://github.com/mixmark-io/turndown), the industry-standard HTML-to-Markdown converter. Custom Turndown rules intercept Docusaurus-specific components (code blocks, admonitions, tabs) at the container level, producing clean output instead of letting default conversion create messy markdown.

### Why all tabs, not just the active tab?

API reference pages show cURL, Python, Java, etc. in tab groups. Only the active tab is visible, but the hidden panels exist in the DOM with `[hidden]` attributes. Capturing only the active tab loses the majority of the code examples — real content loss. We use index-based pairing (`tabs[i]` labels `panels[i]`) rather than matching `aria-controls` to `id`, because Docusaurus auto-generates those IDs and index pairing is simpler and more reliable.

### Why a Table of Contents?

LLMs benefit from structural context before reading a long document. A TOC with anchor links lets agents (and humans in markdown viewers) scan the page structure and jump to relevant sections. We limit it to H2 headings only — enough for structural overview without noise from H3/H4 subsections.

The TOC is generated from the markdown output (not the DOM), which means it reflects the actual converted content. We strip fenced code blocks before scanning for headings so that `## ` lines inside code examples don't leak into the TOC as false positives.

### Why GitHub-style callouts for admonitions?

Docusaurus admonitions (info, warning, tip, caution) map naturally to GitHub's `> [!NOTE]` / `> [!WARNING]` syntax. These are well-understood by LLMs and render natively in GitHub and most markdown viewers.

### Why image placeholders instead of image URLs?

LLMs can't fetch or display images from URLs. Replacing `<img>` tags with `*[Image: alt text]*` preserves the semantic meaning while keeping the markdown clean and focused on text content.

### Why YAML frontmatter?

The metadata header gives LLMs essential context: what document they're reading, where it came from, which product it covers, and where it sits in the documentation hierarchy. Values containing YAML special characters (`:` in URLs, `>` in breadcrumb paths) are automatically quoted so the frontmatter parses correctly.

### Why preserve `<details>` as raw HTML?

Most markdown renderers pass through `<details>` / `<summary>` natively. Converting to something else would lose the expand/collapse behavior. Keeping the HTML means collapsible sections work when pasted into GitHub, VS Code preview, or any CommonMark-compatible renderer.

## Technical Approach

- **Turndown.js** with 7 custom rules handles the HTML → Markdown conversion
- **DOM cloning + stripping** removes UI chrome (permalink icons, copy buttons, admonition labels) on a clone before conversion, never mutating the live page
- **Stable Docusaurus selectors** (`theme-*` classes from Docusaurus' public theming API) and **ARIA attributes** (`[role="tab"]`, `[role="tabpanel"]`) resist version updates
- **MutationObserver** detects SPA navigation to re-inject the button on client-side page transitions, with ID-based deduplication to prevent double-injection
- **CSS custom properties** from Docusaurus (`--ifm-color-primary`, etc.) ensure the button matches light/dark themes automatically
- **No build tools** — vendored UMD bundles and plain JavaScript for simplicity
- **Automated tests** via jsdom — 29 assertions covering every conversion rule, run with `npm test`

## Testing

The test suite uses jsdom to simulate a browser environment, loading the actual source files (not duplicated rule definitions) and testing the full pipeline from DOM to clipboard:

```
npm install    # one-time: installs jsdom
npm test       # runs 29 assertions
```

Tests cover: frontmatter extraction, code blocks with language annotations, admonitions, all-tab capture (including hidden panels), collapsible sections, image placeholders, internal link resolution, TOC generation with anchor links, TOC code-block exclusion, TOC omission on short pages, and GFM tables.

## File Structure

```
manifest.json               MV3 extension config
content.js                  Core logic (Turndown rules, extraction, TOC, button)
styles.css                  Button styling (uses Docusaurus CSS vars)
turndown.js                 Vendored Turndown UMD bundle
turndown-plugin-gfm.js      Vendored GFM plugin (tables/strikethrough)
icons/                      Extension icons (16/48/128px)
test.js                     Integration tests (jsdom)
package.json                Dev dependency (jsdom) and test script
```
