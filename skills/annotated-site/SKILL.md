---
name: annotated-site
description: "Use when scaffolding a new Vercel-hosted, password-gated site for collaborator review of one or more markdown documents — produces a Next.js app with selection-anchored annotations, replies, resolve flag, per-document overall comments, and a script that pulls annotations back into the repo as markdown for git tracking. Triggers on: \"set up a review site\", \"make a site for collaborators to comment\", \"annotated site\", \"spin up a Vercel review site\"."
---

# annotated-site

## Purpose

Scaffold a lean Vercel-hosted, password-gated review site for one or more markdown documents in the user's repo. Each invocation produces ~1100 lines of Next.js 16 code tailored to the project at hand: selection-anchored annotations, replies, resolve flag, per-document overall comments, and a download script that pulls collaborator feedback back into the repo as markdown for git tracking.

Pattern is proven (it mirrors a site used in an earlier project) with three small extensions: replies, per-doc overall comments, and a `document_id`-keyed schema so doc renames don't orphan rows.

## When to invoke

Trigger phrases: "set up a review site", "make a site for collaborators to comment", "annotated site", "spin up a Vercel review site".

## Prerequisites

Verify before scaffolding (run as a single shell pipeline; abort with the specific install/auth instruction on the first failure):

```bash
which node && node --version | grep -qE 'v(20|22|24)' \
  && which vercel \
  && vercel whoami \
  && git rev-parse --is-inside-work-tree
```

On failure, print the targeted instruction:
- missing node → "install Node 22 LTS via nvm: `nvm install 22 && nvm use 22`"
- missing vercel → "install: `npm install -g vercel`"
- not logged in → "run: `vercel login`"
- not a git repo → "this skill only runs inside a git repo; `git init` first"

## Workflow

Each generated file is reproduced verbatim under "Generated files" below. **Write each file using a single `Write` call; paste the verbatim block from the "Generated files" section below — do not paraphrase or generate from scratch.** Project-specific values (slugs, paths, password, document_ids) are interpolated into the otherwise verbatim contents at the marked spots. Filling in those marked spots is the only edit you make.

1. **Verify prerequisites** as listed in the Prerequisites section.
2. **Discover content.** Glob the repo for `.md` files (top-level + obvious subdirs like `docs/`, `analysis_plan/`, `results/`). Present them via `AskUserQuestion` with multi-select; allow the user to type custom paths.
3. **Gather config** via `AskUserQuestion`:
   - `site_dir` (default `./site/`).
   - `vercel_project_name` (default = repo directory name).
   - `password` (auto-generate 8-char alphanumeric via `node -e "console.log(require('crypto').randomBytes(6).toString('base64url'))"` or accept user input).
   - `annotations_dir` (default `./annotations/`).
4. **Generate `documents.config.json`.** For each user-selected file, allocate one stable `document_id` of the form `doc_<14-char nanoid>`. Emit one `{document_id, slug, path, title}` quad per doc. `slug` is the filename stem with non-`[a-z0-9-]` chars replaced by `-`. `title` defaults to a Title Case of the slug; user may edit. `path` is the relative path from `<site_dir>/` to the source markdown. Write `<site_dir>/documents.config.json` first so subsequent files can reference it.
5. **Scaffold the Next.js app.** Walk the "Generated files" section in order; for each `### File:` block, create the file with a single `Write` call pasting the block contents verbatim, substituting project-specific values only at the explicit `__VAR__` markers. The full file contents are inline — do not abbreviate, paraphrase, or substitute "similar" code from elsewhere.
6. **Vercel + Neon.**
   - `cd <site_dir> && vercel link` — link to existing project if detected (e.g. by repo name match), else create new with name `<vercel_project_name>`.
   - Verify: `cat .vercel/project.json | grep projectName` shows the expected project. Abort if not.
   - Detect Neon vars: `vercel env ls production | grep -E 'POSTGRES_URL|DATABASE_URL'`. If absent, prompt user to run `vercel integration add neon` interactively (browser OAuth) and wait for confirmation that env vars now appear.
   - Set the password in all envs: `for ENV in production preview development; do printf "%s" "<password>" | vercel env add SITE_PASSWORD $ENV --yes; done`.
7. **Local build smoke-test.** `cd <site_dir> && npm install && npm run build`. Fail loudly on TypeScript or build errors; do not proceed.
8. **Deploy.** From `<site_dir>` (verify cwd is not a subdirectory): `vercel deploy --prod --yes`. Capture the printed URL.
9. **Authenticated curl verify.** Each step must pass before continuing.

   ```bash
   URL=<vercel_url>
   PW=<password>
   # login
   curl -sS -c /tmp/site-cookies.txt -X POST "$URL/api/login" \
     -H 'Content-Type: application/json' \
     -d "{\"password\":\"$PW\",\"name\":\"verify-script\"}" \
     -o /tmp/site-login.json -w '%{http_code}\n'   # expect 200
   # protected root with cookies
   curl -sS -b /tmp/site-cookies.txt "$URL/" -o /dev/null -w '%{http_code}\n'  # expect 200 or 307→/docs/<slug>
   # first doc page
   curl -sS -b /tmp/site-cookies.txt "$URL/docs/<first-slug>" -o /tmp/site-doc.html -w '%{http_code}\n'  # expect 200
   grep -q "<first-title>" /tmp/site-doc.html && echo OK || echo MISSING_TITLE
   # post a temp annotation against the first doc
   FIRST_DOC_ID=$(node -e "console.log(require('<site_dir>/documents.config.json')[0].document_id)")
   curl -sS -b /tmp/site-cookies.txt -X POST "$URL/api/annotations" \
     -H 'Content-Type: application/json' \
     -d "{\"documentId\":\"$FIRST_DOC_ID\",\"quote\":\"verification probe text\",\"prefix\":\"\",\"suffix\":\"\",\"startOffset\":0,\"endOffset\":24,\"body\":\"verify\"}" \
     -w '%{http_code}\n'   # expect 200
   ```

10. **Persist scaffold config.** Write `<repo>/.annotated-site.json`:

    ```json
    {
      "site_dir": "<site_dir>",
      "vercel_url": "<vercel_url>",
      "password": "<password>",
      "annotations_dir": "<annotations_dir>",
      "documents": [/* same array as documents.config.json */]
    }
    ```

11. **Report.** Print the dashboard URL + password + the one-line download command, e.g.:

    ```
    Site:     https://<project>.vercel.app
    Password: <password>
    Download: SITE_URL=https://<project>.vercel.app SITE_PASSWORD=<password> node <site_dir>/scripts/download-annotations.mjs
    Suggest sharing the URL + password with collaborators. Run the download command before commits to capture annotations as markdown under <annotations_dir>/.
    ```

### Configuration intake

`AskUserQuestion` payload (one question, four sub-prompts as separate questions if the harness requires single-fact questions):

- "Site directory (relative to repo root)?" default `./site/`
- "Vercel project name?" default = `basename $(git rev-parse --show-toplevel)`
- "Password (leave blank to auto-generate)?" default = auto
- "Annotations directory for git-tracked downloaded annotations?" default `./annotations/`

### Discovery

```bash
fd -e md -d 3 . | grep -vE '^\./(site|node_modules|.next|annotations|.git)/' | head -50
```

(or `find . -maxdepth 3 -name '*.md' -not -path './site/*' -not -path './node_modules/*' -not -path './.git/*' | head -50` if `fd` is unavailable)

Present the list via `AskUserQuestion` multi-select. Add a free-form "other paths (one per line)" question for files not surfaced by the glob.

## Generated files

Each block below is the **complete contents** of one file. Write each as a single `Write` tool call. Do not abbreviate, paraphrase, or substitute "similar" code. The only edits permitted are:
- replace `__SITE_TITLE__` with a human title for the site (default = repo name in title case)

Anything else is verbatim.

### File: package.json

Standard Next 16 + React 19 dependencies pinned by major.

