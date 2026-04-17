---
name: hydro-plugin-services
description: Hydro plugin service APIs including SettingService (plugin configuration), StorageService/StorageModel (file storage), TaskModel (background job queue), ScheduleModel (delayed task execution), and OAuth provider registration for third-party login.
---

# Hydro Plugin Development: Settings, Storage & OAuth Services

This skill covers service-level APIs for Hydro plugins: declaring plugin settings, file storage operations, and registering OAuth login providers.

---

## 1. Plugin Settings — SettingService

Plugins can declare configuration settings that appear in Hydro's admin panels. Settings are declared via schema functions exposed through `ctx.setting`.

### Setting scopes

| Method | Location | Who sees it |
|--------|----------|-------------|
| `ctx.setting.PreferenceSetting(schema)` | User → Preferences | Per-user settings |
| `ctx.setting.AccountSetting(schema)` | User → Account | Account-level settings |
| `ctx.setting.DomainSetting(schema)` | Domain → Settings | Per-domain settings |
| `ctx.setting.DomainUserSetting(schema)` | Domain → User Settings | Per-domain-user settings |
| `ctx.setting.SystemSetting(schema)` | System → Settings | System-wide (superadmin only) |

All methods accept a `Schema` object from Schemastery and auto-register cleanup on plugin unload.

### Declaring settings in a Service

```typescript
import { Context, Service, Schema } from 'hydrooj';

export default class MyPluginService extends Service {
    static Config = Schema.object({
        apiKey: Schema.string().description('API Key').required(),
        maxRetries: Schema.number().default(3).description('Max retry count'),
        endpoint: Schema.string().description('API Endpoint'),
        enabled: Schema.boolean().default(true),
    });

    constructor(ctx: Context, config: ReturnType<typeof MyPluginService.Config>) {
        super(ctx, 'myPlugin');
        // Access validated config
        // config.apiKey, config.maxRetries, etc.
    }
}
```

The `static Config` schema is automatically loaded by the Hydro Loader. Configuration values are read from the system config YAML stored in MongoDB, validated against the schema, and passed to the constructor.

Important caveat: this constructor injection assumes the service class is the plugin entry. If the module also exports a named `apply`, then `apply` becomes the entry point. In that mixed style, export `Config` at the module level, accept `config` in `apply`, and forward it with `ctx.plugin(MyPluginService, config)` if you still want the service constructor to receive the resolved config.

### Reading system settings

```typescript
// In any context with setting service injected
const value = ctx.setting.get('server.name');
const port = ctx.setting.get('server.port');

// Reading plugin-specific config in a Service:
// Use the config parameter from the constructor (see above)
```

### Setting types (`SettingType`)

```typescript
type SettingType =
    | 'text'        // Single-line text input
    | 'password'    // Password input (masked)
    | 'textarea'    // Multi-line text
    | 'markdown'    // Markdown editor
    | 'number'      // Integer input
    | 'float'       // Floating point input
    | 'boolean'     // Checkbox
    | 'json'        // JSON editor
    | 'yaml'        // YAML editor
    | [string, string][]    // Dropdown select (value-label pairs)
    | Record<string, string>; // Dropdown select (value-label map)
```

### Setting flags

```typescript
import { FLAG_HIDDEN, FLAG_DISABLED, FLAG_SECRET, FLAG_PRO, FLAG_PUBLIC, FLAG_PRIVATE } from 'hydrooj';

FLAG_HIDDEN   = 1   // Hidden from settings UI
FLAG_DISABLED = 2   // Visible but non-editable
FLAG_SECRET   = 4   // Stored encrypted, shown as password field
FLAG_PRO      = 8   // Only available in Hydro Pro
FLAG_PUBLIC   = 16  // Visible to all users
FLAG_PRIVATE  = 32  // Only visible to superadmins
```

### Real example: GitHub OAuth plugin settings

