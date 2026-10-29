# Homepage Site Builder Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a new "Homepage Design" tab to `index.html` that lets a club admin pick a visual theme, show/hide homepage sections, edit section text inline, swap section photos, and preview the result in desktop/mobile — ported from the provided mockup (`Account Information – Friends of Fairview (mockup) (1).html`) into this app's existing tab/panel/save conventions.

**Architecture:** `index.html` is a single-file, no-build, vanilla-JS prototype (no backend calls, no test runner — state lives in JS variables, "Save" just clears a dirty flag). The new tab follows that same pattern: one more `data-tab`/`data-panel` pair, one more self-contained IIFE-scoped block of JS state (`SB` namespace) and CSS (`.sb-` prefixed classes) appended to the existing `<script>`/`<style>` blocks. No new files, no dependencies.

**Tech Stack:** Vanilla HTML/CSS/JS (ES5-leaning, matches existing file), inline SVG placeholder images (ported from the mockup), `contenteditable` for inline text editing, `FileReader` for image swap.

**Spec:** The mockup file `Account Information – Friends of Fairview (mockup) (1).html` (read in full during planning; see conversation) is the design reference. Relevant pieces: the `.site` theme CSS (lines 38–59 of its `<style>`), the SVG placeholder generator (`DEFS`/`W`/`P`/`sky`/`cl`/`blades`/`crowd`/`SC`/`uri`/`SLOTS`), the section content (`SECS`/`secs()`/`e()`/`ph()`/`hdr()`/`render()`), the tools panel (`panel()`/`frame()`), and the template picker (`tcs`/`tplCards()`).

## Global Constraints

- Single file only: all changes go into `index.html`. No new files, no build step, no package manager.
- No backend/network calls — matches the rest of this prototype (confirmed: `grep -n "fetch(\|XMLHttpRequest\|axios\|ajax" index.html` returns nothing but a CDN `<link>`).
- All new CSS classes and JS globals from the mockup MUST be renamed with an `sb-` (CSS) or `SB` (JS namespace object) prefix before porting — the mockup's class names collide with this app's existing classes (confirmed collisions: `.btn`, `.seg`, `.panel`, `.tg`, `.row`, `.g`, `.f` already exist here with different meanings). Never reuse a mockup class name as-is in this file.
- The new tab is `data-tab="sitebuilder"` with panel `data-panel="sitebuilder"`, inserted as a tab button right after "Webpage Detail" (`data-tab="web"`) in the tablist (`index.html:468-473`), and its panel section placed immediately after the `web` panel closes (currently closing `</section>` at `index.html:901` — line numbers will have shifted once Task 1 lands, so locate by `data-panel="web"`/`data-panel="invoice"` instead of the raw number).
- The existing "Club home page description" field (`index.html:757-761`, `data-f="desc"`, `.rte[data-id="desc"]`) is removed entirely, per explicit user decision — Site Builder's hero/about text now owns that content. Its `rteSeed.desc` entry (`index.html:915`) and its save-validation line `if(!strip($('#desc').innerHTML)) flag($('[data-f=desc]'));` (`index.html:1166`) must be removed too, or Save will throw on a missing element.
- Reuse existing `$`/`$$`/`toast`/`markDirty` helpers already defined in the file; do not redefine them.
- The Site Builder's global Save button is the same `#saveBtn`/`.savebar` already in the file (shown for `web`; extend the visibility rule at `index.html:1181` to also show it for `sitebuilder`). Any edit inside Site Builder calls the existing `markDirty()`.
- Verification throughout this plan is manual (open `index.html` in a browser, click through), matching the fact that this repo has no test framework (`package.json`, test runner, or `tests/` directory do not exist here). Each task still ends with a concrete, checkable deliverable — just checked by hand in a browser instead of `pytest`/`jest`.

## Review Focus