```json
{
  "name": "annotated-site",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "sync-content": "node scripts/sync-content.mjs",
    "predev": "npm run sync-content",
    "prebuild": "npm run sync-content",
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "pull-annotations": "node scripts/download-annotations.mjs"
  },
  "dependencies": {
    "@neondatabase/serverless": "^0.10.0",
    "nanoid": "^5.0.0",
    "next": "^16.0.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "react-markdown": "^10.0.0",
    "rehype-highlight": "^7.0.0",
    "rehype-raw": "^7.0.0",
    "remark-gfm": "^4.0.0",
    "highlight.js": "^11.10.0"
  },
  "devDependencies": {
    "@tailwindcss/postcss": "^4.0.0",
    "@types/node": "^22.0.0",
    "@types/react": "^19.0.0",
    "@types/react-dom": "^19.0.0",
    "autoprefixer": "^10.4.0",
    "postcss": "^8.4.0",
    "tailwindcss": "^4.0.0",
    "typescript": "^5.7.0"
  }
}
```

### File: next.config.ts

Default Next 16 config; expand here only if a project needs custom redirects.

```ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {};

export default nextConfig;
```

### File: tsconfig.json

Strict TS with the Next plugin and `@/*` path alias.

```json
{
  "compilerOptions": {
    "target": "ES2017",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "react-jsx",
    "incremental": true,
    "plugins": [{ "name": "next" }],
    "paths": { "@/*": ["./*"] }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts", "**/*.mts"],
  "exclude": ["node_modules"]
}
```

### File: postcss.config.mjs

Tailwind 4 PostCSS pipeline.

```js
const config = {
  plugins: {
    "@tailwindcss/postcss": {},
  },
};

export default config;
```

### File: proxy.ts

Taken unchanged from the earlier project's `proxy.ts`. Auth gate: any non-`/login`, non-`/api/login` request without the `site_auth` cookie is redirected to `/login?from=…`.

```ts
import { NextRequest, NextResponse } from "next/server";

const COOKIE_NAME = "site_auth";
const PUBLIC_ROUTES = ["/login", "/api/login"];

export const config = {
  matcher: ["/((?!_next/static|_next/image|favicon.ico).*)"],
};

export function proxy(req: NextRequest) {
  const { pathname } = req.nextUrl;

  if (
    PUBLIC_ROUTES.some(
      (p) => pathname === p || pathname.startsWith(p + "/"),
    )
  ) {
    return NextResponse.next();
  }

  const cookie = req.cookies.get(COOKIE_NAME);
  const expected = process.env.SITE_PASSWORD || "annotated_site_dev";

  if (!cookie || cookie.value !== expected) {
    const url = req.nextUrl.clone();
    url.pathname = "/login";
    url.searchParams.set("from", pathname);
    return NextResponse.redirect(url);
  }

  return NextResponse.next();
}
```

### File: lib/normalize.ts

Three-line normalizer applied to `quote / prefix / suffix` at save and at fallback search.

```ts
export function normalizeText(s: string): string {
  return s.normalize("NFC").replace(/\s+/g, " ").trim();
}
```

### File: lib/db.ts

Neon serverless client + `ensureSchema` covering all three tables and their indexes. The Neon HTTP driver does not support sessions or transactions (each `await sql\`…\`` is a separate HTTP request), so atomicity is provided by idempotent `CREATE … IF NOT EXISTS` plus a process-level `schemaReady` memoization guard that prevents repeated bootstrap runs.

```ts
import { neon } from "@neondatabase/serverless";

const url =
  process.env.DATABASE_URL ||
  process.env.POSTGRES_URL ||
  process.env.POSTGRES_URL_NON_POOLING;

if (!url) {
  console.warn(
    "[db] no Neon URL set (DATABASE_URL / POSTGRES_URL). API routes will return warnings until `vercel integration add neon` is run.",
  );
}

export const sql = url ? neon(url) : null;

let schemaReady = false;

export async function ensureSchema() {
  if (!sql || schemaReady) return;
  await sql`
    CREATE TABLE IF NOT EXISTS annotations (
      id            TEXT PRIMARY KEY,
      document_id   TEXT NOT NULL,
      author_name   TEXT NOT NULL,
      quote         TEXT NOT NULL,
      prefix        TEXT,
      suffix        TEXT,
      start_offset  INTEGER NOT NULL,
      end_offset    INTEGER NOT NULL,
      body          TEXT NOT NULL,
      resolved      BOOLEAN NOT NULL DEFAULT FALSE,
      created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
    )
  `;
  await sql`
    CREATE TABLE IF NOT EXISTS replies (
      id              TEXT PRIMARY KEY,
      annotation_id   TEXT NOT NULL REFERENCES annotations(id) ON DELETE CASCADE,
      author_name     TEXT NOT NULL,
      body            TEXT NOT NULL,
      created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
    )
  `;
  await sql`
    CREATE TABLE IF NOT EXISTS comments (
      id            TEXT PRIMARY KEY,
      document_id   TEXT NOT NULL,
      author_name   TEXT NOT NULL,
      body          TEXT NOT NULL,
      created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
    )
  `;
  await sql`CREATE INDEX IF NOT EXISTS annotations_doc_idx    ON annotations(document_id)`;
  await sql`CREATE INDEX IF NOT EXISTS replies_annotation_idx ON replies(annotation_id)`;
  await sql`CREATE INDEX IF NOT EXISTS comments_doc_idx       ON comments(document_id)`;
  schemaReady = true;
}
```

### File: lib/documents.ts

Reads `documents.config.json`. `getDocBySlug` and `getDocById` helpers used throughout the app.

```ts
import { readFileSync } from "node:fs";
import { join } from "node:path";

export type DocumentMeta = {
  document_id: string;
  slug: string;
  path: string;
  title: string;
};

let cache: DocumentMeta[] | null = null;

export function loadDocuments(): DocumentMeta[] {
  if (cache) return cache;
  const raw = readFileSync(
    join(process.cwd(), "documents.config.json"),
    "utf8",
  );
  cache = JSON.parse(raw) as DocumentMeta[];
  return cache;
}

export function getDocBySlug(slug: string): DocumentMeta | undefined {
  return loadDocuments().find((d) => d.slug === slug);
}

export function getDocById(documentId: string): DocumentMeta | undefined {
  return loadDocuments().find((d) => d.document_id === documentId);
}
```

### File: scripts/sync-content.mjs

Pre-build copy of source markdown into `public/docs/<slug>.md`. Iterates `documents.config.json`. Tolerant of missing source files inside a Vercel build context (will use the previously synced `public/docs/<slug>.md` if present).

```js
import {
  copyFileSync,
  existsSync,
  mkdirSync,
  readFileSync,
  statSync,
  writeFileSync,
} from "node:fs";
import { join, dirname, isAbsolute, resolve } from "node:path";
import { fileURLToPath } from "node:url";

const here = dirname(fileURLToPath(import.meta.url));
const siteRoot = join(here, "..");
const publicDocsDir = join(siteRoot, "public", "docs");
const config = JSON.parse(
  readFileSync(join(siteRoot, "documents.config.json"), "utf8"),
);

mkdirSync(publicDocsDir, { recursive: true });

for (const doc of config) {
  const src = isAbsolute(doc.path) ? doc.path : resolve(siteRoot, doc.path);
  const dst = join(publicDocsDir, `${doc.slug}.md`);
  if (existsSync(src)) {
    const text = readFileSync(src, "utf8");
    writeFileSync(dst, text, "utf8");
    console.log(
      `[sync-content] copied ${doc.slug}: ${src} -> ${dst} (${statSync(dst).size} bytes)`,
    );
  } else if (existsSync(dst)) {
    console.log(
      `[sync-content] source missing (${src}); using existing ${dst} (${statSync(dst).size} bytes) — typical inside Vercel build context`,
    );
  } else {
    console.error(
      `[sync-content] FATAL: neither source (${src}) nor destination (${dst}) exists for slug "${doc.slug}"`,
    );
    process.exit(1);
  }
}
```

### File: scripts/download-annotations.mjs

Adapted from the earlier project's `scripts/download-annotations.mjs`. Iterates `documents.config.json`, POSTs `/api/login`, GETs `/api/annotations?documentId=…` and `/api/comments?documentId=…` for each doc, writes `<annotations_dir>/<slug>.md` with the canonical template.

