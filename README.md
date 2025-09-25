# CSES Deployment Guide · Choose Your Own Adventure 

Read this like a flowchart. 

## 1. Prereqs for all

1. Monorepo layout is clear

   1. Each app folder contains frontend and backend when it is a web app
   2. Mobile apps may only have backend
   3. Example

```
/opportune/{frontend,backend}
/tritonscript/{frontend,backend}
/tritonspend/backend
/low-price-center/{frontend,backend}
```

2. Frontend build works locally

   1. CRA or Vite runs `npm --prefix frontend run build` and produces build or dist
   2. Next.js runs `npm --prefix frontend run build` and completes without blocking errors
3. Backend is serverless ready
   
   1. One file at `backend/api/index.ts` exports the handler that Vercel will invoke
5. Relative API calls only

   1. Frontends call `/api/...` not absolute URLs
6. One runtime and one lockfile per app

   1. Use either npm or pnpm, not both
7. Gmail account exists for project ops

   1. `csesos_[PROJECT_NAME]@gmail.com`
   2. Password is flexible but please SAVE it somewhere safe.
8. Components to expect

   1. Vercel Projects with Root Directory per app
   2. Databases: Neon Postgres or MongoDB Atlas depending on app
   3. Optional AWS S3 for specific projects

## 2. Specific Prereqs

1. Opportune

   1. Convert CRA to Vite before deploy
    https://adhithiravi.medium.com/migrating-from-create-react-app-to-vite-a-modern-approach-76148adb8983
2. TritonSpend

   1. Mobile client is Expo only
   2. Only the backend deploys on Vercel
   3. Expo reads `EXPO_PUBLIC_API_BASE_URL` to call the deployed backend

## 3. Steps to deploy on Vercel 

Use the sample JSON below as the baseline. Then choose the backend runtime line that matches your app. Screenshots to be added at placeholders.

1. Prepare `vercel.json` at the app root

   1. Sample JSON (Node + Next)
      
NPM:
````json
{
  "version": 2,
  "devCommand": "npm frontend run dev",
  "builds": [
    { "src": "frontend/package.json", "use": "@vercel/next" },
    { "src": "backend/api/index.ts", "use": "@vercel/node" }
  ],
  "routes": [
    { "src": "^/api/(.*)$", "dest": "backend/api/index.ts" },
    { "handle": "filesystem" },
    { "src": "/(.*)", "dest": "/frontend/$1" }
  ]
}
````
PNPM:
````json
{
  "version": 2,
  "devCommand": "pnpm --filter frontend run dev",
  "builds": [
    { "src": "frontend/package.json", "use": "@vercel/next" },
    { "src": "backend/api/index.ts", "use": "@vercel/node" }
  ],
  "routes": [
    { "src": "^/api/(.*)$", "dest": "backend/api/index.ts" },
    { "handle": "filesystem" },
    { "src": "/(.*)", "dest": "/frontend/$1" }
  ]
}
````

2. Create a [Vercel](http://vercel.com/) Project for each app

   1. Click Add New Project
   2. 
     <img width="785" height="514" alt="image" src="https://github.com/user-attachments/assets/0b4528c3-3aaf-411b-a273-b08916ad7aad" />

   3. Import from GitHub
   4. 
      <img width="422" height="484" alt="image" src="https://github.com/user-attachments/assets/bca91f14-2255-4783-810f-c91527bf9d31" />

   5. Set Root Directory to the app folder
   6. Copy both of your .env files into the project
      \[PLACEHOLDER IMAGE: Framework detection]
   7. Save

## 4. Steps to deploy on Neon and Atlas

Atlas is already used by most of our apps. Only confirm connection and network access. Neon steps are included with an optional local import path.

1. MongoDB Atlas

   1. Confirm `MONGODB_URI` works locally with a ping
   2. Confirm Network Access settings are correct for your Vercel deployment
   3. No extra deploy action is needed if the cluster already exists
2. [Neon](https://neon.com/) Postgres

   1. Create a Neon project and database if not present
   2. Option one: start empty and run your app migrations
   3. Option two: import from local using `pg_dump` and `pg_restore`

```bash
pg_dump --no-owner --format=custom --file=backup.dump "$LOCAL_DATABASE_URL"
pg_restore --no-owner --no-privileges --clean --if-exists --dbname="$DATABASE_URL" backup.dump
```

4. If you prefer plain SQL

```bash
pg_dump --no-owner --schema-only "$LOCAL_DATABASE_URL" | psql "$DATABASE_URL"
pg_dump --no-owner --data-only   "$LOCAL_DATABASE_URL" | psql "$DATABASE_URL"
```

5. Verify with a trivial query from your backend ping route

## 6. Backend entrypoint templates (backend/api/index.ts)

Pick one based on what this app uses. These templates assume your Express app is exported from `src/app.ts` and **does not** call `listen`. If you use path aliases like `src/*`, keep `import "module-alias/register"` in the entry; otherwise switch that import to a relative path like `import app from "../src/app"`.

### 6.1 Base or S3 only

S3 does not change the entrypoint. Keep it thin and pass through to your app. Routes that use S3 live in `src/app.ts`.

```ts
// backend/api/index.ts
import "module-alias/register";
import type { VercelRequest, VercelResponse } from "@vercel/node";
import app from "src/app";

export default async function handler(req: VercelRequest, res: VercelResponse) {
  return (app as unknown as (req: any, res: any) => void)(req, res);
}
```

### 6.2 MongoDB Atlas present

Connect Mongoose once per runtime, reuse on warm invocations.

```ts
// backend/api/index.ts
import "module-alias/register";
import type { VercelRequest, VercelResponse } from "@vercel/node";
import mongoose from "mongoose";
import app from "src/app";

const uri = process.env.MONGODB_URI!;

declare global { var __mongooseReady: Promise<typeof mongoose> | undefined }

function connectMongoose() {
  if (mongoose.connection.readyState === 1) return Promise.resolve(mongoose);
  if (mongoose.connection.readyState === 2) return mongoose.connection.asPromise();
  if (!global.__mongooseReady) global.__mongooseReady = mongoose.connect(uri);
  return global.__mongooseReady;
}

export default async function handler(req: VercelRequest, res: VercelResponse) {
  await connectMongoose();
  return (app as unknown as (req: any, res: any) => void)(req, res);
}
```

### 6.3 Neon Postgres present

Neon’s fetch client is serverless friendly. No port binding, optional warm-up.

```ts
// backend/api/index.ts
import "module-alias/register";
import type { VercelRequest, VercelResponse } from "@vercel/node";
import app from "src/app";

import { Pool, neonConfig } from "@neondatabase/serverless";
import ws from "ws";
neonConfig.webSocketConstructor = ws;

const url = process.env.DATABASE_URL;
if (!url) throw new Error("Missing DATABASE_URL");

declare global {
  // eslint-disable-next-line no-var
  var __pgPool: Pool | undefined;
}

export const client: Pool =
  global.__pgPool ?? (global.__pgPool = new Pool({ connectionString: url, max: 1 }));

client.on("error", (e) => console.error("pg pool error", e));

export default function handler(req: VercelRequest, res: VercelResponse) {
  return (app as unknown as (req: any, res: any) => void)(req, res);
}

```

Notes

1. Keep S3 logic in `src/app.ts` routes or small helper modules. The entrypoint does not need to initialize S3.
3. For local or non-Vercel hosting, keep a separate `server.ts` that calls `app.listen(PORT)` and connects to your DB before starting the server.