```typescript
export default class LoginWithGithubService extends Service {
    static inject = ['oauth'];
    static Config = Schema.object({
        id: Schema.string().description('GitHub OAuth AppID').required(),
        secret: Schema.string().description('GitHub OAuth Secret').role('secret').required(),
        endpoint: Schema.string().description('GitHub Endpoint'),
        canRegister: Schema.boolean().default(true),
    });

    constructor(ctx: Context, config: ReturnType<typeof LoginWithGithubService.Config>) {
        super(ctx, 'oauth.github');
        // config.id, config.secret, config.endpoint, config.canRegister
    }
}
```

---

## 2. File Storage — StorageModel

`StorageModel` provides a unified file storage API over either S3-compatible storage (RemoteStorageService) or local filesystem (LocalStorageService). Plugins should always use `StorageModel` rather than accessing the storage backend directly.

### Import

```typescript
import { StorageModel } from 'hydrooj';
// Also: global.Hydro.model.storage
```

### Core methods

```typescript
// Generate a unique storage ID with extension
const storageId = StorageModel.generateId('.pdf');
// Returns: "abc/1234567890abcdef.pdf"

// Upload a file
await StorageModel.put(
    path: string,                              // Storage path/key
    file: string | Buffer | Readable,          // File content or local path
    owner?: number,                            // Owner user ID
);

// Download a file
const stream: Readable = await StorageModel.get(
    path: string,        // Storage path
    savePath?: string,   // Optional: save to local path instead of returning stream
);

// Delete files
await StorageModel.del(
    path: string[],      // Array of paths to delete
    operator?: number,   // User ID performing the operation
);

// Check if a file exists
const exists: boolean = await StorageModel.exists(path: string);

// Get file metadata
const meta = await StorageModel.getMeta(path: string);
// Returns: { size, lastModified, etag, metaData, ... }

// List files in a directory
const files = await StorageModel.list(
    target: string,      // Directory path
    recursive?: boolean, // Default true
);
// Returns: Array<FileNode & { name: string }>

// Rename/move a file
await StorageModel.rename(
    path: string,       // Current path
    newPath: string,    // New path
    operator?: number,
);

// Move a file
await StorageModel.move(src: string, dst: string): Promise<boolean>;

// Copy a file
await StorageModel.copy(src: string, dst: string): Promise<string>;
```

### Signed download links

```typescript
// Generate a time-limited download URL
const url = await StorageModel.signDownloadLink(
    target: string,                        // File path
    filename?: string,                     // Override download filename
    noExpire?: boolean,                    // If true, link never expires
    useAlternativeEndpointFor?: 'user' | 'judge',  // Use alt endpoint
);
```

### Real example: problem testdata upload

```typescript
// Upload testdata for a problem
const fileId = StorageModel.generateId('.zip');
await StorageModel.put(`problem/${domainId}/${docId}/testdata/${fileId}`, buffer);

// Generate download link for judge
const link = await StorageModel.signDownloadLink(
    `problem/${domainId}/${docId}/testdata/${fileId}`,
    'testdata.zip',
);
```

### Real example: contest file attachment

```typescript
// Upload a contest attachment
const path = `contest/${domainId}/${tid}/files/${filename}`;
await StorageModel.put(path, file, this.user._id);

// List all contest files
const files = await StorageModel.list(`contest/${domainId}/${tid}/files`);
```

---

## 3. StorageService (Low-Level Backend)

The `StorageService` is the underlying storage backend. There are two implementations:

- **`RemoteStorageService`** — S3-compatible object storage (production)
- **`LocalStorageService`** — Local filesystem (development / simple deployments)

Plugins typically use `StorageModel` (higher-level) instead of accessing `StorageService` directly.

### Available when needed

