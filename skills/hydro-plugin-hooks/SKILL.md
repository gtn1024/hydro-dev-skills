---
name: hydro-plugin-hooks
description: Complete reference to Hydro's event system including event API (ctx.on/emit/broadcast), cluster-safe broadcasting, 60+ event types, resource cleanup (ctx.effect), timed tasks (ctx.interval), and replaceable modules (ctx.provideModule).
---

# Hydro Plugin Development: Event System & Hooks

This skill covers Hydro's event system: how to listen for system events, broadcast across processes, manage plugin lifecycle resources, and provide replaceable modules.

---

## 1. Event API Overview

All event methods are available on the `Context` object (`ctx`):

| Method | Signature | Description |
|--------|-----------|-------------|
| `ctx.on(event, handler)` | Returns dispose function | Register listener |
| `ctx.once(event, handler)` | Returns dispose function | Register one-time listener |
| `ctx.emit(event, ...args)` | `void` | Synchronous fire (in-process only) |
| `ctx.parallel(event, ...args)` | `Promise<void>` | Fire all listeners concurrently (in-process only) |
| `ctx.serial(event, ...args)` | `Promise<void>` | Fire listeners one by one, await each (in-process only) |
| `ctx.broadcast(event, ...args)` | `void` | Cross-process broadcast (see cluster safety below) |

### Cluster safety

**`ctx.emit()` / `ctx.parallel()` / `ctx.serial()` — in-process only.** They do NOT cross process boundaries. If you have PM2 with 4 instances, calling `ctx.parallel('problem/add', ...)` only triggers listeners in the **current** process.

**`ctx.broadcast()` — cross-process AND cross-server (cluster safe).** The implementation in `packages/hydrooj/src/service/bus.ts` and `packages/hydrooj/src/model/task.ts`:

1. **PM2 cluster mode** (multiple processes on same server): Uses PM2 `launchBus` to relay events across all processes.
2. **Multi-server / standalone mode**: Uses MongoDB `event` collection:
   - `broadcast` inserts a document into the `event` collection
   - Each server watches via MongoDB Change Stream (`collEvent.watch()`)
   - If Change Stream unavailable (standalone MongoDB), falls back to polling (`findOneAndUpdate` every 500ms)
   - Events have `ack` array (server IDs that processed it) and `expire` (TTL auto-delete)
   - Each server processes unacknowledged events and calls `app.parallel()` locally

```
ctx.broadcast('record/judge', rdoc, updated, pdoc)
  → ctx.emit('bus/broadcast', 'record/judge', [rdoc, updated, pdoc])

  PM2 mode:
    → process.send({ type: 'hydro:broadcast', ... }) via PM2 bus
    → All processes on same server receive

  Multi-server / standalone:
    → Insert into MongoDB 'event' collection
    → Each server's watcher/poller detects new event
    → Each server calls app.parallel('record/judge', rdoc, updated, pdoc)
```

**Rule of thumb for plugin developers:**
- If the event affects shared state (e.g., "record judged", "problem deleted"), use `ctx.broadcast()` so all processes and servers react.
- If the event is local (e.g., handler lifecycle), `ctx.parallel()` is fine.
- When listening: `ctx.on()` registers in the **current process only**. For broadcast events, every process/server has its own listeners, and `broadcast` triggers all of them.

---

## 2. Complete Event Catalog

Defined in `packages/hydrooj/src/service/bus.ts` (`EventMap` interface).

### Application lifecycle

| Event | Signature | When |
|-------|-----------|------|
| `app/listen` | `() => void` | Server starts listening on port |
| `app/started` | `() => void` | Application startup complete |
| `app/ready` | `() => VoidReturn` | All plugins loaded |
| `app/exit` | `() => VoidReturn` | Application shutting down |
| `app/before-reload` | `(entries: Set<string>) => VoidReturn` | Before hot-reloading plugins |
| `app/reload` | `(entries: Set<string>) => VoidReturn` | After hot-reloading plugins |

### Database

| Event | Signature | When |
|-------|-----------|------|
| `database/connect` | `(db: Db) => void` | MongoDB connection established |
| `database/config` | `() => VoidReturn` | Database config loaded |

### User

| Event | Signature | When |
|-------|-----------|------|
| `user/get` | `(udoc: User) => void` | User data fetched |
| `user/message` | `(uid: number[], mdoc) => void` | Message sent to users |
| `user/delcache` | `(content: string \| true) => void` | User cache invalidated |
| `user/import/parse` | `(payload: any) => VoidReturn` | Parsing user import data |
| `user/import/create` | `(uid: number, udoc: any) => VoidReturn` | Creating imported user |

