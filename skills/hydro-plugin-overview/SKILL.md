---
name: hydro-plugin-overview
description: Hydro plugin fundamentals including entry functions (apply vs Service class), import conventions, plugin package structure, database access (ctx.db and deprecated db), management scripts (ctx.addScript), and a complete minimal working example.
---

# Hydro Plugin Development: Overview & Fundamentals

This skill covers the absolute basics of writing a Hydro plugin: the entry function, import conventions, plugin package structure, and a minimal working example.

---

## 1. Plugin Entry: `apply(ctx: Context)`

Every Hydro plugin must export either:
- An `apply(ctx: Context)` function (function-style, for simple plugins)
- A class extending `Service` (class-style, for stateful plugins with config/dependencies)

### Function-style (simple plugins)

```typescript
import { Context, Handler, param, Types, PRIV } from 'hydrooj';

class MyHandler extends Handler {
    @param('id', Types.String, true)
    async get(domainId: string, id?: string) {
        this.response.body = { message: 'Hello', id };
    }
}

export async function apply(ctx: Context) {
    ctx.Route('my_route', '/my/:id', MyHandler);
    ctx.on('problem/add', (doc, docId) => {
        console.log('Problem added:', docId);
    });
}
```

### Class-style (stateful plugins)

```typescript
import { Context, Service } from 'hydrooj';
import Schema from 'schemastery';

export default class MyService extends Service {
    // Declare service dependencies (Cordis auto-resolves initialization order)
    static inject = ['db'];

    // Declare configuration schema
    static Config = Schema.object({
        apiKey: Schema.string().required(),
        timeout: Schema.number().default(30),
    });

    constructor(ctx: Context, config: ReturnType<typeof MyService.Config>) {
        super(ctx, 'myService');
        // Access context via this.ctx
        // Access validated config via config parameter
        this.ctx.on('problem/add', (doc, docId) => {
            // Use this.ctx for all plugin operations
        });
    }
}
```

When using class-style, the `apply` function is implicitly the constructor — Cordis calls `ctx.plugin(MyService, config)` which instantiates the class.

---

## 2. Import Conventions

**ALWAYS import from `'hydrooj'`** — it is the unified public API surface. All public APIs are re-exported from `plugin-api.ts` inside the `hydrooj` package.

When developing a plugin as an external npm package, you install `hydrooj` as a dependency and import everything from it. You do NOT need access to the Hydro source repo — the `hydrooj` npm package contains all type definitions.

```typescript
// Correct ✅
import {
    Context, Service, Handler, ConnectionHandler,
    param, get, post, route, query, subscribe,
    Types, PRIV, PERM, STATUS,
    ObjectId, db,
    EventMap,
    UserFacingError,
    DocumentModel, ProblemModel, RecordModel,
    // ... etc
} from 'hydrooj';

// Wrong ❌ — never import from internal paths
import Handler from 'hydrooj/src/service/server';
import { ObjectId } from 'hydrooj/src/model/basic';
```

**Tip for AI assistants**: If you're unsure what's available from `hydrooj`, check `node_modules/hydrooj/dist/plugin-api.js` (or `.d.ts`) for the full list of exports. The key exported categories are:
- Core: `Context`, `Service`, `Handler`, `ConnectionHandler`
- Decorators: `param`, `get`, `post`, `route`, `query`, `subscribe`
- Validation: `Types`
- Permissions: `PERM`, `PRIV`, `STATUS`
- Database: `db`, `ObjectId`, `Collections`
- Models: `*Model` (UserModel, ProblemModel, RecordModel, DomainModel, ContestModel, DocumentModel, DiscussionModel, StorageModel, SettingModel, etc.)
- Errors: `UserFacingError` and various specific error classes
- Utilities: `_` (lodash), `nanoid`, `moment`
- Event types: `EventMap`

---

## 3. Plugin Package Structure

A plugin is an npm package.

### Minimal structure

```
my-plugin/
├── package.json       # Must have "name" and "main" fields
└── index.ts           # Default export: Service class OR named export: apply()
```

### Full structure (with templates and i18n)

```
my-plugin/
├── package.json
├── index.ts           # Backend entry
├── templates/         # Nunjucks HTML templates (auto-discovered by TemplateService)
│   ├── my_page.html
│   └── my_detail.html
└── locales/           # Optional: separate i18n files
```

### package.json requirements

```json
{
    "name": "hydro-plugin-my-feature",
    "main": "index.ts",
    "version": "1.0.0",
    "dependencies": {
        "hydrooj": "*"
    }
}
```

The plugin name is used for:
- Version tracking (`global.Hydro.version[name]`)
- Module path resolution (`global.addons[name]`)
- Config scoping in the database

