---
name: hydro-plugin-handler
description: Comprehensive guide to HTTP request handling and WebSocket connections in Hydro plugins. Covers route registration, handler lifecycle, parameter decorators, response building, operations, WebSocket handlers, and inheritance patterns.
---

# Hydro Plugin Development: Handler & Route System

This skill covers how to handle HTTP requests and WebSocket connections in Hydro plugins: Handler lifecycle, route registration, parameter decorators, response building, the operation mechanism, and WebSocket handlers.

---

## 1. Route Registration

```typescript
ctx.Route(name: string, path: string, HandlerClass: typeof Handler, ...permPrivChecker);
```

### Parameters

| Param | Type | Description |
|-------|------|-------------|
| `name` | `string` | Route name, used in `url('route_name', { key: val })` to generate URLs |
| `path` | `string` | URL path pattern, supports `:param` placeholders (e.g. `/blog/:uid/:did`) |
| `HandlerClass` | `class extends Handler` | The handler class |
| `...permPrivChecker` | `PERM` / `PRIV` / `Function` / arrays | Permission checks applied at route level |

### Permission checker variants

```typescript
// Single permission (bitwise bigint)
ctx.Route('my_route', '/my', MyHandler, PERM.PERM_VIEW_PROBLEM);

// Single privilege (number)
ctx.Route('admin_route', '/admin', AdminHandler, PRIV.PRIV_EDIT_SYSTEM);

// Multiple: PERM array, PRIV array, or combined
ctx.Route('my_route', '/my', MyHandler, PERM.PERM_VIEW_PROBLEM, PRIV.PRIV_USER_PROFILE);

// Custom checker function
ctx.Route('my_route', '/my', MyHandler, (this: Handler) => {
    if (!this.user.someCondition) throw new ForbiddenError();
});
```

### Route name conventions

The route name is derived from the Handler class name with `Handler` suffix removed. For example `ContestDetailHandler` → `contest_detail`. The name is used:
- In templates: `{{ url('contest_detail', { tid: tdoc._id }) }}`
- In handlers: `this.url('contest_detail', { tid })`
- In `data-page` attribute on `<html>` for frontend page matching

---

## 2. Handler Lifecycle (Complete Steps)

When an HTTP request arrives, `WebService.handleHttp()` executes the following steps in order. Each step is either a handler method call, an event hook, or a log marker.

```
 Step                     Type            Description
──────────────────────────────────────────────────────────────────
 log/__init               log             Record init timestamp
 init                     method          CSRF protection, basic setup
 handler/init             event           ctx.serial('handler/init', h)
 handler/before-prepare/* event           ctx.serial('handler/before-prepare/${name}#${method}', h)
                          event           ctx.serial('handler/before-prepare/${name}', h)
                          event           ctx.serial('handler/before-prepare', h)
 log/__prepare            log             Record prepare start
 __prepare                method          Base class data loading (lowest layer)
 _prepare                 method          Intermediate data loading
 prepare                  method          Final pre-check / business setup
 log/__prepareDone        log             Record prepare end
 handler/before/*         event           ctx.serial('handler/before/${name}#${method}', h)
                          event           ctx.serial('handler/before/${name}', h)
                          event           ctx.serial('handler/before', h)
 log/__method             log             Record method start
 all                      method          Runs for ALL HTTP methods
 method                   method          'get', 'post', 'put', 'delete', etc.
 log/__methodDone         log             Record method end
 [post${operation}]       method          Only for POST with body.operation
 log/__operationDone      log             Record operation end
 after                    method          Post-method cleanup / UI setup
 handler/after/*          event           ctx.serial('handler/after/${name}#${method}', h)
                          event           ctx.serial('handler/after/${name}', h)
                          event           ctx.serial('handler/after', h)
 cleanup                  method          Final cleanup (always runs)
 handler/finish/*         event           ctx.serial('handler/finish/${name}#${method}', h)
                          event           ctx.serial('handler/finish/${name}', h)
                          event           ctx.serial('handler/finish', h)
 log/__finish             log             Record finish timestamp
```