### Domain

| Event | Signature | When |
|-------|-----------|------|
| `domain/create` | `(ddoc: DomainDoc) => VoidReturn` | Domain created |
| `domain/before-get` | `(query: Filter<DomainDoc>) => VoidReturn` | Before fetching domain |
| `domain/get` | `(ddoc: DomainDoc) => VoidReturn` | Domain fetched |
| `domain/before-update` | `(domainId, $set) => VoidReturn` | Before updating domain |
| `domain/update` | `(domainId, $set, ddoc) => VoidReturn` | Domain updated |
| `domain/delete` | `(domainId: string) => VoidReturn` | Domain deleted |
| `domain/delete-cache` | `(domainId: string) => VoidReturn` | Domain cache invalidated |

### Problem

| Event | Signature | When |
|-------|-----------|------|
| `problem/before-add` | `(domainId, content, owner, docId, doc) => VoidReturn` | Before creating problem |
| `problem/add` | `(doc: Partial<ProblemDoc>, docId: number) => VoidReturn` | Problem created |
| `problem/before-edit` | `(doc, $unset) => VoidReturn` | Before editing problem |
| `problem/edit` | `(doc: ProblemDoc) => VoidReturn` | Problem edited |
| `problem/before-del` | `(domainId, docId) => VoidReturn` | Before deleting problem |
| `problem/del` | `(domainId, docId) => VoidReturn` | Problem deleted |
| `problem/list` | `(query, handler, sort?) => VoidReturn` | Problem list queried |
| `problem/get` | `(doc: ProblemDoc, handler) => VoidReturn` | Problem fetched |
| `problem/addTestdata` | `(domainId, docId, name, payload) => VoidReturn` | Testdata file added |
| `problem/renameTestdata` | `(domainId, docId, name, newName) => VoidReturn` | Testdata renamed |
| `problem/delTestdata` | `(domainId, docId, name[]) => VoidReturn` | Testdata deleted |
| `problem/addAdditionalFile` | `(domainId, docId, name, payload) => VoidReturn` | Additional file added |

### Contest

| Event | Signature | When |
|-------|-----------|------|
| `contest/before-add` | `(payload) => VoidReturn` | Before creating contest |
| `contest/add` | `(payload, id: ObjectId) => VoidReturn` | Contest created |
| `contest/edit` | `(payload: Tdoc) => VoidReturn` | Contest edited |
| `contest/list` | `(query, handler) => VoidReturn` | Contest list queried |
| `contest/scoreboard` | `(tdoc, rows, udict, pdict) => VoidReturn` | Scoreboard generated |
| `contest/balloon` | `(domainId, tid, bdoc) => VoidReturn` | Balloon event |
| `contest/del` | `(domainId, tid) => VoidReturn` | Contest deleted |

### Record (submission)

| Event | Signature | When |
|-------|-----------|------|
| `record/change` | `(rdoc, $set?, $push?, body?) => void` | Record updated |
| `record/judge` | `(rdoc, updated, pdoc?, updater?) => VoidReturn` | Judging complete |

### Discussion

| Event | Signature | When |
|-------|-----------|------|
| `discussion/before-add` | `(payload) => VoidReturn` | Before creating discussion |
| `discussion/add` | `(payload) => VoidReturn` | Discussion created |

### Training

| Event | Signature | When |
|-------|-----------|------|
| `training/list` | `(query, handler) => VoidReturn` | Training list queried |
| `training/get` | `(tdoc, handler) => VoidReturn` | Training fetched |

### Document

| Event | Signature | When |
|-------|-----------|------|
| `document/add` | `(doc: any) => VoidReturn` | Any document added |
| `document/set` | `(domainId, docType, docId, $set, $unset) => VoidReturn` | Document updated |

### System & monitoring

| Event | Signature | When |
|-------|-----------|------|
| `system/setting` | `(args) => VoidReturn` | System setting changed |
| `monitor/update` | `(type, $set) => VoidReturn` | Monitor data updated |
| `monitor/collect` | `(info: any) => VoidReturn` | Collect monitoring info |
| `api/update` | `() => void` | API registry updated |
| `task/daily` | `() => VoidReturn` | Daily task triggered |
| `task/daily/finish` | `(pref) => void` | Daily tasks complete |
| `oplog/log` | `(type, handler, args, data) => VoidReturn` | Operation logged |

### Handler hooks