```typescript
// The storage service is available via ctx.inject
// RemoteStorageService has direct S3 client access:
ctx.storage.client   // S3Client instance (only for RemoteStorageService)

// Upload with metadata
await ctx.storage.put(target: string, file: string | Buffer | Readable, meta?: Record<string, string>);

// Direct S3 signed upload (pre-signed POST)
const { url, fields } = await ctx.storage.signUpload(target: string, size: number);
// Client uploads directly to S3 using these credentials
```

---

## 4. OAuth Provider Registration

Plugins can register third-party OAuth login providers (e.g., GitHub, Google, custom providers). This makes the provider appear on Hydro's login page.

### OAuthProvider interface

```typescript
interface OAuthProvider {
    text: string;           // Button text (e.g., 'Login with GitHub')
    name: string;           // Provider display name (e.g., 'GitHub')
    icon?: string;          // SVG icon markup
    hidden?: boolean;       // If true, not shown on login page (programmatic only)
    canRegister?: boolean;  // Whether this provider can auto-register new users
    lockUsername?: boolean;  // Whether username is locked to OAuth profile

    // Step 1: Redirect user to OAuth provider
    get: (this: Handler) => Promise<void>;

    // Step 2: Handle callback, return user info
    callback: (this: Handler, args: Record<string, any>) => Promise<OAuthUserResponse>;
}

interface OAuthUserResponse {
    _id: string;                    // Unique user ID from the provider
    email: string;                  // User email (must be verified by provider)
    bio?: string;                   // User bio
    uname?: string[];               // Array of candidate usernames (first available is used)
    avatar?: string;                // Avatar identifier (e.g., 'github:username')
    viewLang?: string;              // Preferred language
    set?: Record<string, any>;      // Additional user fields to set
    setInDomain?: Record<string, any>;  // Domain-specific fields to set
}
```

### Registration pattern

```typescript
export default class MyOAuthService extends Service {
    static inject = ['oauth'];
    static Config = Schema.object({
        clientId: Schema.string().required(),
        clientSecret: Schema.string().role('secret').required(),
        canRegister: Schema.boolean().default(true),
    });

    constructor(ctx: Context, config: ReturnType<typeof MyOAuthService.Config>) {
        super(ctx, 'oauth.myprovider');

        ctx.oauth.provide('myprovider', {
            text: 'Login with MyProvider',
            name: 'MyProvider',
            icon: '<svg>...</svg>',
            canRegister: config.canRegister,

            // Redirect user to OAuth authorize URL
            get: async function get(this: Handler) {
                const [state] = await TokenModel.add(TokenModel.TYPE_OAUTH, 600, {
                    redirect: this.request.referer,
                });
                this.response.redirect =
                    `https://myprovider.com/oauth/authorize?client_id=${config.clientId}&state=${state}&scope=email`;
            },

            // Handle callback after user authorizes
            callback: async function callback({ state, code }) {
                // Validate state token
                const s = await TokenModel.get(state, TokenModel.TYPE_OAUTH);
                if (!s) throw new ValidationError('token');

                // Exchange code for access token
                const tokenRes = await superagent.post('https://myprovider.com/oauth/token')
                    .send({
                        client_id: config.clientId,
                        client_secret: config.clientSecret,
                        code,
                    })
                    .set('accept', 'application/json');

                const accessToken = tokenRes.body.access_token;

                // Fetch user info
                const userRes = await superagent.get('https://api.myprovider.com/user')
                    .set('Authorization', `Bearer ${accessToken}`);

                // Clean up state token
                await TokenModel.del(s._id, TokenModel.TYPE_OAUTH);

                return {
                    _id: userRes.body.id.toString(),
                    email: userRes.body.email,
                    uname: [userRes.body.name, userRes.body.username].filter(Boolean),
                    avatar: `myprovider:${userRes.body.username}`,
                    bio: userRes.body.bio,
                };
            },
        });
    }
}
```

### OAuth helper methods (from OauthModel)

```typescript
// Accessible via ctx.oauth