---

## 4. Complete Minimal Example: Blog Plugin

This is a real plugin from `packages/blog/index.ts`, simplified:

```typescript
import {
    _, Context, DiscussionNotFoundError, DocumentModel,
    Handler, ObjectId, OplogModel, param, PRIV, Types, UserModel,
} from 'hydrooj';

// --- Define a new document type ---
export const TYPE_BLOG = 70 as const;
export interface BlogDoc {
    docType: 70;
    docId: ObjectId;
    owner: number;
    title: string;
    content: string;
    views: number;
}
declare module 'hydrooj' {
    interface Model { blog: typeof BlogModel; }
    interface DocType { [TYPE_BLOG]: BlogDoc; }
}

// --- Define a Model (data access layer) ---
export class BlogModel {
    static async add(owner: number, title: string, content: string, ip?: string) {
        const res = await DocumentModel.add(
            'system', content, owner, TYPE_BLOG, null, null, null,
            _.omit({ title, ip, nReply: 0, views: 0, updateAt: new Date() }, ['domainId']),
        );
        return res;
    }
    static async get(did: ObjectId) {
        return await DocumentModel.get('system', TYPE_BLOG, did);
    }
    static edit(did: ObjectId, title: string, content: string) {
        return DocumentModel.set('system', TYPE_BLOG, did, { title, content });
    }
    static del(did: ObjectId) {
        return DocumentModel.deleteOne('system', TYPE_BLOG, did);
    }
    static getMulti(query = {}) {
        return DocumentModel.getMulti('system', TYPE_BLOG, query).sort({ _id: -1 });
    }
}
global.Hydro.model.blog = BlogModel;

// --- Define Handlers ---
class BlogUserHandler extends Handler {
    @param('uid', Types.Int)
    @param('page', Types.PositiveInt, true)
    async get(domainId: string, uid: number, page = 1) {
        const [ddocs, dpcount] = await this.ctx.db.paginate(
            BlogModel.getMulti({ owner: uid }), page, 10,
        );
        const udoc = await UserModel.getById(domainId, uid);
        this.response.template = 'blog_main.html';
        this.response.body = { ddocs, dpcount, udoc, page };
    }
}

class BlogEditHandler extends Handler {
    @param('title', Types.Title)
    @param('content', Types.Content)
    async postCreate(domainId: string, title: string, content: string) {
        await this.limitRate('add_blog', 3600, 60);
        const did = await BlogModel.add(this.user._id, title, content, this.request.ip);
        this.response.redirect = this.url('blog_detail', { uid: this.user._id, did });
    }
}

// --- Plugin entry ---
export async function apply(ctx: Context) {
    // Register routes
    ctx.Route('blog_main', '/blog/:uid', BlogUserHandler);
    ctx.Route('blog_create', '/blog/:uid/create', BlogEditHandler, PRIV.PRIV_USER_PROFILE);

    // Register UI injection
    ctx.injectUI('UserDropdown', 'blog_main',
        (h) => ({ icon: 'book', displayName: 'Blog', uid: h.user._id.toString() }),
        PRIV.PRIV_USER_PROFILE,
    );

    // Register i18n
    ctx.i18n.load('zh', {
        "{0}'s blog": '{0} 的博客',
        Blog: '博客',
    });
    ctx.i18n.load('en', {
        blog_main: 'Blog',
    });
}
```

And the corresponding template `templates/blog_main.html`:
```html
{% extends "layout/basic.html" %}
{% block content %}
<div class="row">
  <div class="medium-9 columns">
    <div class="section">
      <ol class="section__list">
        {%- for ddoc in ddocs -%}
          <li class="section__list__item">
            <h1><a href="{{ url('blog_detail', uid=udoc._id, did=ddoc._id) }}">{{ ddoc.title }}</a></h1>
          </li>
        {%- endfor -%}
      </ol>
    </div>
  </div>
</div>
{% endblock %}
```

---

## 5. Database Access (`ctx.db` / `db`)

The `db` service provides MongoDB access. There are two ways to use it:

### Via `ctx.db` (recommended)

Available as `this.ctx.db` in handlers or `ctx.db` in services:

```typescript
// Get a typed collection reference
ctx.db.collection<K extends keyof Collections>(name: K): Collection<Collections[K]>

// Pagination — returns [docs, numPages, totalCount]
const [docs, numPages, count] = await ctx.db.paginate(cursor, page, pageSize);

// Ranked pagination (with equal-rank grouping)
const [ranked, docs] = await ctx.db.ranked(cursor, (a, b) => a.score === b.score);

// Index management
await ctx.db.ensureIndexes(collection, ...indexDescriptions);
await ctx.db.clearIndexes(collection, dropIndexNames?);
```