### Error handling (any step throws)

```
 handler/error/${name}    event           ctx.serial('handler/error/${name}', h, e)
 handler/error            event           ctx.serial('handler/error', h, e)
 onerror                  method          h.onerror(e) → renders error page
```

### Control flow

Any handler method can **return a step name** to jump to that step:

```typescript
async prepare() {
    if (!this.user.hasPriv(PRIV.PRIV_USER_PROFILE)) {
        this.response.redirect = '/login';
        return 'after';  // Skip method execution, jump to 'after'
    }
}
```

### The three prepare layers

Hydro uses an inheritance-based convention for the prepare chain:

| Method | Layer | Purpose | Example |
|--------|-------|---------|---------|
| `__prepare` | Base class | Load core shared data | `ContestDetailBaseHandler.__prepare` loads `tdoc`/`tsdoc` |
| `_prepare` | Intermediate | Load related data, preprocess params | `ProblemHandler._prepare` loads `pdoc` by pid |
| `prepare` | Final subclass | Permission checks, business-specific setup | `ContestDetailHandler.prepare` checks if contest is hidden |

This allows subclass handlers to inherit data loading from parent classes without repeating code.

---

## 3. HTTP Method Handling

### GET handler

```typescript
class MyHandler extends Handler {
    @param('id', Types.ObjectId)
    @param('page', Types.PositiveInt, true)
    async get(domainId: string, id: ObjectId, page = 1) {
        const data = await MyModel.get(domainId, id);
        this.response.template = 'my_detail.html';
        this.response.body = { data, page };
    }
}
```

### POST handler (simple)

```typescript
class MyHandler extends Handler {
    @param('title', Types.Title)
    @param('content', Types.Content)
    async post(domainId: string, title: string, content: string) {
        const id = await MyModel.create(domainId, title, content);
        this.response.redirect = this.url('my_detail', { id });
    }
}
```

### POST with operation mechanism

When POST body contains an `{ "operation": "star" }` field, the framework converts it to a method name:
- `operation: "star"` → calls `postStar()`
- `operation: "delete_item"` → calls `postDeleteItem()` (snake_case → camelCase)

```typescript
class BlogDetailHandler extends Handler {
    // GET request
    @param('did', Types.ObjectId)
    async get(domainId: string, did: ObjectId) {
        this.response.template = 'blog_detail.html';
        this.response.body = { ddoc: await BlogModel.get(did) };
    }

    // POST without operation → post()
    async post() {
        this.checkPriv(PRIV.PRIV_USER_PROFILE);
    }

    // POST with operation=star → postStar()
    @param('did', Types.ObjectId)
    async postStar(domainId: string, did: ObjectId) {
        await BlogModel.setStar(did, this.user._id, true);
        this.back({ star: true });
    }

    // POST with operation=unstar → postUnstar()
    @param('did', Types.ObjectId)
    async postUnstar(domainId: string, did: ObjectId) {
        await BlogModel.setStar(did, this.user._id, false);
        this.back({ star: false });
    }
}
```

### The `all()` method

Runs before the specific HTTP method. Useful for shared logic:

```typescript
class MyHandler extends Handler {
    async all() {
        // Runs for GET, POST, PUT, DELETE, etc.
        this.response.body = { sharedData: '...' };
    }

    async get() {
        // GET-specific logic, can use this.response.body from all()
    }
}
```

---

## 4. Parameter Decorators

Decorators extract and validate parameters from different parts of the request.

| Decorator | Source | Example |
|-----------|--------|---------|
| `@param(name, Type, ...)` | GET query + POST body + route params | General purpose |
| `@get(name, Type, ...)` | URL query string only (`?key=val`) | GET parameters |
| `@post(name, Type, ...)` | Request body only | POST form/JSON |
| `@route(name, Type, ...)` | URL path parameters (`/:id`) | Path params |
| `@query(name, Type, ...)` | Alias for `@get` | — |