// Get the Hydro user ID linked to an OAuth account
const uid = await ctx.oauth.get(platform: string, id: string): Promise<number | null>;

// Link an OAuth account to a Hydro user
await ctx.oauth.set(platform: string, id: string, uid: number): Promise<number>;

// Unlink an OAuth account
await ctx.oauth.unbind(platform: string, uid: number): Promise<DeleteResult>;

// List all OAuth accounts for a user
const accounts = await ctx.oauth.list(uid: number): Promise<OauthMap[]>;
```

### Key security considerations

1. **Always validate the `state` parameter** using `TokenModel.get()` / `TokenModel.del()` to prevent CSRF
2. **Delete the state token** after successful callback to prevent replay attacks
3. **Require verified email** — throw `ForbiddenError` if no verified email is available
4. **Use `role('secret')`** for client_secret in Config schema to prevent leaking in API responses
5. **`lockUsername: true`** prevents users from changing their username after OAuth registration (useful for institutional SSO)

---

## 5. Background Tasks — `TaskModel`

`TaskModel` provides a persistent background job queue backed by MongoDB. Tasks survive process restarts and can be consumed by worker processes at a controlled concurrency.

### Import

```typescript
import { TaskModel } from 'hydrooj';
// Also available as global.Hydro.model.task
```

### Task document shape

```typescript
interface Task {
    _id: ObjectId;
    type: string;       // Your custom task type identifier
    subType?: string;   // Optional sub-categorization
    priority: number;   // Higher = processed first (default 0)
    [key: string]: any; // Any additional payload data
}
```

### Core methods

```typescript
// Enqueue a task
const taskId = await TaskModel.add({
    type: 'my_plugin/send_email',   // Required: task type
    subType: 'verification',        // Optional
    priority: 10,                   // Optional, default 0
    to: 'user@example.com',         // Any custom payload
    subject: 'Welcome',
});

// Enqueue multiple tasks at once
const taskIds = await TaskModel.addMany([
    { type: 'my_plugin/process', file: 'a.txt', priority: 0 },
    { type: 'my_plugin/process', file: 'b.txt', priority: 0 },
]);

// Get / count / delete
await TaskModel.get(taskId);
await TaskModel.count({ type: 'my_plugin/send_email' });
await TaskModel.del(taskId);
await TaskModel.deleteMany({ type: 'my_plugin/send_email' });
await TaskModel.getFirst({ type: 'my_plugin/send_email' });
```

### Consuming tasks (worker pattern)

`TaskModel.consume()` creates a long-running consumer that picks up tasks from the queue:

```typescript
export async function apply(ctx: Context) {
    const consumer = TaskModel.consume(
        { type: 'my_plugin/send_email' },  // Query filter
        async (task) => {
            // Process the task
            await sendEmail(task.to, task.subject);
            // Task is auto-deleted on success
        },
        true,   // destroyOnError: if true, failed tasks are deleted; if false, they're retried
        5,      // concurrency: how many tasks to process in parallel
    );

    // Cleanup on plugin unload
    ctx.effect(() => () => consumer.destroy());
}
```

### Consumer API

```typescript
class Consumer {
    consuming: boolean;           // Whether actively consuming
    processing: Set<Task>;        // Currently processing tasks

    async consume(): Promise<void>;   // Start consuming
    async destroy(): Promise<void>;   // Stop consuming
    setConcurrency(n: number): void;  // Adjust concurrency
    setQuery(query: any): void;       // Change the filter query
}
```

### Real example: VJudge submission queue

```typescript
// Producer: enqueue a judge task
await TaskModel.add({
    type: 'vjudge',
    subType: provider.type,
    priority: 1,
    rid,          // Record ID
    domainId,
    pid: provider.pid,
});

