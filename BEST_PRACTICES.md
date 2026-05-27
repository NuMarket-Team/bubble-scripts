# Best Practices: Figma-to-Code + Bubble HTML Elements

How we build front-end components at NuMarket: pull specs from Figma, write self-contained HTML elements, and keep all logic out of Bubble's dynamic expressions.

> **Companion reference:** [DATA_SCHEMA.md](DATA_SCHEMA.md) — the auto-generated reference for all 73 Bubble data types, field names, and the Data API ↔ Toolbox key translation rules. Consult it whenever you need a field name.

---

## Part 1 — Working with Figma MCP

### Setup

Figma Dev Mode MCP is registered in Claude Code as a remote server (`https://mcp.figma.com/mcp`). Confirm with `claude mcp list`. If it shows "needs authentication", run `/mcp` → figma → Authenticate.

### Pulling design specs

1. Get the Figma URL from the designer. It looks like:
   ```
   https://www.figma.com/design/<fileKey>/<fileName>?node-id=<nodeId>&m=dev
   ```
2. Parse the URL: extract `fileKey` and `nodeId` (convert `-` to `:` in nodeId).
3. Call `get_design_context` with `fileKey` and `nodeId` — this returns code suggestions, a screenshot, and contextual hints (tokens, component docs, annotations).
4. Call `get_screenshot` if you need a standalone visual reference at a specific resolution.
5. Call `get_metadata` for structural info (auto-layout direction, padding, gaps, constraints).

### What you get back

- **Code Connect snippets** — if the team has mapped Figma components to codebase components, use the mapped component directly.
- **Design tokens as CSS variables** — map them to the project's token system (NuMarket brand colours in `brand.md`).
- **Raw values** (hex colours, absolute px) — the design is loosely structured. Use the screenshot as the source of truth and translate to your CSS conventions.
- **Annotations** — follow any notes or constraints from the designer.

### Common pitfalls

| Problem | Fix |
|---|---|
| `get_design_context` returns 162K+ chars | The node is too broad. Zoom into a child node or use `get_screenshot` for visual reference and `get_metadata` for layout values. |
| Image dimension limit error (`>2000px`) | Figma screenshot is too large for the conversation. Use `get_metadata` (text-only) instead, or ask for specific specs rather than the full screenshot. |
| Padding/gap values look wrong | Figma auto-layout reports padding per-side (`paddingLeft`, `paddingTop`, etc.) and `itemSpacing` for gap. Always verify against the screenshot — nested frames can double up padding. |
| Colours don't match brand | Check whether the Figma file uses design tokens or raw hex. If raw hex, cross-reference `brand.md` for the canonical NuMarket palette. |

### Workflow summary

```
Figma URL → get_design_context / get_metadata → read current HTML file
→ edit CSS/HTML to match specs → verify with preview_inspect + preview_screenshot
```

Always **adapt** the Figma output to the project stack. The returned code is React+Tailwind by default — we write vanilla HTML/CSS for Bubble elements.

---

## Part 2 — Writing Front-End Code for Bubble HTML Elements

This part is the complete reference for building NuMarket HTML elements: file structure, HTML/CSS/JS conventions, fetch patterns, lifecycle, and debugging.

### The core principle

**Push all logic into the HTML element's JavaScript. Bubble does nothing except host the element.**

Bubble dynamic expressions (`:filtered`, `:sorted`, `:first item's...`) are:
- Invisible to git — they live only in the Bubble editor
- Impossible to diff, grep, or debug
- Brittle — a renamed field breaks silently with no compile error
- Untransferable — you can't copy an element between pages without rewiring every expression

A self-contained HTML element can be version-controlled, copy-pasted between pages, and debugged with `console.log`.

### File anatomy

Every NuMarket HTML element is a single `.html` file with three blocks, in this order:

```
┌─────────────────────────────────────┐
│  <!-- HTML markup -->                │
│  Empty shells with stable IDs        │
│  Semantic tags, no hardcoded data    │
├─────────────────────────────────────┤
│  <style>                             │
│  Scoped CSS (BEM + nm- prefix)       │
│  All breakpoints via @media queries  │
│  </style>                            │
├─────────────────────────────────────┤
│  <script>                            │
│  IIFE wrapper                        │
│  bubbleApiRoot() helper              │
│  CONFIG block                        │
│  Helpers (el, esc, formatDate)       │
│  fetch + render                      │
│  Event handlers                      │
│  Lifecycle bindings                  │
│  </script>                           │
└─────────────────────────────────────┘
```

The same file works in dev, version-test, and live. No Bubble wiring, no token replacement, no per-environment configuration.

### HTML conventions