### Type specifications

```typescript
import { Types } from 'hydrooj';
// Also: any Schemastery Schema can be used as a type

Types.String          // string, non-empty
Types.Int             // integer
Types.PositiveInt     // positive integer (> 0)
Types.Float           // float number
Types.Boolean         // boolean
Types.ObjectId        // MongoDB ObjectId (24-char hex string)
Types.Title           // string, length 1-256
Types.Content         // string, non-empty
Types.Cuid            // string, valid CUID
Types.Array           // array
Types.UnsignedInt     // integer >= 0
Types.Range           // number range string like "1-100"
Types.PositiveFloat   // float > 0

// Schema as type (validates AND converts)
@param('config', Schema.object({
    name: Schema.string(),
    value: Schema.number(),
}))
```

### Optional parameters

```typescript
// Third argument = true → optional (undefined if missing)
@param('page', Types.PositiveInt, true)
async get(domainId: string, page = 1) { }

// Optional with convert: third arg = 'convert'
// Will pass through undefined but still apply convert if present
@param('filter', Types.String, 'convert')
```

### Validation + Convert

```typescript
// Fourth positional arg = validator function
@param('age', Types.Int, false, (v) => v >= 0 && v <= 150)

// Custom convert function
@param('ids', Types.String, false, null, (v) => v.split(',').map(Number))

// Full form: @param(name, type, isOptional, validator, converter)
```

### The domainId convention

`domainId` is **always the first parameter** and is automatically injected from the request context. It does NOT come from the decorator source — it's extracted from the route or the session.

```typescript
@param('id', Types.ObjectId)
async get(domainId: string, id: ObjectId) {
    // domainId is auto-injected, not from @param
    // id comes from the decorated source
}
```

When the first argument name starts with `domainId` (case-insensitive), the framework detects this and passes `domainId` as the first argument automatically.

---

## 5. Response Building

### Render a template (HTML page)

```typescript
this.response.template = 'my_page.html';
this.response.body = { title: 'Hello', items: [...] };
// Nunjucks renders my_page.html with the body as template context
```

**IMPORTANT — `request.json` behavior**: If the request has `Accept: application/json` header, the template is **NOT rendered** — `this.response.body` is returned as JSON instead. This is controlled by `request.json`:

```typescript
// Set in framework/framework/base.ts:
json: (ctx.request.headers.accept || '').includes('application/json'),
```

When `request.json` is `true`:
- Template rendering is **skipped entirely**
- `response.body` is serialized to JSON (`response.type = 'application/json'`)
- `response.redirect` is also skipped (redirect URL is included in JSON body as `url` field)
- Error responses are JSON too: `{ error: ... }`

This means the **same handler/route serves both HTML and JSON** based on the `Accept` header:

```typescript
// In handler code, check this.request.json:
if (this.request.json) {
    // API call — skip template-only logic
    return;
}

// Real example from contest handler:
async get(domainId: string, tid: ObjectId) {
    this.response.template = 'contest_detail.html';
    this.response.body = { tdoc, tsdoc };
    if (this.request.json) return; // Skip template-only content processing
    this.response.body.tdoc.content = /* process content */;
}
```

### Return JSON (API endpoint)

```typescript
this.response.body = { success: true, data: [...] };
// No template set → automatically returns JSON
```

### Redirect

```typescript
// Redirect to a named route
this.response.redirect = this.url('my_detail', { id });

// Redirect to a literal URL
this.response.redirect = '/some/path';
```

### Back to referer

```typescript
this.back({ star: true });  // Sets body + redirects to request referer or '/'
```

### File download

```typescript
this.binary(buffer, 'report.pdf');  // Sets Content-Disposition header
```

### Set response status

```typescript
this.response.status = 201;   // HTTP status code
this.response.type = 'text/plain';  // Content-Type
```

### Set response headers

```typescript
this.response.set('X-Custom-Header', 'value');
```

### Available response properties