// Consumer: process judge tasks
const consumer = TaskModel.consume(
    { type: 'vjudge', subType: type },
    async (t) => {
        const result = await provider.judge(t);
        await record.updateResult(t.domainId, t.rid, result);
    },
    false,  // Don't destroy on error — allow retry
    3,      // Process 3 concurrently
);
```

---

## 6. Scheduled Tasks — `ScheduleModel`

`ScheduleModel` stores tasks that should execute after a specific time. The Hydro core scheduler checks for due tasks and executes them.

### Import

```typescript
import { ScheduleModel } from 'hydrooj';
// Also available as global.Hydro.model.schedule
```

### Schedule document shape

```typescript
interface Schedule {
    _id: ObjectId;
    type: string;           // Task type identifier
    subType?: string;       // Optional sub-categorization
    executeAfter: Date;     // Earliest execution time
    [key: string]: any;     // Any custom payload
}
```

### Core methods

```typescript
// Schedule a future task
const scheduleId = await ScheduleModel.add({
    type: 'my_plugin/notify',       // Required
    subType: 'deadline',            // Optional
    executeAfter: new Date(Date.now() + 3600_000),  // Run after 1 hour
    uid: 12345,                     // Custom payload
    message: 'Your homework is due',
});

// Get / count / delete
await ScheduleModel.get(scheduleId);
await ScheduleModel.count({ type: 'my_plugin/notify' });
await ScheduleModel.del(scheduleId);
await ScheduleModel.deleteMany({ type: 'my_plugin/notify' });
await ScheduleModel.getFirst({ type: 'my_plugin/notify' });
```

### How scheduling works

1. Plugin calls `ScheduleModel.add()` with `executeAfter` set to the desired time
2. Hydro's core scheduler periodically scans for schedules where `executeAfter <= now`
3. When a schedule is due, the system fires a `task/daily` or custom event based on the type
4. Your plugin listens for the event and processes the schedule:

```typescript
// Schedule: send reminder in 1 hour
await ScheduleModel.add({
    type: 'my_plugin/reminder',
    executeAfter: new Date(Date.now() + 3600_000),
    uid: userId,
    content: 'Don\'t forget!',
});

// Process: listen for due schedules
ctx.on('task/daily', async () => {
    const now = new Date();
    const due = await ScheduleModel.getFirst({
        type: 'my_plugin/reminder',
        executeAfter: { $lte: now },
    });
    if (due) {
        await sendNotification(due.uid, due.content);
        await ScheduleModel.del(due._id);
    }
});
```

### Real example: contest start notification

```typescript
// When contest is created, schedule notifications
ctx.on('contest/add', async (payload, tid) => {
    await ScheduleModel.add({
        type: 'contest/start_notify',
        executeAfter: payload.beginAt,
        tid,
        domainId: payload.domainId,
    });
});
```

---

## 7. Quick Reference

| Service | Access | Purpose |
|---------|--------|---------|
| Settings | `ctx.setting.{scope}Setting(schema)` | Declare config UI |
| Settings | `ctx.setting.get(key)` | Read config value |
| Storage | `StorageModel.put/get/del/list/signDownloadLink` | File operations |
| Storage | `StorageModel.generateId(ext)` | Unique file IDs |
| OAuth | `ctx.oauth.provide(name, provider)` | Register login provider |
| OAuth | `ctx.oauth.get/set/unbind/list` | Manage account links |
| Token | `TokenModel.add(type, expireSec, data)` | Create temp tokens |
| Token | `TokenModel.get(id, type)` / `TokenModel.del(id, type)` | Validate/clean tokens |
| Task | `TaskModel.add/addMany` | Enqueue background jobs |
| Task | `TaskModel.consume(query, handler, destroyOnError, concurrency)` | Worker process pattern |
| Task | `TaskModel.get/count/del/getFirst` | Query/manage tasks |
| Schedule | `ScheduleModel.add({ type, executeAfter, ... })` | Schedule delayed tasks |
| Schedule | `ScheduleModel.get/count/del/getFirst` | Query/manage schedules |