- **Lead with semantic tags** — `<header>`, `<article>`, `<section>`, `<aside>`, `<nav>`, `<main>` — not generic `<div>` everywhere. Better SEO and accessibility for free.
- **Stable, prefixed IDs** — every node JS will touch needs an `id`. Prefix per-element (`nmh-title`, `nmh-image` for blog header; `ab-title`, `ab-send-btn` for admin broadcast) so multiple HTML elements on one page can't collide.
- **Empty shells, not placeholders** — text starts empty (`<h1 id="nmh-title"></h1>`), images start with `src=""`. JS fills them after fetch. Hardcoding "Loading..." or example text causes stale content to flash before the real data renders.
- **Accessibility**:
  - `alt=""` on decorative images, descriptive `alt` on content images
  - `aria-label` on icon-only buttons (`<button aria-label="Share">`)
  - `aria-live="polite"` on dynamic regions (errors, toast messages)
  - Heading hierarchy without skipping (`h1` → `h2` → `h3`) — don't use `<h3>` because `<h2>` is too big; restyle the `<h2>` instead
- **No inline styles** — all styling in the `<style>` block
- **No inline event handlers** — bind via `addEventListener` in JS (never `onclick=""`)
- **Performance hints on images**:
  - Above-the-fold: `fetchpriority="high" decoding="async"`
  - Below-the-fold: `loading="lazy"`

```html
<header id="nm-blog-header" class="nm-blog-header">
  <div class="nm-blog-header__hero">
    <img id="nmh-image" class="nm-blog-header__image"
         src="" alt="" fetchpriority="high" decoding="async" />
  </div>
  <div class="nm-blog-header__inner">
    <h1 id="nmh-title" class="nm-blog-header__title"></h1>
    <p  id="nmh-subhead" class="nm-blog-header__subhead"></p>
    <ul id="nmh-tags" class="nm-blog-header__tags" aria-label="Tags"></ul>
  </div>
</header>
```

### CSS conventions

#### BEM with `nm-` prefix

Block / element / modifier naming, prefixed to avoid collisions with Bubble's own classes or other elements on the page:

```
.nm-blog-header                   /* block */
.nm-blog-header__inner            /* element */
.nm-blog-header__title            /* element */
.nm-blog-header__inner--featured  /* modifier */
```

#### Scope every selector to your block root

Bubble pages have many overlapping styles. Scope to your root or styles will leak:

```css
.nm-blog-header h1 { ... }   /* yes — only inside this element */
h1 { ... }                    /* no — bleeds across the whole page */
```

#### Reset within scope

Don't trust the host page's box-sizing or margin defaults. Reset locally:

```css
.nm-blog-header *,
.nm-blog-header *::before,
.nm-blog-header *::after {
  box-sizing: border-box;
}
.nm-blog-header h1,
.nm-blog-header h2,
.nm-blog-header p,
.nm-blog-header ul {
  margin: 0;
  padding: 0;
}
```

#### CSS custom properties for design tokens

Define brand tokens on the block root so they cascade only inside your element:

```css
.nm-blog-header {
  --nm-color-text: #1a1a1a;
  --nm-color-bg:   #f8f5f0;
  --nm-spacing-md: 24px;
  --nm-spacing-lg: 40px;
}
.nm-blog-header__title {
  color: var(--nm-color-text);
  margin-bottom: var(--nm-spacing-md);
}
```

Brand values live in `brand.md` — translate them into custom properties at the top of each element so the file is self-documenting.

#### Fonts

NuMarket uses Inter (UI/body) and Recoleta (headlines, **regular weight only** — bold Recoleta is reserved for the logo). Bubble loads both app-wide; reference by family:

```css
.nm-blog-header__title {
  font-family: 'Recoleta', Georgia, serif;
  font-weight: 400;
}
.nm-blog-header__body {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;
}
```

### JavaScript conventions

#### IIFE wrapping

Wrap all JS in an immediately-invoked function expression so locals don't leak to the global scope and can't collide with other elements on the page:

```js
<script>
(function () {
  'use strict';

  // CONFIG, helpers, fetch, render, events…

})();
</script>
```

The only deliberate globals are namespaced (`window.NM_CONTEXT`, `window.NM_BLOG_DEBUG`) — see Part 3 and the Debugging section below.

#### `var` by default

Use `var` for top-level declarations. Bubble HTML elements run in the host page's context, and you don't control which browser version Bubble serves. `let`/`const` are fine inside loops and conditionals — but consistency matters more than micro-optimization, and the existing codebase uses `var`.

#### Helpers at the top

After CONFIG, define small reusable helpers. These two are universal — copy them into every new element:

```js
function el(id) { return document.getElementById(id); }

function esc(s) {
  return String(s).replace(/[&<>"']/g, function (c) {
    return {'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[c];
  });
}
```

Add specifics as needed:

```js
function formatDate(iso) {
  if (!iso) return '';
  try {
    return new Date(iso).toLocaleDateString('en-US', {
      year: 'numeric', month: 'long', day: 'numeric'
    });
  } catch (e) { return iso; }
}
```

#### XSS prevention — `textContent` by default

API responses can contain user input (titles, names, tag names, comments). Default to safe insertion:

```js
el('nmh-title').textContent = post.Title;          // safe — escapes automatically
el('nmh-author').textContent = author.First + ' ' + author.Last;
```