```js
import {
  writeFileSync,
  mkdirSync,
  readFileSync,
  existsSync,
} from "node:fs";
import { join, dirname, resolve } from "node:path";
import { fileURLToPath } from "node:url";

const here = dirname(fileURLToPath(import.meta.url));
const siteRoot = join(here, "..");
const repoRoot = resolve(siteRoot, "..");

const SITE_URL = (process.env.SITE_URL || "http://localhost:3000").replace(/\/$/, "");
const SITE_PASSWORD = process.env.SITE_PASSWORD;
const ANNOTATIONS_DIR = process.env.ANNOTATIONS_DIR
  ? resolve(process.env.ANNOTATIONS_DIR)
  : (() => {
      // Read .annotated-site.json from repoRoot if present.
      const cfgPath = join(repoRoot, ".annotated-site.json");
      if (existsSync(cfgPath)) {
        const cfg = JSON.parse(readFileSync(cfgPath, "utf8"));
        return resolve(repoRoot, cfg.annotations_dir || "annotations");
      }
      return resolve(repoRoot, "annotations");
    })();

if (!SITE_PASSWORD) {
  console.error("[download-annotations] SITE_PASSWORD env var is required.");
  process.exit(2);
}

const docs = JSON.parse(
  readFileSync(join(siteRoot, "documents.config.json"), "utf8"),
);
const requestedSlug = process.argv[2];
const targets = requestedSlug
  ? docs.filter((d) => d.slug === requestedSlug)
  : docs;
if (targets.length === 0) {
  console.error(`[download-annotations] no doc found for slug "${requestedSlug}"`);
  process.exit(2);
}

// Login
const loginRes = await fetch(`${SITE_URL}/api/login`, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ password: SITE_PASSWORD, name: "download-script" }),
});
if (!loginRes.ok) {
  console.error(`[download-annotations] login failed (${loginRes.status}):`, await loginRes.text());
  process.exit(3);
}
const cookies = loginRes.headers.getSetCookie();
const cookieHeader = cookies.map((c) => c.split(";")[0]).filter(Boolean).join("; ");

mkdirSync(ANNOTATIONS_DIR, { recursive: true });

for (const doc of targets) {
  const aRes = await fetch(
    `${SITE_URL}/api/annotations?documentId=${encodeURIComponent(doc.document_id)}`,
    { headers: { cookie: cookieHeader } },
  );
  const cRes = await fetch(
    `${SITE_URL}/api/comments?documentId=${encodeURIComponent(doc.document_id)}`,
    { headers: { cookie: cookieHeader } },
  );
  if (!aRes.ok) {
    console.error(`[download-annotations] GET annotations ${doc.slug} failed:`, await aRes.text());
    process.exit(4);
  }
  if (!cRes.ok) {
    console.error(`[download-annotations] GET comments ${doc.slug} failed:`, await cRes.text());
    process.exit(4);
  }
  const { annotations = [] } = await aRes.json();
  const { comments = [] } = await cRes.json();

  const now = new Date().toISOString();
  const lines = [];
  lines.push(`# Annotations on ${doc.title}`);
  lines.push("");
  lines.push(`Document: \`${doc.slug}\` (id: \`${doc.document_id}\`)`);
  lines.push(`Pulled: ${now} from ${SITE_URL}`);
  lines.push("");
  lines.push(`Total: ${annotations.length} annotation(s), ${comments.length} overall comment(s).`);
  lines.push("");
  lines.push("---");
  lines.push("");
  lines.push("## Overall comments");
  lines.push("");
  if (comments.length === 0) {
    lines.push("_(none yet)_");
    lines.push("");
  } else {
    comments.forEach((c, i) => {
      lines.push(`### ${i + 1}. ${c.author_name} — ${c.created_at}`);
      lines.push("");
      lines.push(c.body || "");
      lines.push("");
    });
  }
  lines.push("---");
  lines.push("");
  lines.push("## Inline annotations");
  lines.push("");
  if (annotations.length === 0) {
    lines.push("_(none yet)_");
    lines.push("");
  }
  annotations.forEach((a, i) => {
    const status = a.resolved ? "RESOLVED" : "open";
    lines.push(`### ${i + 1}. ${a.author_name} (char ${a.start_offset}-${a.end_offset}) [${status}]`);
    lines.push("");
    lines.push(`_Created: ${a.created_at}_`);
    lines.push("");
    lines.push("**Quoted passage:**");
    lines.push("");
    lines.push("> " + String(a.quote).replace(/\n/g, "\n> "));
    lines.push("");
    if (a.prefix) {
      lines.push(`*…prefix: "…${a.prefix}"*`);
      lines.push("");
    }
    if (a.suffix) {
      lines.push(`*…suffix: "${a.suffix}…"*`);
      lines.push("");
    }
    lines.push("**Comment:**");
    lines.push("");
    lines.push(a.body || "");
    lines.push("");
    if (a.replies && a.replies.length > 0) {
      lines.push("**Replies:**");
      lines.push("");
      a.replies.forEach((r) => {
        lines.push(`- _${r.author_name}, ${r.created_at}:_ ${r.body}`);
      });
      lines.push("");
    }
    lines.push("---");
    lines.push("");
  });
  const outPath = join(ANNOTATIONS_DIR, `${doc.slug}.md`);
  writeFileSync(outPath, lines.join("\n"), "utf8");
  console.log(
    `[download-annotations] wrote ${annotations.length} annotations + ${comments.length} comments for ${doc.slug} -> ${outPath}`,
  );
}
```

### File: app/api/login/route.ts

POST `{password, name}` → sets `site_auth` (httpOnly) and `site_user` (client-readable). 401 on password mismatch.

```ts
import { NextRequest, NextResponse } from "next/server";

const COOKIE_NAME = "site_auth";
const NAME_COOKIE = "site_user";

export async function POST(req: NextRequest) {
  const { password, name } = await req.json();
  const expected = process.env.SITE_PASSWORD || "annotated_site_dev";
  if (typeof password !== "string" || password !== expected) {
    return NextResponse.json({ ok: false }, { status: 401 });
  }
  const res = NextResponse.json({ ok: true });
  const thirtyDays = 60 * 60 * 24 * 30;
  res.cookies.set(COOKIE_NAME, expected, {
    httpOnly: true,
    sameSite: "lax",
    secure: true,
    path: "/",
    maxAge: thirtyDays,
  });
  res.cookies.set(NAME_COOKIE, String(name || "Anonymous").slice(0, 60), {
    httpOnly: false,
    sameSite: "lax",
    secure: true,
    path: "/",
    maxAge: thirtyDays,
  });
  return res;
}
```

### File: app/api/annotations/route.ts

GET (returns annotations with replies joined inline), POST (rejects `quote.length < 5` after normalization), PATCH (resolved toggle), DELETE (cookie-author check).

```ts
import { NextRequest, NextResponse } from "next/server";
import { nanoid } from "nanoid";
import { sql, ensureSchema } from "@/lib/db";
import { normalizeText } from "@/lib/normalize";

export const runtime = "nodejs";

function readCookie(req: NextRequest, name: string): string {
  return req.cookies.get(name)?.value || "";
}

export async function GET(req: NextRequest) {
  if (!sql) return NextResponse.json({ annotations: [], warning: "DATABASE_URL not configured" });
  await ensureSchema();
  const documentId = new URL(req.url).searchParams.get("documentId");
  if (!documentId) return NextResponse.json({ error: "missing documentId" }, { status: 400 });
  const annotations = await sql`
    SELECT id, document_id, author_name, quote, prefix, suffix,
           start_offset, end_offset, body, resolved, created_at
      FROM annotations
     WHERE document_id = ${documentId}
     ORDER BY created_at ASC
  `;
  const ids = annotations.map((a: { id: string }) => a.id);
  let replies: Array<{ id: string; annotation_id: string; author_name: string; body: string; created_at: string }> = [];
  if (ids.length > 0) {
    replies = (await sql`
      SELECT id, annotation_id, author_name, body, created_at
        FROM replies
       WHERE annotation_id = ANY(${ids})
       ORDER BY created_at ASC
    `) as typeof replies;
  }
  const grouped = new Map<string, typeof replies>();
  for (const r of replies) {
    const arr = grouped.get(r.annotation_id) || [];
    arr.push(r);
    grouped.set(r.annotation_id, arr);
  }
  const out = annotations.map((a: { id: string }) => ({
    ...a,
    replies: grouped.get(a.id) || [],
  }));
  return NextResponse.json({ annotations: out });
}