### Via `import { db } from 'hydrooj'` (deprecated but widely used)

```typescript
import { db } from 'hydrooj';

// db is a Proxy that forwards all property access to app.get('db')
// So you can use it anywhere without a context:
const coll = db.collection('user');
const docs = await coll.find({}).toArray();
```

> **Note**: This `db` export is marked `@deprecated` — it accesses the MongoService via a global proxy (`app.get('db')`). Despite being deprecated, it is still widely used throughout the Hydro ecosystem and all existing plugins. Prefer `ctx.db` in new code, but you will encounter `db` frequently in existing codebases.

### Accessing raw MongoDB

```typescript
// Via ctx.db
ctx.db.client  // MongoClient instance
ctx.db.db      // native Db instance (from mongodb package)

// Via deprecated import
db.client
db.db
```

### Common patterns

```typescript
// Paginated query in a handler
const [items, pageCount, count] = await this.ctx.db.paginate(
    MyModel.getMulti(domainId, { status: 'active' }),
    page,
    20,
);

// Using deprecated db import in a model
import { db } from 'hydrooj';
const coll = db.collection('my_collection');
await coll.updateOne({ _id: oid }, { $set: { field: value } });
```

---

## 6. Management Scripts (`ctx.addScript`)

Scripts appear in the **System Control Panel → Scripts** page. They are admin-only tools for batch operations, data migration, or maintenance tasks.

### Registration

```typescript
ctx.addScript(
    name: string,          // Unique script identifier
    description: string,   // Human-readable description shown in UI
    validate: Schema<T>,   // Schemastery schema for script arguments
    run: (args: T, report: (msg: string) => void) => boolean | Promise<boolean>,
);
```

The `run` callback receives:
- `args` — validated input matching the schema
- `report` — progress callback, shows messages in the UI during execution

Return `true` to indicate success, or throw an error for failure.

### Example: batch cleanup script

```typescript
import { Context, Schema, iterateAllProblem } from 'hydrooj';

export async function apply(ctx: Context) {
    ctx.addScript('clean_orphan_data', 'Remove orphan data from problems',
        Schema.object({
            dryRun: Schema.boolean().default(true).description('Only report, do not delete'),
        }),
        async (args, report) => {
            let count = 0;
            await iterateAllProblem(['domainId', 'docId', 'title'], async (pdoc, i, total) => {
                if (i % 100 === 0) report(`Processing ${i}/${total}...`);
                const hasOrphan = checkOrphan(pdoc);
                if (hasOrphan) {
                    count++;
                    if (!args.dryRun) await cleanupOrphan(pdoc);
                }
            });
            report(`Found ${count} problems with orphan data`);
            return true;
        },
    );
}
```

### Built-in iteration helpers (from `hydrooj`)

```typescript
import { iterateAllDomain, iterateAllUser, iterateAllContest,
         iterateAllProblem, iterateAllRecord, iterateAllPsdoc } from 'hydrooj';

// Iterate all domains
await iterateAllDomain(async (ddoc, current?, total?) => { ... });

// Iterate all users
await iterateAllUser(async (udoc, current?, total?) => { ... });

// Iterate all problems across all domains
await iterateAllProblem(['title', 'content'], async (pdoc, current?, total?) => {
    // Return a partial update to auto-apply changes:
    return { title: pdoc.title.trim() };
});
```

---

## 7. Key Concepts Summary

| Concept | API | Purpose |
|---------|-----|---------|
| Entry | `apply(ctx)` or `default class extends Service` | Plugin bootstrap |
| Routes | `ctx.Route(name, path, Handler, ...perms)` | HTTP endpoints |
| Events | `ctx.on(event, handler)` | React to system events |
| UI Injection | `ctx.injectUI(position, name, args, ...perms)` | Add UI elements |
| Modules | `ctx.provideModule(type, id, impl)` | Register replaceable modules |
| Scripts | `ctx.addScript(name, desc, schema, run)` | Admin batch operations |
| Config | `static Config = Schema.object({...})` | Plugin settings |
| i18n | `ctx.i18n.load(lang, dict)` | Translations |
| DB Access | `ctx.db.collection()`, `ctx.db.paginate()` | MongoDB operations |
| Templates | `templates/*.html` directory | Nunjucks server-rendered HTML |

See also: `hydro-plugin-handler` for routes/handlers, `hydro-plugin-hooks` for events, `hydro-plugin-frontend` for templates/UI/i18n, `hydro-plugin-services` for storage/settings/oauth.