When you must use `innerHTML` (template-style insertion of multiple nested elements), escape every interpolated value:

```js
el('nmh-tags').innerHTML = tags.map(function (t) {
  return '<li><a href="' + esc(tagUrl(t.name)) + '">'
       +   esc(t.name)
       + '</a></li>';
}).join('');
```

Treat all API data as untrusted — even from your own backend. Escape at render time, even if you sanitize at write time.

#### Fetch — Promise chains, consistent style

The blog code uses `.then()` chains. Pick one style per file and stay consistent:

```js
fetch(CONFIG.BUBBLE_API + '/obj/blogpost', { cache: 'no-store' })
  .then(function (r) { return r.json(); })
  .then(function (data) {
    if (!data.response || !data.response.results.length) {
      throw new Error('Not found');
    }
    return data.response.results[0];
  })
  .then(render)
  .catch(function (e) {
    console.error('[nm-blog-header]', e);
    showError(e);
  });
```

Always include `{ cache: 'no-store' }` for Data API reads — Bubble's `Cache-Control` headers vary, and stale data is harder to debug than slower fetches.

#### State as a single object

For interactive elements with multiple flags, keep a single `S` object near the top of the IIFE rather than scattering booleans:

```js
var S = {
  post: null,
  loading: false,
  submitting: false,
  selected: new Set(),
};
```

Easier to log (`console.log(S)`), easier to reason about, easier to reset.

### The CONFIG block

```js
var CONFIG = {
  // Bubble Data API root — derived at runtime, NOT hardcoded.
  // See "API version detection" below.
  BUBBLE_API: bubbleApiRoot(),

  // Data type API key (Settings → API → Data API)
  DATA_TYPE: 'blogpost',

  // Field map: internal name → Data API display name
  // (See "Data API field names vs Toolbox field names" below.)
  FIELDS: {
    slug:   'Slug',
    title:  'Title',
    image:  'header image',
    author: 'author',
  },

  // Slug extraction from URL
  SLUG_PATH_SEGMENT: -1,  // last segment of /blog/my-post
};
```

### API version detection

**Never hardcode `BUBBLE_API: 'https://numarket.co/api/1.1'`.** Bubble has multiple deployed versions (live, version-test, version-<branch>), and each has a separate Data API endpoint scoped to that version's data. Hardcoding live means version-test pages will silently read live data — confusing in dev and dangerous in QA.

Detect the version from the current URL path:

```js
function bubbleApiRoot() {
  var path = window.location.pathname;
  // Match a version segment: /version-test/, /version-abc123/, etc.
  var match = path.match(/^\/(version-[^\/]+)/);
  var versionPrefix = match ? '/' + match[1] : '';
  return window.location.origin + versionPrefix + '/api/1.1';
}
```

Behaviour:
- `numarket.co/blog/coffee-shop` → `numarket.co/api/1.1` (live)
- `numarket.co/version-test/blog/coffee-shop` → `numarket.co/version-test/api/1.1`
- `numarket.co/version-feat-x/admin` → `numarket.co/version-feat-x/api/1.1`

Drop this helper into every element above the CONFIG block. The same HTML works everywhere with no swap needed when promoting to live.

> ⚠️ Custom backend URLs (e.g. `leads.numarket.info`) don't have Bubble version prefixes — they're separate services. Keep those hardcoded in CONFIG, but consider adding a flag for `version-test` vs live (e.g. swap to a staging endpoint when running in `version-test`).

### Fetching from the Data API

#### The base fetch

Read a single record by constraint (typical for slug-based pages):

```js
function fetchPost(slug) {
  var constraints = [
    { key: CONFIG.FIELDS.slug, constraint_type: 'equals', value: slug }
  ];
  var url = CONFIG.BUBBLE_API + '/obj/' + CONFIG.DATA_TYPE
    + '?constraints=' + encodeURIComponent(JSON.stringify(constraints));

  return fetch(url, { cache: 'no-store' })
    .then(function (r) { return r.json(); })
    .then(function (data) {
      if (!data.response || !data.response.results || !data.response.results.length) {
        throw new Error('Post not found: ' + slug);
      }
      return data.response.results[0];
    });
}
```

Read by unique ID (when you have an `_id` from a parent fetch):

```js
fetch(CONFIG.BUBBLE_API + '/obj/user/' + authorId, { cache: 'no-store' })
```

Read a list with constraints + sorting + pagination:

```js
var url = CONFIG.BUBBLE_API + '/obj/' + CONFIG.DATA_TYPE
  + '?constraints=' + encodeURIComponent(JSON.stringify(constraints))
  + '&sort_field=' + encodeURIComponent('Created Date')
  + '&descending=true'
  + '&limit=20';
```

#### Constraint types

The Bubble Data API accepts these `constraint_type` values:

