---
name: hydro-plugin-frontend
description: Guide to frontend plugin development including page registration (NamedPage, AutoloadPage), Nunjucks templates, UI injection, and internationalization (i18n). Build interactive frontend features for Hydro plugins.
---

# Hydro Plugin Development: Frontend, Templates, UI & Integration

This skill covers the frontend side of Hydro plugins: page registration (`addPage`), Nunjucks templates, UI injection (`injectUI`), internationalization (`i18n`).

---

## 1. Frontend Pages (`addPage`)

Frontend pages use the `@hydrooj/ui-default` npm package. The page system is built on top of webpack's `require.context` for auto-discovery within `@hydrooj/ui-default` itself. External plugins import from `@hydrooj/ui-default` as a dependency.

### Import

```typescript
// External plugin imports from @hydrooj/ui-default
import { NamedPage, AutoloadPage, addPage } from '@hydrooj/ui-default';
import { Notification, $, _, React, i18n, request, tpl, delay } from '@hydrooj/ui-default';
```

### Page class API

```typescript
class Page {
    moduleName?: string;
    autoload = false;
    afterLoading?: (pagename: string, loadPage: (name: string) => Promise<any>) => any;
    beforeLoading?: (pagename: string, loadPage: (name: string) => Promise<any>) => any;

    constructor(
        name: string | string[],                          // Page name(s) to match
        afterLoading?: Callback,                          // Runs after page loads
        beforeLoading?: Callback,                         // Runs before page loads
    )
    constructor(
        name: string | string[],
        moduleName: string,                               // For loadPage() targeting
        afterLoading?: Callback,
        beforeLoading?: Callback,
    )
}
```

### NamedPage — runs only on specific pages

```typescript
import { NamedPage, i18n, request, tpl } from '@hydrooj/ui-default';

addPage(new NamedPage('home_account', (pagename) => {
    // Only runs when data-page="home_account" on <html>
    const $type = $(tpl`<select class="select">
        <option value="gravatar">${i18n('Gravatar')}</option>
        <option value="upload">${i18n('Upload')}</option>
    </select>`);
    // ... DOM manipulation
}));
```

**Page name matching**: The current page name is read from `document.documentElement.getAttribute('data-page')`. This attribute is set server-side by the template system based on the route name.

```typescript
// Match a single page
new NamedPage('problem_detail', callback);

// Match multiple pages
new NamedPage(['problem_detail', 'contest_detail_problem', 'homework_detail_problem'], callback);
```

### AutoloadPage — runs on EVERY page

```typescript
import { AutoloadPage, Notification, i18n } from '@hydrooj/ui-default';

addPage(new AutoloadPage('notificationPage', (pagename) => {
    // Runs on EVERY page load
    const message = i18n(`Hint::Page::${pagename}`);
    const item = localStorage.getItem(`hint.${message}`);
    if (message !== `Hint::Page::${pagename}` && !item) {
        Notification.info(message, message.length * 500);
        localStorage.setItem(`hint.${message}`, true);
    }
}));
```

AutoloadPage callbacks run on **every** page load and sit at the **front** of the load sequence (all autoload `beforeLoading` hooks run first). An `await` inside one taxes the entire site, not just the current page — keep AutoloadPage callbacks strictly synchronous (see "Async work in callbacks" below).

### Loading sequence

Pages are executed in this order (see `hydro.ts`):

```
1. All AutoloadPage beforeLoading hooks
2. NamedPage beforeLoading hooks (for current page)
3. All AutoloadPage afterLoading hooks
4. NamedPage afterLoading hooks (for current page)
```

### Registering pages with `addPage()`

Plugins **must** use `addPage()` to register pages (external plugins are not compiled by `@hydrooj/ui-default`'s webpack, so `export default` won't work):