| Property | Type | Description |
|----------|------|-------------|
| `this.response.body` | `any` | Response data (object for JSON, template context for HTML) |
| `this.response.template` | `string` | Nunjucks template name |
| `this.response.redirect` | `string` | Redirect URL |
| `this.response.status` | `number` | HTTP status code |
| `this.response.type` | `string` | Content-Type |

---

## 6. Handler Context Properties

Every handler has access to these built-in properties:

| Property | Type | Description |
|----------|------|-------------|
| `this.user` | `User` | Current logged-in user (or guest) |
| `this.domain` | `DomainDoc` | Current domain |
| `this.request` | `KoaRequest` | Koa request object |
| `this.response` | `KoaResponse` | Koa response object |
| `this.args` | `Record<string, any>` | All parsed arguments + timing data |
| `this.session` | `Session` | Session data |
| `this.ctx` | `Context` | Cordis context (access services) |

### Common handler methods

```typescript
// Permission checks
this.checkPerm(PERM.PERM_EDIT_PROBLEM);  // Throws ForbiddenError if no permission
this.checkPriv(PRIV.PRIV_EDIT_SYSTEM);    // Throws if no privilege
this.checkPriv(PRIV.PRIV_USER_PROFILE);   // Common: must be logged in

// Rate limiting
this.limitRate('add_blog', 3600, 60);  // (key, period_seconds, max_attempts)

// URL generation
this.url('route_name', { key: val });  // Generate URL from route name + params

// Translation
this.translate('key');  // i18n translate
```

---

## 7. The `after()` Method

The `after()` method runs after the main HTTP method handler and before the `handler/after` event hooks. It's commonly used for post-processing the response.

Real example from `ContestDetailBaseHandler`:

```typescript
@param('tid', Types.ObjectId, true)
async after(domainId: string, tid: ObjectId) {
    if (!tid || this.tdoc.rule === 'homework') return;
    if (this.request.json || !this.response.template) return;
    // Set breadcrumb navigation after the page is loaded
    this.response.body.overrideNav = [
        { name: 'contest_main', args: {}, displayName: 'Back to contest list', checker: () => true },
        { name: 'contest_detail', displayName: this.tdoc.title, args: { tid }, checker: () => true },
        { name: 'contest_scoreboard', args: { tid, prefix: 'contest_scoreboard' },
          checker: () => contest.canShowScoreboard.call(this, this.tdoc, true) },
    ];
}
```

---

## 8. WebSocket (ConnectionHandler)

WebSocket connections use `ConnectionHandler` instead of `Handler`.

### Basic WebSocket handler

```typescript
import { ConnectionHandler, subscribe } from 'hydrooj';

class MyWsHandler extends ConnectionHandler<Context> {
    @subscribe('my-channel')
    async send(channel: string) {
        // this.send(data) — send data to client
        // this.close() — close connection
        // this.conn — the connection object
    }

    async prepare() {
        // Optional: setup before subscribing
    }

    async message(payload: any) {
        // Handle incoming messages from client
    }

    async cleanup() {
        // Cleanup when connection closes
    }
}
```

### Registering a WebSocket route

```typescript
ctx.Connection('my_ws', '/ws/my-channel', MyWsHandler);
```

### Using WebSocket events

```typescript
// Listen for new subscriptions
ctx.on('subscription/enable', (channel, h, privileged, onDispose) => {
    // h is the ConnectionHandler instance
    // onDispose(callback) registers cleanup for this subscription
    onDispose(() => { /* cleanup */ });
});
```

### WebSocket lifecycle

1. `handler/create` events fire
2. Permission checker runs
3. `_prepare()` → `prepare()` called
4. Subscriptions registered via `@subscribe` decorator
5. Connection enters active state
6. `message()` handler receives incoming messages
7. `cleanup()` runs on disconnect/error

### SSE (Server-Sent Events) mode

ConnectionHandler also supports SSE when accessed via HTTP (not WebSocket upgrade):
```typescript
// SSE auto-detected when client connects via HTTP but handler is ConnectionHandler
// this.conn.send(data) works for both WebSocket and SSE
```