| Event | Signature | When |
|-------|-----------|------|
| `handler/create` | `(h, type) => VoidReturn` | Handler instantiated |
| `handler/init` | `(h) => VoidReturn` | Handler init phase |
| `handler/before-prepare` | `(h) => VoidReturn` | Before prepare phase |
| `handler/before-prepare/${name}` | `(h) => VoidReturn` | Before prepare for specific handler |
| `handler/before-prepare/${name}#${method}` | `(h) => VoidReturn` | Before prepare for specific handler+method |
| `handler/before` | `(h) => VoidReturn` | Before main method |
| `handler/before/${name}` | `(h) => VoidReturn` | Before specific handler |
| `handler/after` | `(h) => VoidReturn` | After main method |
| `handler/after/${name}` | `(h) => VoidReturn` | After specific handler |
| `handler/after/${name}#${method}` | `(h) => VoidReturn` | After specific handler+method |
| `handler/finish` | `(h) => VoidReturn` | Handler finished |
| `handler/error` | `(h, e) => VoidReturn` | Handler error |
| `handler/error/${name}` | `(h, e) => VoidReturn` | Handler error for specific handler |

### WebSocket / Subscription

| Event | Signature | When |
|-------|-----------|------|
| `subscription/init` | `(h, privileged) => VoidReturn` | WebSocket connection init |
| `subscription/subscribe` | `(channel, user, metadata) => VoidReturn` | Client subscribes |
| `subscription/enable` | `(channel, h, privileged, onDispose) => VoidReturn` | Subscription activated |

### File watching (dev mode)

| Event | Signature | When |
|-------|-----------|------|
| `app/watch/change` | `(path: string) => VoidReturn` | File changed |
| `app/watch/unlink` | `(path: string) => VoidReturn` | File deleted |

---

## 3. Real-world Examples

### Search index sync (ElasticSearch plugin)

```typescript
this.ctx.on('problem/add', async (doc, docId) => {
    await this.client.index({
        index: 'problem',
        id: `${doc.domainId}/${docId}`,
        document: processDocument(doc),
    });
});

this.ctx.on('problem/edit', async (pdoc) => {
    await this.client.index({
        index: 'problem',
        id: `${pdoc.domainId}/${pdoc.docId}`,
        document: processDocument(pdoc),
    });
});

this.ctx.on('problem/del', async (domainId, docId) => {
    await this.client.delete({
        index: 'problem',
        id: `${domainId}/${docId}`,
    });
});
```

### Post-handler hook (UI Default plugin)

```typescript
// Modify response after a specific handler method runs
ctx.on('handler/after/DiscussionRaw', async (that) => {
    if (that.args.render && that.response.type === 'text/markdown') {
        that.response.type = 'text/html';
        that.response.body = await markdown.render(that.response.body);
    }
});

// Run after ALL handlers
ctx.on('handler/after', async (that) => {
    that.UiContext.SWConfig = {
        preload: SystemModel.get('ui-default.preload'),
        // ...
    };
});
```

### One-time setup on user registration (A11Y plugin)

```typescript
ctx.on('handler/after/UserRegisterWithCode#post', async (that) => {
    if (that.session.uid === 2) await UserModel.setSuperAdmin(2);
});
```

### Modify domain query before execution

```typescript
ctx.on('domain/before-get', (query) => {
    // Modify the query before it's executed
    query.someField = 'value';
});
```

### Monitor contest balloon events

```typescript
ctx.on('contest/balloon', (domainId, tid, bdoc) => {
    // Send notification, update scoreboard, etc.
});
```

---

## 4. `ctx.effect()` — Automatic Resource Cleanup

`ctx.effect()` registers a cleanup function that runs when the plugin is unloaded or the context is disposed.

```typescript
// Inside a Service class
constructor(ctx: Context) {
    super(ctx, 'myService');

    ctx.effect(() => {
        const conn = createConnection();
        const timer = setInterval(() => { /* ... */ }, 5000);
        return () => {
            conn.close();
            clearInterval(timer);
        };
    });
}
```

### Real example (VJudge plugin)

```typescript
this.ctx.effect(() => {
    this.providers[type] = provider;
    const services = [];
    for (const account of this.accounts.filter((a) => a.type === type)) {
        if (account.enableOn && !account.enableOn.includes(os.hostname())) continue;
        const service = new AccountService(provider, account, this.ctx);
        services.push(service);
        this.pool[`${account.type}/${account.handle}`] = service;
    }
    return () => {
        // Cleanup: stop all services for this provider
        for (const service of services) service.stop();
        delete this.providers[type];
    };
});
```

### `ctx.on()` already returns a dispose function