```typescript
import { addPage, NamedPage, AutoloadPage } from '@hydrooj/ui-default';

// Register a Page instance
addPage(new NamedPage('my_page', (pagename) => {
    // Page logic
}));

// Register an AutoloadPage
addPage(new AutoloadPage('my_feature', (pagename) => {
    // Runs on every page
}));

// Register a plain function (runs during page loader init)
addPage(() => { console.log('init'); });
```

The registration must happen before page initialization (i.e., during script loading, not in an async callback).

### beforeLoading / afterLoading

```typescript
addPage(new NamedPage('problem_detail', {
    // afterLoading (default): runs after the page content is ready
    afterLoading(pagename, loadPage) {
        // loadPage(name) can dynamically load another page's hooks
        // e.g., loadPage('problem_sidebar')
    },
    // beforeLoading: runs before page content is rendered
    beforeLoading(pagename, loadPage) {
        // Pre-fetch data, setup global listeners, etc.
    },
}));
```

#### Async work in callbacks — keep it out of the load sequence

All page callbacks are awaited **sequentially** in `initPageLoader()` (see `hydro.ts`), and the `.page-loader` spinner only hides after every callback has finished. There is also a performance watchdog: a callback taking >16ms (dev) or >256ms (production) logs a `took Xms` warning. Callbacks are therefore meant to be **fast, synchronous initialization** — avoid awaiting anything slow inside them:

- **Don't** `await` network requests (`request.get` / `request.post` / `fetch` etc.) or slow I/O in a callback. It blocks every other page's `beforeLoading` / `afterLoading` behind it and delays the whole page.
- **OK**: `await import(...)` for heavy modules (editor, socket, echarts…). This is the standard way to keep them out of the main bundle. Put all imports at the top of the callback and run them in parallel with `Promise.all`.
- **Prefer** real work in event handlers (`.on('click', ...)`, `setInterval`, …) where `await` is unrestricted, or fire-and-forget inside the callback: `someAsync().catch(...)`.

```typescript
import { addPage, NamedPage, request } from '@hydrooj/ui-default';

addPage(new NamedPage('my_page', async () => {
    // OK: dynamic imports first, in parallel
    const [{ default: Editor }] = await Promise.all([
        import('./editor'),
    ]);
    // sync DOM binding...
    $('.my-btn').on('click', async () => {
        // OK: awaited network calls belong in event handlers
        await request.post('', { op: 'run' });
    });
}));
```

#### Bad examples — what NOT to do

```javascript
// BAD: network request in a callback — the whole load sequence (and the
// .page-loader spinner) waits for this to resolve
addPage(new NamedPage('my_page', async () => {
    const data = await request.get('/api/slow');   // blocks every later callback
    $('.my-widget').text(data);
}));

// BAD: same mistake in an AutoloadPage — this runs on EVERY page load,
// so the entire site pays the latency, not just one page
addPage(new AutoloadPage('my_feature', async () => {
    await request.get('/api/whatever');            // blocks on every page
}));

// BAD: artificial delay awaited in a callback
addPage(new AutoloadPage('my_feature', async () => {
    await delay(500);                              // +500ms on every page load
}));

// GOOD: bind synchronously, update later (fire-and-forget)
addPage(new AutoloadPage('my_feature', () => {
    request.get('/api/whatever').then((data) => {
        $('.my-widget').text(data);
    }).catch((e) => Notification.error(e));
}));
```

Also keep `addPage()` calls at module top level (synchronous module evaluation) — a top-level `await` in your plugin script delays registration and can race with page loader initialization.

### Available frontend utilities