export async function POST(req: NextRequest) {
  if (!sql) return NextResponse.json({ error: "no db" }, { status: 503 });
  await ensureSchema();
  const author = readCookie(req, "site_user") || "Anonymous";
  const body = await req.json();
  const required = ["documentId", "quote", "startOffset", "endOffset", "body"];
  for (const k of required) {
    if (body[k] === undefined || body[k] === null || body[k] === "") {
      return NextResponse.json({ error: `missing field: ${k}` }, { status: 400 });
    }
  }
  const quote = normalizeText(String(body.quote));
  if (quote.length < 5) {
    return NextResponse.json(
      { error: "quote too short (min 5 chars after normalization)" },
      { status: 400 },
    );
  }
  if (quote.length > 800) {
    return NextResponse.json({ error: "quote too long (max 800)" }, { status: 400 });
  }
  const prefix = body.prefix ? normalizeText(String(body.prefix)).slice(0, 80) : null;
  const suffix = body.suffix ? normalizeText(String(body.suffix)).slice(0, 80) : null;
  const id = nanoid(12);
  await sql`
    INSERT INTO annotations
      (id, document_id, author_name, quote, prefix, suffix,
       start_offset, end_offset, body, resolved)
    VALUES
      (${id}, ${String(body.documentId)},
       ${author.slice(0, 60)},
       ${quote},
       ${prefix}, ${suffix},
       ${Number(body.startOffset)}, ${Number(body.endOffset)},
       ${String(body.body).slice(0, 5000)},
       ${false})
  `;
  return NextResponse.json({ ok: true, id });
}

export async function PATCH(req: NextRequest) {
  if (!sql) return NextResponse.json({ error: "no db" }, { status: 503 });
  await ensureSchema();
  const { id, resolved } = await req.json();
  if (!id) return NextResponse.json({ error: "missing id" }, { status: 400 });
  await sql`UPDATE annotations SET resolved = ${!!resolved} WHERE id = ${id}`;
  return NextResponse.json({ ok: true });
}

export async function DELETE(req: NextRequest) {
  if (!sql) return NextResponse.json({ error: "no db" }, { status: 503 });
  await ensureSchema();
  const id = new URL(req.url).searchParams.get("id");
  if (!id) return NextResponse.json({ error: "missing id" }, { status: 400 });
  const author = readCookie(req, "site_user") || "Anonymous";
  const rows = await sql`SELECT author_name FROM annotations WHERE id = ${id}`;
  if (rows.length === 0) return NextResponse.json({ error: "not found" }, { status: 404 });
  if (rows[0].author_name !== author) {
    return NextResponse.json({ error: "forbidden" }, { status: 403 });
  }
  await sql`DELETE FROM annotations WHERE id = ${id}`;
  return NextResponse.json({ ok: true });
}
```

### File: app/api/replies/route.ts

POST `{annotationId, body}`, DELETE `?id=…` (cookie-author check).

```ts
import { NextRequest, NextResponse } from "next/server";
import { nanoid } from "nanoid";
import { sql, ensureSchema } from "@/lib/db";

export const runtime = "nodejs";

function readCookie(req: NextRequest, name: string): string {
  return req.cookies.get(name)?.value || "";
}

export async function POST(req: NextRequest) {
  if (!sql) return NextResponse.json({ error: "no db" }, { status: 503 });
  await ensureSchema();
  const author = readCookie(req, "site_user") || "Anonymous";
  const { annotationId, body } = await req.json();
  if (!annotationId || !body) {
    return NextResponse.json({ error: "missing annotationId or body" }, { status: 400 });
  }
  const id = nanoid(12);
  await sql`
    INSERT INTO replies (id, annotation_id, author_name, body)
    VALUES (${id}, ${String(annotationId)}, ${author.slice(0, 60)}, ${String(body).slice(0, 5000)})
  `;
  return NextResponse.json({ ok: true, id });
}

export async function DELETE(req: NextRequest) {
  if (!sql) return NextResponse.json({ error: "no db" }, { status: 503 });
  await ensureSchema();
  const id = new URL(req.url).searchParams.get("id");
  if (!id) return NextResponse.json({ error: "missing id" }, { status: 400 });
  const author = readCookie(req, "site_user") || "Anonymous";
  const rows = await sql`SELECT author_name FROM replies WHERE id = ${id}`;
  if (rows.length === 0) return NextResponse.json({ error: "not found" }, { status: 404 });
  if (rows[0].author_name !== author) {
    return NextResponse.json({ error: "forbidden" }, { status: 403 });
  }
  await sql`DELETE FROM replies WHERE id = ${id}`;
  return NextResponse.json({ ok: true });
}
```

### File: app/api/comments/route.ts

GET `?documentId=…`, POST `{documentId, body}`, DELETE `?id=…` (cookie-author check).

```ts
import { NextRequest, NextResponse } from "next/server";
import { nanoid } from "nanoid";
import { sql, ensureSchema } from "@/lib/db";

export const runtime = "nodejs";

function readCookie(req: NextRequest, name: string): string {
  return req.cookies.get(name)?.value || "";
}

export async function GET(req: NextRequest) {
  if (!sql) return NextResponse.json({ comments: [], warning: "no db" });
  await ensureSchema();
  const documentId = new URL(req.url).searchParams.get("documentId");
  if (!documentId) return NextResponse.json({ error: "missing documentId" }, { status: 400 });
  const rows = await sql`
    SELECT id, document_id, author_name, body, created_at
      FROM comments
     WHERE document_id = ${documentId}
     ORDER BY created_at ASC
  `;
  return NextResponse.json({ comments: rows });
}

export async function POST(req: NextRequest) {
  if (!sql) return NextResponse.json({ error: "no db" }, { status: 503 });
  await ensureSchema();
  const author = readCookie(req, "site_user") || "Anonymous";
  const { documentId, body } = await req.json();
  if (!documentId || !body) {
    return NextResponse.json({ error: "missing documentId or body" }, { status: 400 });
  }
  const id = nanoid(12);
  await sql`
    INSERT INTO comments (id, document_id, author_name, body)
    VALUES (${id}, ${String(documentId)}, ${author.slice(0, 60)}, ${String(body).slice(0, 5000)})
  `;
  return NextResponse.json({ ok: true, id });
}

export async function DELETE(req: NextRequest) {
  if (!sql) return NextResponse.json({ error: "no db" }, { status: 503 });
  await ensureSchema();
  const id = new URL(req.url).searchParams.get("id");
  if (!id) return NextResponse.json({ error: "missing id" }, { status: 400 });
  const author = readCookie(req, "site_user") || "Anonymous";
  const rows = await sql`SELECT author_name FROM comments WHERE id = ${id}`;
  if (rows.length === 0) return NextResponse.json({ error: "not found" }, { status: 404 });
  if (rows[0].author_name !== author) {
    return NextResponse.json({ error: "forbidden" }, { status: 403 });
  }
  await sql`DELETE FROM comments WHERE id = ${id}`;
  return NextResponse.json({ ok: true });
}
```

### File: app/layout.tsx

Root layout with NavBar.

```tsx
import type { Metadata } from "next";
import "./globals.css";
import NavBar from "@/components/NavBar";

export const metadata: Metadata = {
  title: "__SITE_TITLE__ · Review",
  description: "Password-gated collaborator annotation site.",
};

export default function RootLayout({
  children,
}: Readonly<{ children: React.ReactNode }>) {
  return (
    <html lang="en" className="h-full antialiased">
      <body className="min-h-full flex flex-col">
        <NavBar />
        {children}
      </body>
    </html>
  );
}
```

### File: app/globals.css

Tailwind 4 imports + minimal `.prose` tweaks for the rendered markdown. Also imports highlight.js for code blocks.

```css
@import "tailwindcss";
@import "highlight.js/styles/github.css";

:root {
  --background: #ffffff;
  --foreground: #171717;
}

body {
  background: var(--background);
  color: var(--foreground);
  font-family: ui-sans-serif, system-ui, -apple-system, sans-serif;
}