- **Switching themes after editing text/images:** a user edits hero text, swaps a photo, then clicks the other template card — edits and swapped images must survive (mockup keeps `S.T` text overrides and `IMG` image overrides independent of `S.tpl`; verify in Task 4).
- **Hiding a section that's mid-edit:** toggling a section off while its contenteditable has focus must not throw or leave a dangling event listener — `render()` fully replaces `#sb-site`'s innerHTML each time, so re-verify no stale references are held after a toggle (Task 2/3).
- **Leaving the Site Builder tab without saving, then returning:** the panel is `hidden`, not destroyed, so `SB` state and the live preview must still reflect prior edits when the tab is reopened (Task 1 lazy-init must only run once, not reset state on every tab click).
- **Removing the old description field breaks Save:** the existing `saveBtn` click handler validates `$('#desc')` directly (`index.html:1166`); deleting the markup without deleting that validation line throws a TypeError and silently breaks the whole Save button for every tab, not just Homepage (Task 1 must remove both together and manually verify Save still works end-to-end after the change).
- **Mobile frame width vs. real mobile viewport:** the device toggle only resizes the inner preview `<div>` width (390px), it does not change the admin page's own responsive layout — confirm this doesn't fight the app's existing `@media(max-width:900px)` rules for `.panel`/`.tab` (Task 3).

---

## File Structure

Everything lives in `index.html`. No files are created.