```typescript
// From '@hydrooj/ui-default':
import {
    i18n,           // i18n('key') → translated string
    request,        // HTTP client { .get(url), .post(url, data), .postFile(url, formData) }
    tpl,            // Template literal tag for safe HTML (escapes by default)
    delay,          // (ms) → Promise<void>
    pjax,           // PJAX navigation { .request({ url }) }
    loadReactRedux, // Load React + Redux provider
    rawHtml,        // Mark string as raw HTML (bypasses escaping in tpl)
} from '@hydrooj/ui-default';

// Additional exports from '@hydrooj/ui-default':
import {
    $, _, React, ReactDOM, redux,   // Libraries
    Notification,                     // Toast notifications
    Socket,                           // WebSocket client
    uploadFiles,                      // File upload helper
    loadMonaco,                       // Monaco editor loader
    addPage,                          // Page registration
    AutoComplete,                     // Autocomplete component
} from '@hydrooj/ui-default';
```

### Global context objects

```typescript
// Available on window:
window.UiContext     // Server-rendered UI context (JSON parsed in entry.js)
window.UserContext   // Current user context (JSON parsed in entry.js)
window.Hydro         // Global Hydro namespace
window.LANGS         // Translation dictionary for current language
```

---

## 2. CSS / Styling

External plugins **cannot** use `.page.styl` auto-discovery (that's a built-in `@hydrooj/ui-default` feature compiled by webpack). Options for plugin styling:

1. **Inline styles in templates**:
```html
<style>
.my-plugin-widget { color: red; }
</style>
<div class="my-plugin-widget">...</div>
```

2. **DOM injection in page scripts**:
```typescript
import { NamedPage } from '@hydrooj/ui-default';
new NamedPage('my_page', () => {
    const style = document.createElement('style');
    style.innerHTML = `.my-widget { color: red; }`;
    document.head.append(style);
});
```

3. **Styling via Nunjucks templates** — put `<style>` blocks in your template HTML.

---

## 3. Nunjucks Templates (Server-side Rendering)

Plugins can provide Nunjucks templates in a `templates/` directory. The `TemplateService` auto-discovers them during initialization.

### Auto-discovery

```typescript
// TemplateService scans all global.addons for template/ or templates/ directories
// in [Context.init] lifecycle (backendlib/template.ts:254-275)
const p = locateFile(addonPath, ['template', 'templates']);
if (p && isDirectory(p)) {
    const files = await getFiles(p);
    for (const file of files) {
        this.registry[file] = await fs.readFile(path.resolve(p, file), 'utf-8');
    }
}
```

### Template inheritance

```html
{% extends "layout/basic.html" %}
{% block content %}
<div class="section">
    <h1>{{ ddoc.title }}</h1>
</div>
{% endblock %}
```

Common base templates:
- `layout/basic.html` — Standard page layout with nav, sidebar, footer
- `layout/server_rendered_markdown.html` — Markdown content page

### Template imports

```html
{% import "components/nothing.html" as nothing with context %}
{% import "components/paginator.html" as paginator with context %}
{% import "components/user.html" as user with context %}
```

### Built-in template variables & functions

Available in all templates:

| Variable | Description |
|----------|-------------|
| `_()` | i18n translate function |
| `handler` | Current handler instance |
| `ctx` | Current Cordis context |
| `page_name` | Current page/route name |
| `UserContext` | Current user context object |
| `UiContext` | UI context from handler |
| `PERM` | Permission constants |
| `PRIV` | Privilege constants |
| `STATUS` | Status constants |

| Function/Filter | Description |
|-----------------|-------------|
| `url(name, args)` | Generate URL from route name |
| `datetimeSpan(date, relative?, format?)` | Render datetime |
| `paginate(page, count)` | Pagination helper |
| `\|json` | JSON stringify |
| `\|markdown` | Render markdown to HTML |
| `\|ansi` | ANSI to HTML |
| `\|content(language, html)` | Render localized content |
| `\|base64_encode` / `\|base64_decode` | Base64 |
| `\|assign(data)` | Merge into object |
| `set(obj, key, val)` | Set value in template |
| `model` | Access Hydro models |
| `ui` | Access Hydro.ui (injectUI nodes) |
| `templateExists(name)` | Check if template exists |
| `findSubModule(prefix)` | Find sub-module templates |

### Using handler data in templates

```typescript
// Handler sets template + body:
this.response.template = 'blog_detail.html';
this.response.body = {
    ddoc: this.ddoc,
    udoc,
    dsdoc,
};
```

```html
<!-- blog_detail.html uses the body properties directly: -->
<h1>{{ ddoc.title }}</h1>
<p>{{ _('By {0}').format(udoc.uname) }}</p>
<span>{{ datetimeSpan(ddoc.updateAt)|safe }}</span>
<a href="{{ url('blog_edit', { uid: udoc._id, did: ddoc._id }) }}">{{ _('Edit') }}</a>
```

### Template override per domain

Templates support domain-specific overrides:
```typescript
// TemplateService automatically looks for:
// 1. template_name.domainId.html (domain-specific)
// 2. template_name.html (fallback)
h.renderHTML = ((orig) => function(name, args) {
    let templateName = `${s[0]}.${args.domainId}.${s[1]}`;
    if (!that.registry[templateName]) templateName = name;
    return orig(templateName, args);
})(h.renderHTML);
```

---

## 4. `ctx.injectUI()` — UI Element Injection

Adds UI elements to predefined injection points in the frontend.

### Injection points

| Position | Description | Typical use |
|----------|-------------|-------------|
| `Nav` | Top navigation bar | New section links |
| `ProblemAdd` | Problem creation page | Import buttons |
| `Notification` | System notifications | Warning/info banners |
| `UserDropdown` | User dropdown menu | User-specific links |
| `DomainManage` | Domain management sidebar | Admin sections |
| `ControlPanel` | System control panel | System admin tools |

### Registration

```typescript
// Static args (object)
ctx.injectUI('ProblemAdd', 'my_import', {
    icon: 'copy',
    text: 'From My Source',
});

// Dynamic args (function — receives handler for per-request data)
ctx.injectUI('UserDropdown', 'blog_main',
    (handler) => ({
        icon: 'book',
        displayName: 'Blog',
        uid: handler.user._id.toString(),
    }),
    PRIV.PRIV_USER_PROFILE,  // Only show for logged-in users
);

// With permission check
ctx.injectUI('Nav', 'my_section', {
    prefix: 'my_section',
    icon: 'star',
    text: 'My Section',
}, PERM.PERM_VIEW_PROBLEM);

// With ordering (before another item)
ctx.injectUI('Nav', 'my_item', {
    prefix: 'my_item',
    before: 'problem_main',  // Insert before 'problem_main' in Nav
}, PERM.PERM_VIEW_PROBLEM);
```

### Permission checking in injectUI

The 4th+ arguments are permission/privilege checks:
- `PERM.xxx` (bigint) — domain-level permission
- `PRIV.xxx` (number) — system-level privilege
- `(handler) => boolean` — custom checker function

The checker runs client-side when rendering the UI. If the checker returns false, the item is hidden.

---

## 5. Internationalization (`ctx.i18n`)

### Backend i18n

```typescript
ctx.i18n.load('zh', {
    "{0}'s blog": '{0} 的博客',
    Blog: '博客',
    blog_detail: '博客详情',
    'Create a Post': '创建文章',
});
ctx.i18n.load('en', {
    blog_main: 'Blog',
    blog_detail: 'Blog Detail',
});
ctx.i18n.load('zh_TW', {
    Blog: '部落格',
});
```

### Using translations

**In templates**:
```html
{{ _('Create a Post') }}          <!-- Simple -->
{{ _('{0} views').format(count) }} <!-- With parameters -->
```

**In handlers**:
```typescript
this.translate('key');
```

**In frontend pages**:
```typescript
i18n('Blog');                    // Returns translated string
i18n('{0} views', count);       // With parameters
```

### Translation lookup

1. Exact match: `'Blog'` → looks up `'Blog'` key
2. Fallback chain: `zh_TW` → `zh` → `en` → key itself
3. Frontend uses `window.LANGS` dictionary loaded server-side