.prose h1 { font-size: 1.75rem; font-weight: 600; margin: 2rem 0 1rem; line-height: 1.25; }
.prose h2 { font-size: 1.5rem;  font-weight: 600; margin: 2rem 0 0.75rem; line-height: 1.3; border-top: 1px solid #e5e5e5; padding-top: 1.25rem; }
.prose h3 { font-size: 1.2rem;  font-weight: 600; margin: 1.5rem 0 0.5rem; }
.prose h4 { font-size: 1rem;    font-weight: 600; margin: 1.25rem 0 0.5rem; }
.prose p  { margin: 0.75rem 0; }
.prose ul { list-style: disc; padding-left: 1.25rem; margin: 0.75rem 0; }
.prose ol { list-style: decimal; padding-left: 1.5rem; margin: 0.75rem 0; }
.prose li { margin: 0.25rem 0; }
.prose a  { color: #2563eb; text-decoration: underline; }
.prose blockquote { border-left: 3px solid #d4d4d4; color: #525252; padding: 0.25rem 0 0.25rem 1rem; margin: 0.75rem 0; }
.prose code { background: #f5f5f5; padding: 0.1rem 0.3rem; border-radius: 3px; font-size: 0.85em; }
.prose pre  { background: #fafafa; border: 1px solid #e5e5e5; padding: 0.75rem; border-radius: 6px; font-size: 0.8rem; overflow-x: auto; }
.prose pre code { background: transparent; padding: 0; }
.prose table { border-collapse: collapse; margin: 1rem 0; font-size: 0.85rem; }
.prose th, .prose td { border: 1px solid #e5e5e5; padding: 0.4rem 0.6rem; text-align: left; vertical-align: top; }
.prose th { background: #f9fafb; font-weight: 600; }
.prose hr { border: 0; border-top: 1px solid #e5e5e5; margin: 2rem 0; }
.prose img { display: block; max-width: 100%; height: auto; margin: 1rem auto; }
.prose mark { background: #fde68a; padding: 0 1px; border-radius: 2px; cursor: pointer; }
.annotation-flash { animation: card-flash 1.4s ease-in-out 1; }
@keyframes card-flash { 0% { box-shadow: 0 0 0 2px #f59e0b; } 100% { box-shadow: 0 0 0 0 transparent; } }
```

### File: app/page.tsx

Server component; redirect to first doc slug.

```tsx
import { redirect } from "next/navigation";
import { loadDocuments } from "@/lib/documents";

export default function Home() {
  const docs = loadDocuments();
  if (docs.length === 0) {
    return (
      <main className="mx-auto max-w-md px-6 py-24 text-sm text-neutral-700">
        <h1 className="text-xl font-semibold">No documents configured</h1>
        <p className="mt-2">
          Add entries to <code>documents.config.json</code> and redeploy.
        </p>
      </main>
    );
  }
  redirect(`/docs/${docs[0].slug}`);
}
```

### File: app/login/page.tsx

Login form: name + password + the verbatim trust-model disclosure.

```tsx
"use client";

import { useState, Suspense } from "react";
import { useRouter, useSearchParams } from "next/navigation";

function LoginForm() {
  const router = useRouter();
  const sp = useSearchParams();
  const from = sp.get("from") || "/";
  const [password, setPassword] = useState("");
  const [name, setName] = useState("");
  const [error, setError] = useState<string | null>(null);
  const [submitting, setSubmitting] = useState(false);

  async function onSubmit(e: React.FormEvent) {
    e.preventDefault();
    setSubmitting(true);
    setError(null);
    const res = await fetch("/api/login", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ password, name }),
    });
    if (!res.ok) {
      setError("Wrong password.");
      setSubmitting(false);
      return;
    }
    router.replace(from);
  }

  return (
    <main className="mx-auto max-w-md px-6 py-24">
      <h1 className="text-2xl font-semibold tracking-tight">Review site login</h1>
      <p className="mt-2 text-sm text-neutral-600">
        This review site is password-protected. Ask the site owner for access.
      </p>
      <form onSubmit={onSubmit} className="mt-8 space-y-4">
        <label className="block">
          <span className="text-sm font-medium">Your name</span>
          <input
            value={name}
            onChange={(e) => setName(e.target.value)}
            className="mt-1 block w-full rounded border border-neutral-300 px-3 py-2 text-sm"
            placeholder="e.g., Alex Chen"
            required
          />
          <span className="mt-2 block text-xs leading-relaxed text-neutral-500">
            Your name is used as a label on your annotations. Identity is not authenticated — anyone with the password could enter any name. Please use your real name in good faith.
          </span>
        </label>
        <label className="block">
          <span className="text-sm font-medium">Password</span>
          <input
            type="password"
            value={password}
            onChange={(e) => setPassword(e.target.value)}
            className="mt-1 block w-full rounded border border-neutral-300 px-3 py-2 text-sm"
            required
          />
        </label>
        {error && (
          <p className="text-sm text-red-600" role="alert">
            {error}
          </p>
        )}
        <button
          type="submit"
          disabled={submitting}
          className="w-full rounded bg-neutral-900 px-4 py-2 text-sm font-medium text-white disabled:opacity-60"
        >
          {submitting ? "Checking…" : "Enter"}
        </button>
      </form>
    </main>
  );
}

export default function LoginPage() {
  return (
    <Suspense fallback={null}>
      <LoginForm />
    </Suspense>
  );
}
```

### File: app/docs/[slug]/page.tsx

Server component. `getDocBySlug`, reads `public/docs/<slug>.md`, mounts `OverallComments` + `DocAnnotator`.

```tsx
import { readFileSync } from "node:fs";
import { join } from "node:path";
import { notFound } from "next/navigation";
import { getDocBySlug, loadDocuments } from "@/lib/documents";
import DocAnnotator from "@/components/DocAnnotator";
import OverallComments from "@/components/OverallComments";

export function generateStaticParams() {
  return loadDocuments().map((d) => ({ slug: d.slug }));
}

export default async function DocPage({
  params,
}: {
  params: Promise<{ slug: string }>;
}) {
  const { slug } = await params;
  const doc = getDocBySlug(slug);
  if (!doc) return notFound();
  const text = readFileSync(
    join(process.cwd(), "public", "docs", `${doc.slug}.md`),
    "utf8",
  );
  return (
    <div>
      <OverallComments documentId={doc.document_id} title={doc.title} />
      <DocAnnotator
        markdown={text}
        documentId={doc.document_id}
        title={doc.title}
        slug={doc.slug}
      />
    </div>
  );
}
```

### File: components/NavBar.tsx

Top bar: site title + doc switcher + "signed in as <name>" + tooltip.

```tsx
"use client";

import Link from "next/link";
import { usePathname } from "next/navigation";
import { useEffect, useState } from "react";
import type { DocumentMeta } from "@/lib/documents";

function readCookie(name: string): string {
  if (typeof document === "undefined") return "";
  const c = document.cookie.split("; ").find((r) => r.startsWith(name + "="));
  return c ? decodeURIComponent(c.split("=").slice(1).join("=")) : "";
}

export default function NavBar({ docs }: { docs?: DocumentMeta[] } = {}) {
  const pathname = usePathname();
  const [user, setUser] = useState("");
  const [allDocs, setAllDocs] = useState<DocumentMeta[]>(docs || []);

  useEffect(() => {
    setUser(readCookie("site_user") || "");
    if (!docs) {
      fetch("/api/docs", { cache: "no-store" })
        .then((r) => (r.ok ? r.json() : { docs: [] }))
        .then((d) => setAllDocs(d.docs || []))
        .catch(() => setAllDocs([]));
    }
  }, [docs]);

  if (pathname === "/login") return null;

  return (
    <nav className="sticky top-0 z-10 border-b border-neutral-200 bg-white/90 backdrop-blur">
      <div className="mx-auto flex max-w-[1400px] flex-wrap items-center gap-x-6 gap-y-2 px-6 py-3">
        <Link href="/" className="text-sm font-semibold tracking-tight">
          Review
        </Link>
        <ul className="flex flex-wrap items-center gap-1">
          {allDocs.map((d) => {
            const href = `/docs/${d.slug}`;
            const active = pathname === href;
            return (
              <li key={d.slug}>
                <Link
                  href={href}
                  className={
                    "rounded px-3 py-1.5 text-xs font-medium transition-colors " +
                    (active
                      ? "bg-neutral-900 text-white"
                      : "text-neutral-700 hover:bg-neutral-100")
                  }
                >
                  {d.title}
                </Link>
              </li>
            );
          })}
        </ul>
        <span
          className="ml-auto text-xs text-neutral-600"
          title="this name is self-declared, not verified"
        >
          {user ? `signed in as ${user}` : ""}
        </span>
      </div>
    </nav>
  );
}
```

Also generate this small companion route so NavBar can list docs without duplicating server logic:

### File: app/api/docs/route.ts

GET → `{docs: DocumentMeta[]}`. Client-side helper for the NavBar.

```ts
import { NextResponse } from "next/server";
import { loadDocuments } from "@/lib/documents";

export const runtime = "nodejs";

export async function GET() {
  const docs = loadDocuments().map(({ document_id, slug, title }) => ({
    document_id,
    slug,
    title,
  }));
  return NextResponse.json({ docs });
}
```

### File: components/OverallComments.tsx

Per-doc free-text comments above the annotated content. Collapsed by default once non-empty; always-expanded with a one-line example when empty.

```tsx
"use client";

import { useCallback, useEffect, useMemo, useRef, useState } from "react";

type Comment = {
  id: string;
  document_id: string;
  author_name: string;
  body: string;
  created_at: string;
};

function readCookie(name: string): string {
  if (typeof document === "undefined") return "";
  const c = document.cookie.split("; ").find((r) => r.startsWith(name + "="));
  return c ? decodeURIComponent(c.split("=").slice(1).join("=")) : "";
}

export default function OverallComments({
  documentId,
  title,
}: {
  documentId: string;
  title: string;
}) {
  const [comments, setComments] = useState<Comment[]>([]);
  const [expanded, setExpanded] = useState(true);
  const [draft, setDraft] = useState("");
  const [submitting, setSubmitting] = useState(false);
  const author = useMemo(() => readCookie("site_user") || "Anonymous", []);
  const initialLoad = useRef(true);

  const load = useCallback(async () => {
    const res = await fetch(
      `/api/comments?documentId=${encodeURIComponent(documentId)}`,
      { cache: "no-store" },
    );
    if (!res.ok) return;
    const data = await res.json();
    const list: Comment[] = data.comments || [];
    setComments(list);
    if (initialLoad.current && list.length > 0) {
      setExpanded(false);
    }
    initialLoad.current = false;
  }, [documentId]);

  useEffect(() => {
    load();
  }, [load]);

  async function submit() {
    if (!draft.trim()) return;
    setSubmitting(true);
    const res = await fetch("/api/comments", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ documentId, body: draft }),
    });
    if (!res.ok) {
      console.error("post comment failed:", res.status, await res.text());
      setSubmitting(false);
      return;
    }
    setDraft("");
    setSubmitting(false);
    await load();
  }

  async function remove(c: Comment) {
    if (c.author_name !== author) return;
    if (!confirm("Delete this overall comment?")) return;
    const res = await fetch(`/api/comments?id=${c.id}`, { method: "DELETE" });
    if (!res.ok) {
      console.error("delete comment failed:", res.status, await res.text());
      return;
    }
    await load();
  }

  return (
    <section className="mx-auto max-w-[1400px] px-6 pt-4">
      <div className="rounded border border-neutral-200 bg-neutral-50 p-3">
        <header className="flex items-center justify-between">
          <h2 className="text-sm font-semibold">
            Overall comments on {title} ({comments.length})
          </h2>
          {comments.length > 0 && (
            <button
              onClick={() => setExpanded((v) => !v)}
              className="text-xs text-neutral-600 hover:underline"
            >
              {expanded ? "Hide" : "Show"}
            </button>
          )}
        </header>
        {expanded && (
          <div className="mt-3 space-y-3">
            {comments.length === 0 && (
              <p className="text-xs text-neutral-500">
                No overall comments yet. Use this space for whole-document feedback (e.g., "the conclusion needs a stronger framing of the limitations").
              </p>
            )}
            {comments.map((c) => (
              <div key={c.id} className="rounded border border-neutral-200 bg-white p-3 text-sm">
                <div className="flex items-baseline justify-between">
                  <p className="text-xs font-medium">{c.author_name}</p>
                  <p className="text-[10px] text-neutral-500">
                    {new Date(c.created_at).toLocaleString()}
                  </p>
                </div>
                <p className="mt-1 whitespace-pre-wrap">{c.body}</p>
                {c.author_name === author && (
                  <button
                    onClick={() => remove(c)}
                    className="mt-2 text-xs text-neutral-500 hover:underline"
                  >
                    Delete
                  </button>
                )}
              </div>
            ))}
            <div>
              <textarea
                value={draft}
                onChange={(e) => setDraft(e.target.value)}
                placeholder="Whole-document feedback…"
                rows={3}
                className="block w-full rounded border border-neutral-300 p-2 text-sm"
              />
              <button
                onClick={submit}
                disabled={!draft.trim() || submitting}
                className="mt-2 rounded bg-neutral-900 px-3 py-1 text-xs font-medium text-white disabled:opacity-60"
              >
                {submitting ? "Posting…" : "Post comment"}
              </button>
            </div>
          </div>
        )}
      </div>
    </section>
  );
}
```

### File: components/DocAnnotator.tsx

THE BIG ONE. Adapted from the earlier project's `components/PlanAnnotator.tsx`. Changes vs reference:

- `planVersion` prop → `documentId`.
- API calls use `documentId` (camelCase).
- Each annotation card has a Reply button + inline form; replies render below the parent indented in chronological order (POSTs `/api/replies`).
- Clicking a `<mark>` scrolls the matching card in the right panel into view + adds a `flash` animation.
- Rejects `quote.length < 5` with an inline error.
- The offset-into-concatenated-text walker (lines 82–122 of the reference) is preserved verbatim. It works.

```tsx
"use client";

import { useCallback, useEffect, useMemo, useRef, useState } from "react";
import ReactMarkdown from "react-markdown";
import remarkGfm from "remark-gfm";
import rehypeRaw from "rehype-raw";
import rehypeHighlight from "rehype-highlight";

type Reply = {
  id: string;
  annotation_id: string;
  author_name: string;
  body: string;
  created_at: string;
};

type Annotation = {
  id: string;
  document_id: string;
  author_name: string;
  quote: string;
  prefix: string | null;
  suffix: string | null;
  start_offset: number;
  end_offset: number;
  body: string;
  resolved: boolean;
  created_at: string;
  replies: Reply[];
};

function readCookie(name: string): string {
  if (typeof document === "undefined") return "";
  const c = document.cookie.split("; ").find((r) => r.startsWith(name + "="));
  return c ? decodeURIComponent(c.split("=").slice(1).join("=")) : "";
}

function normalize(s: string): string {
  return s.normalize("NFC").replace(/\s+/g, " ").trim();
}

export default function DocAnnotator({
  markdown,
  documentId,
  title,
  slug,
}: {
  markdown: string;
  documentId: string;
  title: string;
  slug: string;
}) {
  const [annotations, setAnnotations] = useState<Annotation[]>([]);
  const [pending, setPending] = useState<null | {
    quote: string;
    prefix: string;
    suffix: string;
    startOffset: number;
    endOffset: number;
  }>(null);
  const [pendingError, setPendingError] = useState<string | null>(null);
  const [draftBody, setDraftBody] = useState("");
  const [replyTo, setReplyTo] = useState<string | null>(null);
  const [replyDraft, setReplyDraft] = useState("");
  const [loading, setLoading] = useState(true);
  const [showResolved, setShowResolved] = useState(false);
  const author = useMemo(() => readCookie("site_user") || "Anonymous", []);
  const contentRef = useRef<HTMLDivElement | null>(null);
  const cardRefs = useRef<Map<string, HTMLLIElement>>(new Map());

  const loadAnnotations = useCallback(async () => {
    const res = await fetch(
      `/api/annotations?documentId=${encodeURIComponent(documentId)}`,
      { cache: "no-store" },
    );
    const data = await res.json();
    setAnnotations(data.annotations || []);
    setLoading(false);
  }, [documentId]);

  useEffect(() => {
    loadAnnotations();
  }, [loadAnnotations]);

  // Apply <mark> highlights based on annotation offsets.
  useEffect(() => {
    const container = contentRef.current;
    if (!container) return;
    // Strip prior marks.
    container.querySelectorAll("mark[data-aid]").forEach((m) => {
      const parent = m.parentNode;
      if (!parent) return;
      while (m.firstChild) parent.insertBefore(m.firstChild, m);
      parent.removeChild(m);
      parent.normalize();
    });
    if (annotations.length === 0) return;
    const walker = document.createTreeWalker(container, NodeFilter.SHOW_TEXT);
    const nodes: Text[] = [];
    let n = walker.nextNode();
    while (n) {
      nodes.push(n as Text);
      n = walker.nextNode();
    }
    const sorted = [...annotations].sort((a, b) => b.start_offset - a.start_offset);
    for (const a of sorted) {
      let acc = 0;
      let startNode: Text | null = null;
      let startInner = 0;
      let endNode: Text | null = null;
      let endInner = 0;
      for (const t of nodes) {
        const len = (t.textContent || "").length;
        if (!startNode && a.start_offset >= acc && a.start_offset < acc + len) {
          startNode = t;
          startInner = a.start_offset - acc;
        }
        if (!endNode && a.end_offset > acc && a.end_offset <= acc + len) {
          endNode = t;
          endInner = a.end_offset - acc;
        }
        acc += len;
        if (startNode && endNode) break;
      }
      if (!startNode || !endNode) continue;
      try {
        const range = document.createRange();
        range.setStart(startNode, startInner);
        range.setEnd(endNode, endInner);
        const mark = document.createElement("mark");
        mark.setAttribute("data-aid", a.id);
        if (a.resolved) mark.style.opacity = "0.5";
        range.surroundContents(mark);
      } catch {
        // Range spans element boundaries (e.g. across a <strong>); skip silently.
      }
    }
    // Click-mark-to-scroll wiring.
    const onClick = (e: Event) => {
      const t = e.target as HTMLElement;
      const m = t.closest("mark[data-aid]") as HTMLElement | null;
      if (!m) return;
      const id = m.getAttribute("data-aid");
      if (!id) return;
      const card = cardRefs.current.get(id);
      if (card) {
        card.scrollIntoView({ behavior: "smooth", block: "center" });
        card.classList.add("annotation-flash");
        setTimeout(() => card.classList.remove("annotation-flash"), 1500);
      }
    };
    container.addEventListener("click", onClick);
    return () => container.removeEventListener("click", onClick);
  }, [annotations, slug]);

  const onMouseUp = useCallback(() => {
    const sel = window.getSelection();
    if (!sel || sel.isCollapsed || !contentRef.current) return;
    const quoteRaw = sel.toString();
    if (!quoteRaw.trim()) return;
    if (quoteRaw.length > 800) {
      setPendingError("Selection too long (max 800 chars).");
      return;
    }

    const range = sel.getRangeAt(0);
    const container = contentRef.current;
    if (!container.contains(range.commonAncestorContainer)) return;

    // Reject selections fully inside an existing <mark>.
    const ancestor = range.commonAncestorContainer.nodeType === 1
      ? (range.commonAncestorContainer as Element)
      : (range.commonAncestorContainer.parentElement as Element | null);
    if (ancestor && ancestor.closest("mark[data-aid]")) return;

    // Walk all text nodes; concatenated-text offset model.
    const walker = document.createTreeWalker(container, NodeFilter.SHOW_TEXT);
    const nodes: Text[] = [];
    let n = walker.nextNode();
    while (n) {
      nodes.push(n as Text);
      n = walker.nextNode();
    }

    const offsetWithinConcat = (node: Node, nodeOffset: number): number => {
      let acc = 0;
      for (const t of nodes) {
        if (t === node) return acc + nodeOffset;
        acc += (t.textContent || "").length;
      }
      return -1;
    };

    let startOffset = offsetWithinConcat(range.startContainer, range.startOffset);
    let endOffset = offsetWithinConcat(range.endContainer, range.endOffset);

    if (startOffset < 0 || endOffset < 0) {
      const concat = nodes.map((t) => t.textContent || "").join("");
      const first = concat.indexOf(quoteRaw);
      const last = concat.lastIndexOf(quoteRaw);
      if (first >= 0 && first === last) {
        startOffset = first;
        endOffset = first + quoteRaw.length;
      } else {
        setPendingError(
          "Could not uniquely locate the selected passage. Try selecting a slightly longer span (or one that doesn't span across a table or code block).",
        );
        return;
      }
    }

    const concat = nodes.map((t) => t.textContent || "").join("");
    const prefix = concat.slice(Math.max(0, startOffset - 80), startOffset);
    const suffix = concat.slice(endOffset, Math.min(concat.length, endOffset + 80));

    const normalizedQuote = normalize(quoteRaw);
    if (normalizedQuote.length < 5) {
      setPendingError("Selection too short (min 5 chars after whitespace normalization).");
      return;
    }

    setPendingError(null);
    setPending({ quote: quoteRaw, prefix, suffix, startOffset, endOffset });
    setDraftBody("");
  }, []);

  async function saveAnnotation() {
    if (!pending || !draftBody.trim()) return;
    const res = await fetch("/api/annotations", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        documentId,
        quote: pending.quote,
        prefix: pending.prefix,
        suffix: pending.suffix,
        startOffset: pending.startOffset,
        endOffset: pending.endOffset,
        body: draftBody,
      }),
    });
    if (!res.ok) {
      setPendingError("Failed to save: " + (await res.text()));
      return;
    }
    setPending(null);
    setDraftBody("");
    setPendingError(null);
    await loadAnnotations();
  }

  async function toggleResolve(a: Annotation) {
    const res = await fetch("/api/annotations", {
      method: "PATCH",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ id: a.id, resolved: !a.resolved }),
    });
    if (!res.ok) {
      console.error("toggle resolve failed:", res.status, await res.text());
      return;
    }
    await loadAnnotations();
  }

  async function remove(a: Annotation) {
    if (!confirm("Delete this annotation?")) return;
    const res = await fetch(`/api/annotations?id=${a.id}`, { method: "DELETE" });
    if (!res.ok) {
      alert(await res.text());
      return;
    }
    await loadAnnotations();
  }

  async function submitReply(a: Annotation) {
    if (!replyDraft.trim()) return;
    const res = await fetch("/api/replies", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ annotationId: a.id, body: replyDraft }),
    });
    if (!res.ok) {
      console.error("submit reply failed:", res.status, await res.text());
      return;
    }
    setReplyTo(null);
    setReplyDraft("");
    await loadAnnotations();
  }

  async function removeReply(r: Reply) {
    if (!confirm("Delete this reply?")) return;
    const res = await fetch(`/api/replies?id=${r.id}`, { method: "DELETE" });
    if (!res.ok) {
      console.error("delete reply failed:", res.status, await res.text());
      return;
    }
    await loadAnnotations();
  }

  const visible = annotations.filter((a) => showResolved || !a.resolved);

  return (
    <div className="mx-auto grid max-w-[1400px] grid-cols-1 gap-6 px-6 py-6 lg:grid-cols-[1fr_380px]">
      <section>
        <header className="mb-4">
          <p className="text-xs uppercase tracking-wider text-neutral-500">
            {slug} · signed in as {author}
          </p>
          <h1 className="text-xl font-semibold">{title}</h1>
          <p className="mt-1 text-xs text-neutral-500">
            Select any passage (≥ 5 chars) to add an annotation.
          </p>
        </header>
        <div
          ref={contentRef}
          onMouseUp={onMouseUp}
          className="prose prose-neutral max-w-none text-sm leading-relaxed [&_h1]:mt-0 [&_pre]:overflow-x-auto [&_table]:block [&_table]:overflow-x-auto"
        >
          <ReactMarkdown
            remarkPlugins={[remarkGfm]}
            rehypePlugins={[rehypeRaw, rehypeHighlight]}
          >
            {markdown}
          </ReactMarkdown>
        </div>
      </section>

      <aside className="lg:sticky lg:top-4 lg:self-start lg:max-h-[calc(100vh-2rem)] lg:overflow-y-auto">
        <div className="rounded border border-neutral-200 bg-white">
          <header className="flex items-center justify-between border-b border-neutral-200 px-4 py-3">
            <h2 className="text-sm font-semibold">
              Annotations ({annotations.filter((a) => !a.resolved).length} open)
            </h2>
            <label className="flex items-center gap-1 text-xs text-neutral-600">
              <input
                type="checkbox"
                checked={showResolved}
                onChange={(e) => setShowResolved(e.target.checked)}
              />
              show resolved
            </label>
          </header>

          {pendingError && (
            <div className="border-b border-neutral-200 bg-red-50 p-3 text-xs text-red-700">
              {pendingError}
              <button
                onClick={() => setPendingError(null)}
                className="ml-2 text-red-700 hover:underline"
              >
                dismiss
              </button>
            </div>
          )}

          {pending && (
            <div className="border-b border-neutral-200 bg-yellow-50 p-4">
              <p className="text-xs uppercase tracking-wider text-neutral-600">
                New annotation
              </p>
              <blockquote className="my-2 border-l-2 border-neutral-400 pl-3 text-xs text-neutral-700">
                {pending.quote.length > 200
                  ? pending.quote.slice(0, 200) + "…"
                  : pending.quote}
              </blockquote>
              <textarea
                autoFocus
                value={draftBody}
                onChange={(e) => setDraftBody(e.target.value)}
                onKeyDown={(e) => {
                  if (e.key === "Escape") {
                    setPending(null);
                    setDraftBody("");
                  }
                }}
                placeholder="Your comment…"
                rows={4}
                className="block w-full rounded border border-neutral-300 p-2 text-sm"
              />
              <div className="mt-2 flex gap-2">
                <button
                  onClick={saveAnnotation}
                  disabled={!draftBody.trim()}
                  className="rounded bg-neutral-900 px-3 py-1 text-xs font-medium text-white disabled:opacity-60"
                >
                  Save
                </button>
                <button
                  onClick={() => {
                    setPending(null);
                    setDraftBody("");
                  }}
                  className="rounded border border-neutral-300 px-3 py-1 text-xs"
                >
                  Cancel
                </button>
              </div>
            </div>
          )}

          <ol className="divide-y divide-neutral-200">
            {loading && <li className="p-4 text-xs text-neutral-500">Loading…</li>}
            {!loading && visible.length === 0 && (
              <li className="p-4 text-xs text-neutral-500">
                No annotations yet. Select text in the document to add one.
              </li>
            )}
            {visible.map((a) => (
              <li
                key={a.id}
                ref={(el) => {
                  if (el) cardRefs.current.set(a.id, el);
                  else cardRefs.current.delete(a.id);
                }}
                className={"p-4 " + (a.resolved ? "opacity-60" : "")}
              >
                <div className="flex items-baseline justify-between">
                  <p className="text-xs font-medium">{a.author_name}</p>
                  <p className="text-[10px] text-neutral-500">
                    {new Date(a.created_at).toLocaleString()}
                  </p>
                </div>
                <blockquote className="my-2 border-l-2 border-neutral-400 pl-3 text-xs text-neutral-600">
                  {a.quote.length > 160 ? a.quote.slice(0, 160) + "…" : a.quote}
                </blockquote>
                <p className="whitespace-pre-wrap text-sm">{a.body}</p>
                <div className="mt-2 flex gap-3 text-xs text-neutral-500">
                  <button onClick={() => toggleResolve(a)} className="hover:underline">
                    {a.resolved ? "Reopen" : "Resolve"}
                  </button>
                  <button
                    onClick={() =>
                      setReplyTo((cur) => (cur === a.id ? null : a.id))
                    }
                    className="hover:underline"
                  >
                    Reply
                  </button>
                  {a.author_name === author && (
                    <button onClick={() => remove(a)} className="hover:underline">
                      Delete
                    </button>
                  )}
                </div>

                {a.replies && a.replies.length > 0 && (
                  <ul className="mt-3 space-y-2 border-l-2 border-neutral-200 pl-3">
                    {a.replies.map((r) => (
                      <li key={r.id} className="text-xs">
                        <div className="flex items-baseline justify-between">
                          <span className="font-medium">{r.author_name}</span>
                          <span className="text-[10px] text-neutral-500">
                            {new Date(r.created_at).toLocaleString()}
                          </span>
                        </div>
                        <p className="whitespace-pre-wrap">{r.body}</p>
                        {r.author_name === author && (
                          <button
                            onClick={() => removeReply(r)}
                            className="mt-1 text-[10px] text-neutral-500 hover:underline"
                          >
                            Delete
                          </button>
                        )}
                      </li>
                    ))}
                  </ul>
                )}

                {replyTo === a.id && (
                  <div className="mt-3">
                    <textarea
                      autoFocus
                      value={replyDraft}
                      onChange={(e) => setReplyDraft(e.target.value)}
                      onKeyDown={(e) => {
                        if (e.key === "Escape") {
                          setReplyTo(null);
                          setReplyDraft("");
                        }
                      }}
                      placeholder="Your reply…"
                      rows={3}
                      className="block w-full rounded border border-neutral-300 p-2 text-xs"
                    />
                    <div className="mt-2 flex gap-2">
                      <button
                        onClick={() => submitReply(a)}
                        disabled={!replyDraft.trim()}
                        className="rounded bg-neutral-900 px-2 py-1 text-[10px] font-medium text-white disabled:opacity-60"
                      >
                        Post reply
                      </button>
                      <button
                        onClick={() => {
                          setReplyTo(null);
                          setReplyDraft("");
                        }}
                        className="rounded border border-neutral-300 px-2 py-1 text-[10px]"
                      >
                        Cancel
                      </button>
                    </div>
                  </div>
                )}
              </li>
            ))}
          </ol>
        </div>
      </aside>
    </div>
  );
}
```

## Verification template

After deployment, run these smoke checks (executed in workflow step 9). Replace `$URL`, `$PW`, `$DOC_ID`, `$SLUG`, `$TITLE` with the deployment values.

```bash
# 1. Login → 200 + cookies set
curl -sS -c /tmp/c.txt -X POST "$URL/api/login" \
  -H 'Content-Type: application/json' \
  -d "{\"password\":\"$PW\",\"name\":\"verify-script\"}" \
  -o /dev/null -w 'login=%{http_code}\n'