```typescript
const dispose = ctx.on('problem/add', handler);
// dispose is called automatically when plugin unloads
// You can also call dispose() manually to unregister early
```

---

## 5. `ctx.interval()` — Scheduled Tasks

Registers a recurring task that auto-cancels on plugin unload.

```typescript
ctx.interval(async () => {
    // This runs every 5 seconds (adjustable via config)
    const metrics = await collectMetrics();
    ctx.broadcast('metrics', hostname(), metrics);
}, 5000);
```

### Real example (Prometheus client)

```typescript
ctx.interval(async () => {
    try {
        const [gateway, name, pass] = SystemModel.getMany([
            'prom-client.gateway', 'prom-client.name', 'prom-client.password',
        ]);
        if (gateway) {
            const prefix = gateway.endsWith('/') ? gateway : `${gateway}/`;
            const endpoint = `${prefix}metrics/job/hydro-web/instance/${encodeURIComponent(hostname())}:${process.env.NODE_APP_INSTANCE}`;
            let req = superagent.post(endpoint);
            if (name) req = req.auth(name, pass, { type: 'basic' });
            await req.send(await registry.metrics());
        } else {
            ctx.broadcast('metrics', `${hostname()}/${process.env.NODE_APP_INSTANCE}`, await registry.getMetricsAsJSON());
        }
    } catch (e) {
        pushError = e.message;
    }
}, 5000 * (+SystemModel.get('prom-client.collect_rate') || 1));
```

### Real example (VJudge — weekly sync)

```typescript
this.ctx.interval(this.sync.bind(this), Time.week);
```

---

## 6. `ctx.provideModule()` — Replaceable Modules

Registers a named implementation for a module type. Multiple modules can coexist; the system or user selects which one to use.

### Currently supported module types

| Type | Interface | Purpose |
|------|-----------|---------|
| `hash` | `(password: string, salt: string, user: User) => boolean \| string \| Promise<string>` | Password hashing algorithm |
| `problemSearch` | `(domainId: string, q: string, opts?) => Promise<ProblemSearchResponse>` | Problem search backend |
| `richmedia` | `{ get(service, src, md) => string }` | Rich media rendering |

### Registration

```typescript
// Register a search backend
ctx.provideModule('problemSearch', 'elastic', async (domainId, q, opts) => {
    const limit = opts?.limit || 20;
    const result = await client.search({
        index: 'problem',
        body: { query: { multi_match: { query: q, fields: ['title', 'content'] } } },
        size: limit,
    });
    return {
        hits: result.hits.hits.map((h) => h._id),
        total: result.hits.total.value,
        countRelation: result.hits.total.relation,
    };
});
```

### Cleanup (auto via ctx.effect)

```typescript
// provideModule returns a dispose function
const dispose = ctx.provideModule('problemSearch', 'mybackend', mySearchFn);
// Auto-cleaned on plugin unload, or call dispose() manually
```

---

## 7. Generator-based event registration (using `yield`)

In Service constructors, you can use `yield this.ctx.on(...)` for cleaner lifecycle management:

```typescript
// Inside a service that uses generators
* [Context.init]() {
    yield this.ctx.on('problem/add', async (doc, docId) => { /* ... */ });
    yield this.ctx.on('problem/edit', async (pdoc) => { /* ... */ });
    yield this.ctx.provideModule('problemSearch', 'mysearch', this.search.bind(this));
}
```

The `yield` pattern ensures the returned dispose function is tracked by the Cordis framework and called automatically on disposal.

---

## 8. Event Flow Diagram

```
                          ┌─────────────┐
                          │   Plugin A  │
                          │  ctx.on()   │
                          └──────┬──────┘
                                 │
┌──────────────┐    emit     ┌───┴───────────────────┐
│  Core System │───────────► │  Cordis Event System  │
│  (models,    │             │                       │
│   handlers)  │    ◄─────── │  ctx.parallel()       │
└──────┬───────┘  broadcast  │  ctx.serial()         │
       │                      └───┬───────────────────┘
       │                          │
       │              ┌───────────┴───────────┐
       │              │                       │
       │        ┌─────┴─────┐          ┌──────┴──────┐
       │        │ Plugin B  │          │ Plugin C    │
       │        │ listener  │          │ listener    │
       │        └───────────┘          └─────────────┘
       │
       │   ctx.broadcast() for cross-process
       │          │
       │    ┌─────┴─────────────────────┐
       │    │  PM2 Bus / MongoDB Events │
       │    │  (cross-process relay)    │
       │    └───────────────────────────┘
```
