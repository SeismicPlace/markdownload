# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

MarkDownload: a browser extension (Firefox, Chrome, Edge, Safari) that clips web pages to Markdown. Plain JavaScript, **Manifest V2**, no bundler, no transpiler, no tests, no linter. Files in `src/` ship as-is.

## Commands

All npm commands run from `src/` (that's where `package.json` and `manifest.json` live):

```sh
cd src
npm install                    # installs web-ext (only dev dependency)
npm run build                  # web-ext build -> zipped extension in src/web-ext-artifacts/
npm run start:firefoxdeveloper # launch Firefox Dev Edition with the extension on a random Wikipedia page
npm run start:chromedevwin     # same for Chrome Dev on Windows
```

Or load `src/` unpacked in a browser. There is no test suite; verify changes by clipping real pages.

Releases: bump `version` in `src/manifest.json` and add a "Version History" entry in `README.md` (see commit `7e8cc1a`). `CHANGELOG.md` is old and not kept up to date. The Safari build is the Xcode project in `xcode/`, which wraps the same `src/` files.

## Architecture

No modules: every script shares globals. `manifest.json`'s `background.scripts` loads them **in order** into one background page (polyfill → mime types → moment → turndown → gfm plugin → Readability → `shared/context-menus.js` → `shared/default-options.js` → `background/background.js`). A new background file must be added to that list in the right position. The popup and options pages load their own `<script>` tags from their HTML.

`browser-polyfill.min.js` provides the `browser.*` promise API on Chromium. Use `browser.*`, not `chrome.*`.

### Clip flow (popup)

1. `popup/popup.js` injects `contentScript/contentScript.js` into the tab, then runs `getSelectionAndDom()` via `tabs.executeScript`.
2. Popup sends `{type: "clip", dom, selection, clipSelection}` to the background.
3. `background.js` `notify()`: `getArticleFromDom()` parses the DOM string with `DOMParser`, runs Readability, and builds an `article` object (title, byline, excerpt, keywords, baseURI, meta tags, etc.). Then `convertArticleToMarkdown()` runs Turndown with custom rules (`turndown()` near the top of the file) and applies the front/back-matter templates.
4. Background sends `{type: "display.md", markdown, article, imageList, mdClipsFolder}` back. The popup shows it in CodeMirror for editing.
5. Download: popup sends `{type: "download", ...}` and the background's `downloadMarkdown()` saves it.

Context menus and keyboard commands (`browser.contextMenus.onClicked`, `browser.commands.onCommand` in `background.js`) skip the popup. They call `ensureScripts()` and `getArticleFromContent()` directly. Clipboard copies and Obsidian sends run `copyToClipboard(...)` in the page via `executeScript`.

### Key concepts

- **Options**: `shared/default-options.js` defines `defaultOptions` and `getOptions()` (reads `browser.storage.sync`, fills in defaults). Adding an option means updating `defaultOptions`, `options/options.html` and `options/options.js`, and possibly `shared/context-menus.js` (some options are context-menu checkboxes toggled through `toggleSetting()`).
- **Templates**: `textReplace()` in `background.js` substitutes `{key}` from any `article` property, plus casing suffixes (`:lower`, `:kebab`, `:mixed-kebab`, `:snake`, `:obsidian-cal`, `:camel`, `:pascal`…), `{date:FORMAT}` (moment.js) and `{keywords:sep}`. Any leftover `{...}` is stripped. Used for front/back matter, the title, `imagePrefix` and `mdClipsFolder`. The template variables are documented in `user-guide.md`; keep it in sync.
- **Download modes**: `downloadsApi` uses `browser.downloads` with a Blob URL, plus separate downloads per image when `downloadImages` is on. `contentLink` injects into the page and clicks an `<a download>` via the content script; it's forced when `browser.downloads` is missing (Safari).
- **Obsidian**: copies the Markdown to the clipboard, then opens `obsidian://advanced-uri?...&clipboard=true` (needs the Advanced Obsidian URI plugin).
- **MathJax**: `contentScript.js` injects `pageContext.js` into the page's own JS context so it can read MathJax 3 state. It tags math nodes with a `markdownload-latex` attribute, and `getArticleFromDom()` swaps those nodes for their TeX source before Readability runs.

### Vendored libraries

Don't edit these unless you're upgrading them deliberately; the versions are listed in README "External Libraries": `background/Readability.js`, `background/turndown.js`, `background/turndown-plugin-gfm.js`, `background/moment.min.js`, `background/apache-mime-types.js`, `browser-polyfill.min.js`, `popup/lib/` (CodeMirror).