# 2. Auth gate (no cookies) → 307 to /login
curl -sS "$URL/" -o /dev/null -w 'gated=%{http_code}\n'

# 3. Authenticated root → 307 to first doc OR 200
curl -sS -b /tmp/c.txt "$URL/" -o /dev/null -w 'home=%{http_code}\n'

# 4. First doc page renders with title
curl -sS -b /tmp/c.txt "$URL/docs/$SLUG" | grep -q "$TITLE" && echo 'doc=OK' || echo 'doc=MISSING_TITLE'

# 5. Schema bootstraps on first POST (annotation against $DOC_ID)
curl -sS -b /tmp/c.txt -X POST "$URL/api/annotations" \
  -H 'Content-Type: application/json' \
  -d "{\"documentId\":\"$DOC_ID\",\"quote\":\"verification probe\",\"prefix\":\"\",\"suffix\":\"\",\"startOffset\":0,\"endOffset\":18,\"body\":\"verify\"}" \
  -w 'annotation=%{http_code}\n'

# 6. Short-quote rejection (length < 5 after normalize) → 400
curl -sS -b /tmp/c.txt -X POST "$URL/api/annotations" \
  -H 'Content-Type: application/json' \
  -d "{\"documentId\":\"$DOC_ID\",\"quote\":\"hi\",\"prefix\":\"\",\"suffix\":\"\",\"startOffset\":0,\"endOffset\":2,\"body\":\"x\"}" \
  -w 'short=%{http_code}\n'

# 7. Comments endpoint accepts an overall comment
curl -sS -b /tmp/c.txt -X POST "$URL/api/comments" \
  -H 'Content-Type: application/json' \
  -d "{\"documentId\":\"$DOC_ID\",\"body\":\"verification overall comment\"}" \
  -w 'comment=%{http_code}\n'

# 8. Reply endpoint (capture $AID first from annotation POST output if needed)
# AID=$(curl ... | jq -r .id)  ; then
# curl -sS -b /tmp/c.txt -X POST "$URL/api/replies" \
#   -H 'Content-Type: application/json' \
#   -d "{\"annotationId\":\"$AID\",\"body\":\"verification reply\"}" \
#   -w 'reply=%{http_code}\n'
```

Expected exit pattern: `login=200`, `gated=307`, `home=307` or `200`, `doc=OK`, `annotation=200`, `short=400`, `comment=200`, `reply=200`. Any deviation is a hard failure — investigate before reporting success.