| Type | Use for |
|---|---|
| `equals` / `not equal` | Exact text/number/yes-no match |
| `is_empty` / `is_not_empty` | Field has any value (no `value` needed) |
| `text contains` / `not text contains` | Substring match (case-insensitive) |
| `greater than` / `less than` | Numeric or date comparison |
| `in` / `not in` | Value is one of an array |
| `geographic_search` | Within radius of a point |

```js
[
  { key: 'Slug', constraint_type: 'equals', value: 'coffee-shop' },
  { key: 'published date', constraint_type: 'less than', value: '2026-01-01' },
  { key: '_id', constraint_type: 'in', value: ['id1', 'id2', 'id3'] },
]
```

#### Related objects — chained fetches

When a field returns an `_id` reference (e.g. `author` → User ID), make a second fetch:

```js
fetchPost(slug)
  .then(function (post) {
    el('nmh-title').textContent = post.Title;

    var authorId = post[CONFIG.FIELDS.author];
    if (!authorId) return;

    return fetch(CONFIG.BUBBLE_API + '/obj/user/' + authorId, { cache: 'no-store' })
      .then(function (r) { return r.json(); })
      .then(function (data) {
        var author = data.response;
        el('nmh-author').textContent = author.First + ' ' + author.Last;
      });
  });
```

For lists of IDs (e.g. `tags` → array of blogtags IDs), use a single constraint query rather than N parallel fetches:

```js
fetch(CONFIG.BUBBLE_API + '/obj/blogtags?constraints='
  + encodeURIComponent(JSON.stringify([
    { key: '_id', constraint_type: 'in', value: tagIds }
  ])))
```

#### Writes

The public Data API doesn't accept writes from a browser without an API token. For writes, hit a Bubble Backend Workflow exposed as an API endpoint or a custom backend:

```js
fetch('/api/1.1/wf/leadCapture', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ email: email, first_name: name })
})
  .then(function (r) {
    if (!r.ok) throw new Error('Submit failed');
    return r.json().catch(function () { return {}; });
  })
  .then(showSuccess)
  .catch(showError);
```

The session cookie authenticates the user; the workflow's server-side `Only when` guard checks role. See Part 4 for the full security model.

### Loading and error states

Three failure modes to handle, none of which should leave the user staring at empty shells:

1. **Network error** — fetch rejects (`.catch`)
2. **Auth blocked** — 200 response but `data.response.results` is empty
3. **Not found** — 200 response but no record matches

```js
function fetchAndRender() {
  el('nm-blog-header').classList.add('is-loading');

  fetchPost(slugFromUrl())
    .then(render)
    .catch(function (e) {
      console.error('[nm-blog-header]', e);
      showError(e.message);
    })
    .finally(function () {
      el('nm-blog-header').classList.remove('is-loading');
    });
}

function showError(msg) {
  el('nm-blog-header').classList.add('is-error');
  el('nmh-title').textContent = msg || 'Unable to load this page.';
  el('nmh-image').style.display = 'none';
}
```

#### Loading state — skeleton or hide

Don't show empty shells while fetching. Either render a skeleton or hide the inner content until ready:

```css
.nm-blog-header.is-loading .nm-blog-header__inner {
  visibility: hidden;
}
.nm-blog-header.is-loading .nm-blog-header__skeleton {
  display: block;
}
```

The first paint without data is the worst-looking version of your element. Design for it explicitly.

### Event binding

Inline `onclick=""` handlers escape the IIFE scope and can't access CONFIG/state — always bind via `addEventListener`.

```js
el('nmh-share-btn').addEventListener('click', function () {
  navigator.clipboard.writeText(window.location.href);
  showToast('Link copied');
});
```

For form submissions, `preventDefault()` then handle via fetch. Guard against double-submit with a state flag:

```js
el('nmh-lead-form').addEventListener('submit', function (e) {
  e.preventDefault();
  if (S.submitting) return;
  S.submitting = true;

  submitLead()
    .then(showSuccess)
    .catch(showError)
    .finally(function () { S.submitting = false; });
});
```

For event delegation on dynamic content (e.g. clicks on tag chips that JS rendered), bind to the parent and check `e.target`:

```js
el('nmh-tags').addEventListener('click', function (e) {
  var link = e.target.closest('.nm-blog-header__tag');
  if (!link) return;
  // handle the click
});
```

### Bubble's SPA-like navigation — the gotcha

**Bubble pages do not always fully reload between navigations.** Bubble swaps content via JS for certain transitions; the host page's `<script>` blocks may re-execute or may not, depending on Bubble's internals. Globals, intervals, and event listeners can persist across "page changes" within the same browser session.

Implications:
- An element that fetches once on load can show stale data after navigation
- Listeners attached to `window`/`document` accumulate
- `setInterval` timers don't clear themselves

#### Re-fetch on URL change

```js
function loadForCurrentUrl() {
  fetchPost(slugFromUrl()).then(render).catch(showError);
}

// Initial load
loadForCurrentUrl();

// Browser back/forward
window.addEventListener('popstate', loadForCurrentUrl);

// Bubble sometimes pushState's without firing popstate.
// Poll path changes as a backstop.
var lastPath = window.location.pathname;
setInterval(function () {
  if (window.location.pathname !== lastPath) {
    lastPath = window.location.pathname;
    loadForCurrentUrl();
  }
}, 500);
```