---

## 9. Handler Inheritance Patterns

Handlers support class inheritance for sharing logic:

```typescript
// Base handler: loads shared data
class ContestDetailBaseHandler extends Handler {
    tdoc?: Tdoc;
    tsdoc?: any;

    @param('tid', Types.ObjectId, true)
    async __prepare(domainId: string, tid: ObjectId) {
        if (!tid) return;
        [this.tdoc, this.tsdoc] = await Promise.all([
            contest.get(domainId, tid),
            contest.getStatus(domainId, tid, this.user._id),
        ]);
    }
}

// Sub handler: adds specific logic
class ContestScoreboardHandler extends ContestDetailBaseHandler {
    @param('tid', Types.ObjectId)
    async prepare(domainId: string, tid: ObjectId) {
        // tdoc already loaded by __prepare
        if (!contest.canShowScoreboard.call(this, this.tdoc, true)) {
            throw new ContestNotLiveError(tid);
        }
    }

    @param('tid', Types.ObjectId)
    async get(domainId: string, tid: ObjectId) {
        this.response.template = 'contest_scoreboard.html';
        this.response.body = { tdoc: this.tdoc };
    }
}
```

### Inheritance + prepare chain

When using inheritance, the three prepare layers map naturally:
- `__prepare` in the **base** class — load shared data (e.g., tdoc)
- `_prepare` in **intermediate** class — load secondary data (e.g., pdoc)
- `prepare` in the **leaf** class — permission checks, final setup

Each layer uses `@param` decorators independently. The framework collects all `@param` decorators across the inheritance chain for each method name.

---

## 10. Common Patterns

### CRUD Handler (combines GET + multiple POST operations)

```typescript
class ItemHandler extends Handler {
    item?: ItemDoc;

    @param('id', Types.ObjectId, true)
    async _prepare(domainId: string, id?: ObjectId) {
        if (id) {
            this.item = await ItemModel.get(domainId, id);
            if (!this.item) throw new NotFoundError(id);
        }
    }

    // List items
    @param('page', Types.PositiveInt, true)
    async get(domainId: string, page = 1) {
        const [items, count] = await this.ctx.db.paginate(
            ItemModel.getMulti(domainId), page, 20,
        );
        this.response.template = 'item_list.html';
        this.response.body = { items, count, page };
    }

    // Create item (POST without operation)
    @param('title', Types.Title)
    @param('content', Types.Content)
    async post(domainId: string, title: string, content: string) {
        this.checkPerm(PERM.PERM_CREATE_PROBLEM);
        await this.limitRate('create_item', 60, 10);
        const id = await ItemModel.add(domainId, title, content, this.user._id);
        this.response.redirect = this.url('item_detail', { id });
    }

    // Edit (POST with operation=edit)
    @param('id', Types.ObjectId)
    @param('title', Types.Title)
    async postEdit(domainId: string, id: ObjectId, title: string) {
        if (!this.user.own(this.item!)) this.checkPerm(PERM.PERM_EDIT_PROBLEM);
        await ItemModel.edit(domainId, id, { title });
        this.back();
    }

    // Delete (POST with operation=delete)
    @param('id', Types.ObjectId)
    async postDelete(domainId: string, id: ObjectId) {
        if (!this.user.own(this.item!)) this.checkPerm(PERM.PERM_EDIT_PROBLEM);
        await ItemModel.del(domainId, id);
        this.response.redirect = this.url('item_list');
    }
}

// Register once, handles GET + POST + operations
ctx.Route('item_list', '/item', ItemHandler);
ctx.Route('item_detail', '/item/:id', ItemHandler);
```

### JSON API Handler

```typescript
class ApiHandler extends Handler {
    @param('query', Types.String)
    async get(domainId: string, query: string) {
        const results = await searchItems(domainId, query);
        this.response.body = { results };  // No template = JSON response
    }
}
```