- **CSS**: new `<style>` rules appended just before the existing `</style>` tag (find it near the top of the file), all scoped under `.sb-` prefixed classes plus a single `.sb-site` theme block (lifted from the mockup's `.site` rules, renamed `.site` → `.sb-site`, `.classic` → `.sb-classic`, `.modern` stays implicit/default as in the mockup).
- **HTML**: 
  - One new `<button class="tab" ...>` in the tablist (`index.html:468-473`).
  - One new `<section class="panel" data-panel="sitebuilder" hidden>` placed after the `web` panel's closing tag, containing: template picker row, note text, and the editor grid (tools column + preview stage column). Mirrors the mockup's `#tpl` / `.ed` / `.tools` / `.stage` / `#frame` / `#site` structure, with every id/class prefixed `sb-`.
  - Deletion: the `<div class="field full" data-f="desc">...</div>` block (`index.html:757-761`) inside `#sec-home`.
- **JS**: one new IIFE-scoped block appended inside the existing top-level `(function(){ ... })()` in the `<script>` tag (so it can use `$`/`$$`/`toast`/`markDirty` directly), containing a single `SB` object holding all Site Builder state and functions (`SB.IMG`, `SB.DEF`, `SB.state` i.e. `{tpl, brand, dev, hid, T}`, `SB.render()`, `SB.panel()`, `SB.frame()`, `SB.tplCards()`, `SB.init()`), plus:
  - Deletion of the `desc:` entry in `rteSeed` (`index.html:915`).
  - Deletion of the `if(!strip($('#desc').innerHTML))...` line (`index.html:1166`).
  - One-line edit to the tab-click handler's savebar visibility check (`index.html:1181`).
  - One `SB.init()` call wired to fire once, the first time the `sitebuilder` tab is clicked (lazy init, per Review Focus above).

---

### Task 1: Tab scaffold, panel shell, and old-field removal

**Files:**
- Modify: `index.html` (tablist ~L468-473, `web` panel's homepage section ~L757-761, `rteSeed` ~L915, save validation ~L1166, tab-click handler ~L1178-1182)

**Interfaces:**
- Produces: the `data-tab="sitebuilder"` / `data-panel="sitebuilder"` pair that Tasks 2-5 fill in; an empty `<div id="sb-tpl"></div>` (template cards mount point), `<div id="sb-tools"></div>` (tools panel mount point), `<div id="sb-frame"></div>` wrapping `<div id="sb-site"></div>` (preview mount point). These exact ids are what later tasks' JS targets with `$('#sb-tpl')` etc.

- [ ] **Step 1: Add the new tab button**

In `index.html`, add `<button class="tab" role="tab" aria-selected="false" data-tab="sitebuilder">Homepage Design</button>` immediately after the `data-tab="web"` button in the tablist.

- [ ] **Step 2: Add the empty panel shell**

Immediately after the `web` panel's closing `</section>`, add:
```html
<section class="panel" data-panel="sitebuilder" hidden>
  <section class="scard">
    <header><span class="si-t"><i class="fa fa-magic" aria-hidden="true"></i></span><h2>Site Builder</h2></header>
    <div class="sb">
      <p class="note">Pick a starting point. Photos, layout and fonts are prebuilt. Click any text on the preview to edit it, and click any photo to swap it.</p>
      <div class="sb-tpl" id="sb-tpl"></div>
      <div class="sb-ed">
        <div class="sb-tools" id="sb-tools"></div>
        <div class="sb-stage">
          <div class="sb-frame" id="sb-frame">
            <div class="sb-chrome"><i></i><i></i><i></i><span>Preview</span></div>
            <div id="sb-site"></div>
          </div>
        </div>
      </div>
    </div>
  </section>
</section>
```

- [ ] **Step 3: Remove the old homepage description field**

Delete the `<div class="field full" data-f="desc">...</div>` block from inside `#sec-home` (the "Homepage" `<section class="card">`).

- [ ] **Step 4: Remove its now-dead references**

Delete the `desc:` key from the `rteSeed` object literal, and delete the line `if(!strip($('#desc').innerHTML)) flag($('[data-f=desc]'));` from the `saveBtn` click handler.

- [ ] **Step 5: Show the savebar for the new tab too**

Change `$('.savebar').style.display = t.dataset.tab==='web' ? '' : 'none';` to `$('.savebar').style.display = (t.dataset.tab==='web'||t.dataset.tab==='sitebuilder') ? '' : 'none';`.

- [ ] **Step 6: Manual verification**

Open `index.html` in a browser. Confirm: a "Homepage Design" tab appears after "Webpage Detail" and shows an empty "Site Builder" card when clicked; the "Homepage" section under Webpage Detail no longer shows the old description field; clicking "Save changes" on any tab still works without a console error (this specifically catches the dead `$('#desc')` reference).

- [ ] **Step 7: Commit**
```bash
git add index.html
git commit -m "Add Homepage Design tab shell, remove old homepage description field"
```

---

### Task 2: Placeholder image library + section content + live render

**Files:**
- Modify: `index.html` (new JS block inside the main IIFE)

**Interfaces:**
- Consumes: `#sb-site` mount point from Task 1.
- Produces: `SB.IMG` (object, slot key → data-URI string, mutable), `SB.DEF` (object, slot key → original default data-URI, immutable reference for "Reset"), `SB.SLOTS` (array of `[key, label, sceneName]`), `SB.state` (object `{tpl:'modern', brand:{modern:'#1f4e79', classic:'#1b2a4e'}, dev:'d', hid:{}, T:{}}`), `SB.render()` (function, no args, rebuilds `#sb-site`'s innerHTML from current state and rewires its own `[data-img]`/`[data-k]` click/input handlers — called by Task 3 and 4 after any state change).

- [ ] **Step 1: Port the placeholder image generator**

Port the mockup's `DEFS`, `W`, `P`, `sky`, `cl`, `blades`, `crowd`, `SC` (scene generators: `yard`, `street`, `crew`, `truck`, `fire`, `dinner`, `map`), and `uri(k)` verbatim as-is into the new JS block (pure SVG-string-building functions, no renaming needed — they don't collide with anything in this file; confirm with `grep -n "function sky\|function cl\b\|function blades\|function crowd\|const SC\|const uri"` before porting, expect no hits).

- [ ] **Step 2: Define slots and seed images**

```js
SB.SLOTS=[['hero','Hero photo','yard'],['about','About photo','crew'],['g1','Gallery 1','street'],['g2','Gallery 2','truck'],['g3','Gallery 3','fire'],['g4','Gallery 4','yard'],['dinner','Donation image','dinner'],['map','Service area map','map']];
SB.IMG={}; SB.DEF={};
SB.SLOTS.forEach(function(s){ SB.DEF[s[0]]=SB.IMG[s[0]]=uri(s[2]); });
```

- [ ] **Step 3: Define section content and state**

Port `SECS` (section key/label pairs), `secs(m)` (returns an object of section-key → function returning that section's HTML, parameterized by `m` = is-modern-theme boolean), `e(k,d,tag)` (contenteditable span/tag helper reading/writing `SB.state.T[k]`), `ph(k,cls)` (photo `<div>` helper reading `SB.IMG[k]`), and `hdr(k,ey,t,p)` (eyebrow/title/paragraph header helper) — same content and copy as the mockup, with `S.T`/`S.hid`/`IMG` references rewritten to `SB.state.T`/`SB.state.hid`/`SB.IMG`.

```js
SB.state={tpl:'modern',brand:{modern:'#1f4e79',classic:'#1b2a4e'},dev:'d',hid:{},T:{}};
```

- [ ] **Step 4: Implement `SB.render()`**

Build `#sb-site`'s innerHTML the way the mockup's `render()` does: set `className` to `'sb-site '+(SB.state.tpl==='classic'?'sb-classic':'')`, set the `--brand` CSS custom property, concatenate the nav + filtered, non-hidden `SECS` + footer, then re-bind every `[data-img]` element's `onclick` to open the file picker for that slot and every `[data-k]` element's `oninput` to write back into `SB.state.T`. Call `SB.panel()` (Task 3) at the end, same as the mockup.

- [ ] **Step 5: Manual verification**

Temporarily call `SB.render()` from `SB.init()` (stubbed to run immediately, not lazily yet) and open the Homepage Design tab. Confirm the full preview renders with placeholder photos, editable text (click a heading, type, click away, text persists on next `SB.render()` call), and no console errors. Toggle a `SB.state.hid['about']=true` from the devtools console and re-render manually to confirm hiding works before Task 3 wires up a UI control for it.

- [ ] **Step 6: Commit**
```bash
git add index.html
git commit -m "Add Site Builder placeholder image library, section content, and render()"
```

---

### Task 3: Tools panel (device, color, section toggles, image replace)

**Files:**
- Modify: `index.html` (new JS block, continuing `SB` namespace; CSS for `.sb-tools`, `.sb-im`, `.sb-dots`, `.sb-seg-dev`, `.sb-tg`, `.sb-frame`, `.sb-stage`, `.sb-chrome`)

**Interfaces:**
- Consumes: `SB.state`, `SB.IMG`, `SB.DEF`, `SB.SLOTS`, `SB.render()` from Task 2; `#sb-tools`, `#sb-frame` mount points from Task 1.
- Produces: `SB.panel()` (function, no args, rebuilds `#sb-tools`'s innerHTML from current state), `SB.frame()` (function, no args, resizes `#sb-frame` to 390px or 100% based on `SB.state.dev` and calls `SB.panel()`), a hidden `<input type="file" accept="image/*">` for image replace wired to whichever slot was last clicked.

- [ ] **Step 1: Implement `SB.panel()`**

Port the mockup's `panel()` body, renaming classes to the `sb-` prefix and reading from `SB.state`/`SB.IMG`/`SB.DEF`/`SB.SLOTS`: a Device segmented toggle (Desktop/Mobile, calling `SB.state.dev=...; SB.frame();`), a brand-color swatch row (`.sb-dots`, 4 colors per theme, calling `SB.state.brand[SB.state.tpl]='...'; SB.render();`), a Sections list (one row per `SECS` entry with a toggle switch calling `SB.state.hid[key]=!SB.state.hid[key]; SB.render();`), and an Images list (one row per `SB.SLOTS` entry with "Replace" and conditionally "Reset" buttons).

- [ ] **Step 2: Implement `SB.frame()`**

Port the mockup's `frame()`: set `#sb-frame`'s width style and toggle a mobile layout class, then call `SB.panel()`.

- [ ] **Step 3: Wire the image file input**

Add `<input type="file" accept="image/*" id="sb-fi" class="hide">` to the panel shell from Task 1. Port the mockup's `pick(k)`/`onchange` pair: clicking an image (in the preview via `data-img`, or "Replace" in the tools list) opens this input; on file selection, read it as a data URL via `FileReader`, assign into `SB.IMG[k]`, and call `SB.render()`.

- [ ] **Step 4: Add the CSS**

Port the mockup's `.tpl`/`.ed`/`.tools`/`.im`/`.dots`/`.seg`/`.stage`/`.frame`/`.chrome` rules, renamed to `.sb-tpl`/`.sb-ed`/`.sb-tools`/`.sb-im`/`.sb-dots`/`.sb-seg-dev`/`.sb-stage`/`.sb-frame`/`.sb-chrome` (do not reuse `.seg` — it already exists in this file with a radio-based structure that would conflict), plus the mobile breakpoint rule (`@media(max-width:900px){.sb-ed{grid-template-columns:1fr}...}`).

- [ ] **Step 5: Manual verification**

On the Homepage Design tab: toggle Desktop/Mobile and confirm the preview frame resizes; click a brand-color swatch and confirm section accents recolor; toggle a section off/on in the Sections list and confirm it disappears/reappears in the preview; click "Replace" on an image slot, pick a local image file, and confirm it appears both in the tools list thumbnail and in the live preview; click "Reset" and confirm it reverts to the placeholder.

- [ ] **Step 6: Commit**
```bash
git add index.html
git commit -m "Add Site Builder tools panel: device, color, section toggles, image replace"
```

---

### Task 4: Template picker

**Files:**
- Modify: `index.html` (new JS block, continuing `SB` namespace; CSS for `.sb-tc`)

**Interfaces:**
- Consumes: `SB.state.tpl`, `SB.render()`, `SB.DEF`, `#sb-tpl` mount point.
- Produces: `SB.tplCards()` (function, no args, rebuilds `#sb-tpl`'s innerHTML from `SB.state.tpl`).

- [ ] **Step 1: Define the two template definitions**

```js
SB.TPL=[['modern','Modern & Community','Warm neutrals, rounded shapes, photo-led. Friendly and neighborly.'],
        ['classic','Classic & Patriotic','Serif headlines, navy and red, centered layouts. Traditional and formal.']];
```

- [ ] **Step 2: Implement `SB.tplCards()`**

Port the mockup's `tplCards()`: render one `.sb-tc` card per `SB.TPL` entry (thumbnail using `SB.DEF.about` for modern / `SB.DEF.hero` for classic, title, description), highlighting the one matching `SB.state.tpl`. Clicking a card sets `SB.state.tpl`, re-renders the card list, calls `SB.render()`, and shows a `toast(...)` confirming the switch while noting edits are kept.

- [ ] **Step 3: Add the CSS**

Port the mockup's `.tpl`/`.tc` rules as `.sb-tc` (the `.sb-tpl` grid container was already added in Task 1/3's CSS step — if not, add it here).

- [ ] **Step 4: Manual verification**

On the Homepage Design tab: before switching templates, edit a heading's text and swap one photo. Click the other template card. Confirm: the theme visibly changes (fonts/colors/layout), the edited text and swapped photo are still present, and a toast confirms the switch. Switch back and confirm the same holds in reverse.

- [ ] **Step 5: Commit**
```bash
git add index.html
git commit -m "Add Site Builder template picker"
```

---

### Task 5: Lazy init wiring and end-to-end pass

**Files:**
- Modify: `index.html` (tab-click handler ~L1178-1182, or wherever Task 1 left it)

**Interfaces:**
- Consumes: `SB.init()` is defined here for the first time, calling `SB.tplCards(); SB.frame(); SB.render();` in that order (matching the mockup's own bootstrap sequence) guarded by a `SB.inited` flag so it only runs once.

- [ ] **Step 1: Implement lazy init**

```js
SB.inited=false;
SB.init=function(){ if(SB.inited) return; SB.inited=true; SB.tplCards(); SB.frame(); SB.render(); };
```

- [ ] **Step 2: Call it from the tab-click handler**

In the existing `$$('.tab').forEach(function(t){ t.addEventListener('click', ... ) })` handler, add `if(t.dataset.tab==='sitebuilder') SB.init();` right after the panel-visibility toggle line.

- [ ] **Step 3: Manual end-to-end verification**

Fresh page load → click "Profile" tab → click "Homepage Design" tab: confirm the Site Builder renders fully on first click (not blank). Click to another tab and back: confirm prior edits (text/images/toggles/theme) are still there, and `SB.init()`'s body does not re-run (add a temporary `console.log` to confirm, then remove it). Make an edit, click "Save changes": confirm the save status text updates and no console error appears. Reload the page fully and confirm Site Builder resets to its defaults (expected, since there's no persistence in this prototype — matches every other tab's behavior here).

- [ ] **Step 4: Remove any temporary debug logging left over from Task 2/5 verification steps.**