#### Don't double-bind on re-execution

If the script re-runs (Bubble navigation sometimes triggers this), guard global listener attachment with a sentinel:

```js
if (!window.__nmBlogHeaderInit) {
  window.addEventListener('popstate', loadForCurrentUrl);
  window.__nmBlogHeaderInit = true;
}
```

Otherwise listeners stack on every script execution and the page slows down over time.

### DOM lifecycle

Bubble usually has the DOM ready by the time HTML element scripts run, but defend against the alternative:

```js
function init() {
  // CONFIG, helpers, fetch, render, events
}

if (document.readyState === 'loading') {
  document.addEventListener('DOMContentLoaded', init);
} else {
  init();
}
```

### Debugging

Bubble's editor has no JS debugger. Your tools are:

#### Console logging with prefixes

Always prefix logs with the element name so you can filter:

```js
console.log('[nm-blog-header]', 'fetched post:', post);
console.warn('[nm-blog-header]', 'no tags returned');
console.error('[nm-blog-header]', err);
```

#### DevTools tabs

- **Network** — every Data API request and response. Look at the `?constraints=...` query string and the response body. This is where most data bugs hide.
- **Elements** — see Bubble's wrapper around your HTML element. Bubble injects its own div with its own classes and constraints; styles cascade through it.
- **Console** — paste fetch calls directly to test queries:
  ```js
  fetch('/api/1.1/obj/blogpost?constraints=' + encodeURIComponent(JSON.stringify([
    { key: 'Slug', constraint_type: 'equals', value: 'coffee-shop-funding' }
  ]))).then(r => r.json()).then(console.log)
  ```

#### A debug global

Expose internals when a query param is set, so you can inspect from the console without leaving them on in production:

```js
if (window.location.search.indexOf('nm_debug=1') !== -1) {
  window.NM_BLOG_DEBUG = { CONFIG: CONFIG, S: S, fetchPost: fetchPost };
  console.log('[nm-blog-header] debug mode — window.NM_BLOG_DEBUG available');
}
```

Then in the console: `NM_BLOG_DEBUG.fetchPost('test-slug').then(console.log)`.

### One-time Bubble setup

Before any Data API element works:

1. **Settings → API → Enable Data API**
2. **Expose the data type** (e.g. blogpost, blogtags, user)
3. **Privacy rules** — make required fields publicly readable for the types you exposed
4. Match `CONFIG` field names to Bubble's **display names** exactly

### Data API field names vs Toolbox field names

This is the most common source of bugs:

| Context | Field name format | Example |
|---|---|---|
| **Data API** (HTML fetch) | Display name | `response['header image']` |
| **Toolbox** (server-side script) | API ID with type suffix | `await obj.get('header_image_image')` |
| **Bubble expressions** | Display name | `Current Page Blog Post's header image` |

The [DATA_SCHEMA.md](DATA_SCHEMA.md) file documents both formats for every field.

### When Toolbox server-side scripts are better than Data API fetches

| Use case | Approach |
|---|---|
| **Read-only display** (blog header, cards, analytics dashboards) | HTML element + Data API fetch |
| **Write operations** (form submit, lead capture, status update) | Bubble workflow with Toolbox server-side script |
| **Complex computation** (campaign analytics stitching, cumulative sums) | Toolbox server-side script returning flat outputs |
| **Multi-step logic with side effects** | Bubble workflow calling multiple Toolbox scripts in sequence |

### Toolbox script conventions

```
- "Script is Async Function" ON
- "Multiple Outputs" ON — each output slot typed explicitly
- Pass Things/Thing lists via Thing/Thing list slots (not Keys and Values)
- Field access: await obj.get('fieldname_typesuffix')
  - e.g. path_text, amount_number, header_image_image
  - Built-in fields: .get('Created Date'), .get('Modified Date')
- Thing lists: materialize first
    var len = await raw.length();
    var items = await raw.get(0, len);
  Then index into items[]
- Return flat outputs (output1, output2...) — text or number, not JSON
- Bubble-side "Make changes to thing" writes outputs to fields
```

### When Bubble dynamic expressions are acceptable

- Simple `:first item` lookups to wire a Thing into a Toolbox script input
- List filtering as the *input* to a script (pre-filtering before the script runs)
- Trigger conditions — one-line `Only when` comparisons
- Conditional visibility — `This element is visible when Current Page's X is Y`

If the expression is longer than one line or chains two or more operators, push it into a script.

---

## Part 3 — Identifying the Right Record (When the URL Doesn't Tell You)

The blog pattern uses `window.location.pathname` to extract a slug. Most other pages don't have that luxury. Pick the cleanest pattern that fits.

### Pattern ranking (cleanest → last resort)

#### 1. URL slug — preferred for SEO-relevant pages
Used by: blog posts, business profiles, campaign pages.
```
URL: numarket.co/blog/coffee-shop-funding
JS:  window.location.pathname.split('/').filter(Boolean).pop()
```

#### 2. URL query parameter — preferred for app-style pages
Used by: dashboards, modals, filtered views, Bubble's "Go to page → send data".
```
URL: numarket.co/dashboard?campaign_id=abc123
JS:  new URLSearchParams(window.location.search).get('campaign_id')
```
Bubble's "Go to page" action with "Send data" appends the Thing's unique id to the URL automatically. Configure the destination page to read it.

#### 3. Page-scoped window global — the "context bridge"
For Bubble pages where the Thing is set via "Page Type" (Bubble routes the Thing into the page invisibly — it's not in the URL).

Add ONE tiny "bridge" element at the top of the page. It's the single point in the page that uses a Bubble dynamic expression. Every other HTML element on the page reads from `window.NM_CONTEXT` and stays clean.

```html
<!-- nm-context-bridge.html — placed once at the top of the page -->
<script>
  window.NM_CONTEXT = {
    thing_id:   'BUBBLE_DYNAMIC_EXPRESSION_HERE',  // e.g. Current Page Business's unique id
    thing_type: 'business',
    user_role:  'BUBBLE_DYNAMIC_EXPRESSION_HERE',  // e.g. Current User's role — used for UI-only signals
  };
</script>
```

```js
// In any other element on the page:
var ctx = window.NM_CONTEXT || {};
fetch(CONFIG.BUBBLE_API + '/obj/' + ctx.thing_type + '/' + ctx.thing_id, ...)
```

The bridge confines Bubble wiring to one tiny file. Complex elements stay portable.

> ⚠️ Never trust `ctx.user_role` for security. It's a UI signal only (e.g. "show admin button"). Server-side Privacy Rules are the actual boundary — see Part 4.

#### 4. localStorage / sessionStorage — for cross-page state
Used by: multi-step flows, "remember last viewed", saved drafts. Set on one page, read on the next.

```js
// Set after a fetch on page A
sessionStorage.setItem('last_campaign_id', id);

// Read on page B
var id = sessionStorage.getItem('last_campaign_id');
```

Don't store sensitive data here — it's accessible to any script on the same origin.

#### 5. Last resort: one inline Bubble token
If you cannot route the ID any other way, accept ONE token. Document it:
```html
<!-- TOKEN: Bubble dynamic expression — needed because [reason: e.g. Bubble's
     repeating group Cell context is not exposed to URL or Page Type] -->
<script>
  var THING_ID = '[BUBBLE_THING_ID]';
</script>
```
Use this only when 1–4 are genuinely impossible. Repeating-group cells are the typical case.

### Rule of thumb

- One Thing per page, in URL → pattern 1 or 2
- One Thing per page, not in URL → pattern 3 (context bridge)
- Many Things per page (lists, RGs) → pattern 5 per cell, or refactor the cell to fetch its own data via the API using a known constraint

---

## Part 4 — Security Model

### The threat model

**HTML element source code is public.** Anyone can View Source on any page that includes the element. That means:
- No API keys, tokens, or passwords belong in the file
- No "admin = true" flags belong in JS — they're trivially bypassed
- No sensitive data should be hardcoded as fallback values

### Privacy Rules are the security boundary

The Bubble Data API is **public by design**. You don't secure it by hiding it — you secure it via Privacy Rules. For every exposed data type:

1. Define rules per-role (`Everyone`, `This User is logged in`, `Current User's role is admin`, custom roles)
2. Per rule, choose which fields are readable and whether records are findable in searches
3. Default to deny — only enable what the front-end actually needs

If the API call is made by a user without permission, Bubble returns the result filtered to what they're allowed to see (often empty). Always handle the empty case gracefully.

### How authentication works for Data API calls

**The browser already has the auth token.** When any user logs into Bubble — admin or otherwise — Bubble sets a session cookie on the browser. That cookie *is* the auth token. There's nothing to extract, store, or forward manually.

For same-origin fetches, the browser sends the session cookie automatically. Bubble's Data API reads it, resolves `Current User`, and applies Privacy Rules. No `Authorization` header needed — no token handling in code.

```js
// Same-origin call from numarket.co → numarket.co/api/1.1/...
// Browser sends session cookie automatically.
// Bubble identifies Current User → enforces Privacy Rules → returns
// only what Current User is allowed to read.
fetch('/api/1.1/obj/business/' + id, { cache: 'no-store' })
  .then(function(r) { return r.json(); })
  .then(function(data) {
    if (!data.response) return showAccessDenied();
    render(data.response);
  });
```

For cross-origin (e.g. a subdomain like `app.numarket.co` calling `numarket.co/api/...`), you must opt into sending cookies:
```js
fetch(CONFIG.BUBBLE_API + '/obj/...', {
  cache: 'no-store',
  credentials: 'include',  // send cookies cross-origin
})
```
Cross-origin also requires CORS allowlisting in Bubble settings (Settings → API → CORS).

#### Session cookie naming (for debugging)

If you ever need to confirm the session is in place — e.g. while debugging a 401 — Bubble's session cookies on `numarket.co` follow this format:

```
numarket_u1_<versionID>main = <user_unique_id>
```

- `numarket` — app name
- `u1` — user-session slot (Bubble's internal naming)
- `<versionID>` — short ID per Bubble version (e.g. `539ks` = live, `53ak6` = version-test)
- `main` — primary user session suffix
- value — the user's Bubble unique ID

You'll have one cookie per Bubble version you've logged into. Each version's Data API only reads its matching cookie — which is why `bubbleApiRoot()` and the per-version session model fit together: when an HTML element on `version-test` calls `version-test/api/1.1/...`, the browser sends the version-test cookie automatically and Bubble resolves Current User in that version's data context.

Inspect via DevTools → Application → Cookies → `https://numarket.co`.

**Treat as credentials.** Never read `document.cookie` and forward those values, never log them, never store them in `localStorage`, never pass them in URL params or POST bodies. Bubble manages the cookie lifecycle; let it.

### Admin elements — concrete pattern

Because admin users have the same auth mechanism as any logged-in user (session cookie), an admin element looks identical to any other authenticated element. The differentiation is at the **Privacy Rule** layer, not in the fetch code.

```js
// admin-only element: fetch internal_dashboard records
// Privacy Rule on internal_dashboard: "Current User's role is admin"
//   → admin sees data
//   → non-admin sees empty results
//   → logged-out sees empty results
fetch('/api/1.1/obj/internal_dashboard', { cache: 'no-store' })
  .then(function(r) { return r.json(); })
  .then(function(data) {
    var results = (data.response && data.response.results) || [];
    if (!results.length) {
      // Privacy Rules blocked us, OR there's genuinely no data.
      // Don't render an admin shell — admins seeing "empty" by mistake
      // is a UX problem, but non-admins seeing the shell is a security problem.
      return showAccessDenied();
    }
    renderAdminPanel(results);
  });
```

Three things to remember:
1. **You don't pass a token in code.** The session cookie does it. If you find yourself looking for a token to embed, stop — that's the wrong mechanism.
2. **Privacy Rules do the gating.** The fetch code is identical for admin and non-admin elements. The Privacy Rule on the data type is what makes one admin-only.
3. **"Log in as" / impersonation gotcha.** If an admin uses Bubble's "Run as" feature to impersonate another user, the session cookie reflects the impersonated user — NOT the admin. Privacy Rules will treat them as that user. That's usually the desired behaviour for testing, but if you need to detect impersonation, do it server-side (Backend Workflow), not in the HTML element.

### The three access tiers

| Tier | Privacy Rule | When to use |
|---|---|---|
| **Public** | `Everyone (including not logged in)` | Blog, public marketing, public business profiles |
| **Logged-in** | `This User is logged in` | Member dashboards, account settings, contributor views |
| **Admin** | `Current User's role is admin` (or specific role) | Admin tools, broadcast composer, internal analytics, deal-form internals |

### Admin-only elements: defence in depth

Admin pages need protection at three layers — never rely on just one:

1. **Bubble page-level access control** — set the page to redirect non-admins. Stops the page from rendering at all.
2. **Element-level conditional visibility** — hide admin-only elements when `Current User's role is not admin`. Cosmetic but useful.
3. **Privacy Rules on the data type** — the actual boundary. Even if the page or element leaks, the Data API rejects the request.

If the only thing standing between an attacker and admin data is a Bubble visibility condition, you're not secure. Always set Privacy Rules.

### Writes — never via the public Data API

The public Data API doesn't accept writes from a browser without an API token, and tokens can't go in client code. So writes use one of these:

| Pattern | When to use |
|---|---|
| **Bubble Backend Workflow exposed as API** | Authenticated user actions (create lead, update profile). Workflow has a server-side `Only when Current User is logged in / is admin` guard. |
| **Custom backend endpoint** (e.g. `leads.numarket.info`) | Public form submissions where you want explicit rate limiting + validation outside Bubble. |
| **Toolbox server-side script in a Bubble workflow** | Triggered by a button click in Bubble UI. Runs server-side with Bubble's user context — Bubble enforces auth before the workflow runs. |

Admin-only writes follow the same patterns, but the workflow's "Only when" guard checks `Current User's role is admin`. **Never trust a client-side admin check.** The server must verify the role on every request.

### CORS

| Setup | CORS handling |
|---|---|
| Same domain (`numarket.co` → `numarket.co/api/...`) | No issue. Cookies sent by default. |
| Subdomain or different domain | Add origin to Bubble's allowlist (Settings → API → CORS). Use `credentials: 'include'` in fetch. |
| Custom backend (`leads.numarket.info`) | Allowlist `numarket.co` on the backend. |

Bubble's Data API allows reads from any origin by default — that's fine because Privacy Rules are the boundary, not the origin.

> See **Part 6 → Security** for the deployment checklist.

---

## Part 5 — Responsive / Breakpoint Logic in HTML Elements

### Approach: CSS media queries, not Bubble responsive settings

Bubble's responsive engine doesn't apply inside HTML elements. All breakpoint logic must be CSS:

```css
/* Desktop — default */
.nm-blog-header__inner {
  display: flex;
  gap: 40px;
  padding: 40px 80px;
}

/* Tablet */
@media (max-width: 991px) {
  .nm-blog-header__inner {
    padding: 32px 40px;
    gap: 24px;
  }
}

/* Mobile */
@media (max-width: 767px) {
  .nm-blog-header__inner {
    flex-direction: column;
    padding: 24px 20px;
    gap: 16px;
  }
}
```

### Breakpoint conventions

| Name | Max-width | Notes |
|---|---|---|
| Desktop | (default) | No media query needed |
| Tablet | `991px` | Matches Bubble's tablet breakpoint |
| Mobile | `767px` | Matches Bubble's mobile breakpoint |
| Small mobile | `479px` | Only if needed for tight layouts |

### Common responsive patterns

- **Side-by-side → stacked**: `flex-direction: row` → `column` at mobile
- **Padding reduction**: desktop `80px` → tablet `40px` → mobile `20px`
- **Font size scaling**: use `clamp()` or explicit overrides per breakpoint
- **Hide/show elements**: `display: none` at breakpoints rather than Bubble conditional visibility

### Bubble floating nav offset

If the page has floating nav bars, the HTML element needs to account for their height. Measure the nav element's bottom edge at runtime:

```js
var nav = document.getElementById(CONFIG.NAV_ELEMENT_ID);
if (nav) {
  var offset = nav.getBoundingClientRect().bottom;
  el('nm-blog-header').style.paddingTop = offset + 'px';
}
```

---

## Part 6 — Checklist for New HTML Elements

### Structure
- [ ] HTML markup uses empty shells with `id` attributes — no hardcoded data
- [ ] All CSS is inside a `<style>` block in the same file
- [ ] All JS is inside a `<script>` block, wrapped in an IIFE
- [ ] `CONFIG` block at the top of `<script>` with all tunable values
- [ ] File is self-contained — can be pasted into any matching Bubble page and work

### HTML
- [ ] Semantic tags (`<header>`, `<article>`, `<section>`, `<aside>`) over generic `<div>`
- [ ] All JS-touched nodes have stable, prefixed IDs
- [ ] `alt` attributes set, `aria-label` on icon-only buttons, `aria-live` on dynamic regions
- [ ] No inline styles or inline event handlers
- [ ] Above-the-fold images: `fetchpriority="high" decoding="async"`

### CSS
- [ ] BEM naming with `nm-` prefix
- [ ] All selectors scoped to the block root
- [ ] Box-sizing and margin reset within scope
- [ ] Design tokens defined as CSS custom properties on the block root
- [ ] Recoleta font weight is `400` (never bold)

### JavaScript
- [ ] IIFE wrapper, `'use strict';`
- [ ] `el()` and `esc()` helpers defined at the top
- [ ] All API data inserted via `textContent` (or escaped if using `innerHTML`)
- [ ] `console.log` / `console.error` prefixed with element name
- [ ] No leaked globals (only deliberate ones like `window.NM_CONTEXT`)

### Data
- [ ] Data fetched via Bubble Data API (`/api/1.1/obj/...`) for reads
- [ ] `BUBBLE_API` derived from `bubbleApiRoot()` — not hardcoded to live
- [ ] All Data API reads include `{ cache: 'no-store' }`
- [ ] Writes go through Backend Workflow or custom endpoint, never the public Data API
- [ ] [DATA_SCHEMA.md](DATA_SCHEMA.md) consulted for correct field display names
- [ ] Record identification uses URL slug, query param, or context bridge — not inline tokens (or has a documented exception)

### Lifecycle
- [ ] Loading state shown while fetching (skeleton or hidden inner)
- [ ] Error state shown on network failure / empty response / not-found
- [ ] Re-fetch on URL change (popstate + path-polling backstop)
- [ ] Listener attachment guarded against double-binding

### Security
- [ ] No API keys, tokens, passwords, or admin flags in the file
- [ ] Privacy Rules reviewed for every exposed data type and access tier
- [ ] Sensitive fields (emails, financial data, internal notes) explicitly restricted by role
- [ ] Admin elements: Privacy Rules require admin role (not just page-level visibility)
- [ ] Element handles empty API response gracefully (auth blocked OR no data)
- [ ] No `document.cookie` reads or session-cookie forwarding
- [ ] CORS allowlist updated if cross-origin

### Visuals
- [ ] Responsive breakpoints handled via CSS `@media` queries
- [ ] Tested at desktop (default), tablet (≤991px), and mobile (≤767px)
- [ ] Visual matches Figma reference (verified via `preview_screenshot`)
