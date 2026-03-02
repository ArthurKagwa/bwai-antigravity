# 🚀 Workshop: Build a To-Do Calendar App in 60 Minutes

> **What you'll build:** A full-stack to-do app with a calendar view, add/edit/delete tasks, user login, and a PostgreSQL database — deployed live on Cloud Run — in under an hour.

---

## 📋 Before You Start (Pre-requisites)

| Requirement | Why you need it |
|---|---|
| **Google Cloud account** with billing enabled | All services used have a free tier, but billing must be active. |
| **GitHub account** | We deploy straight from a GitHub repo. |
| **Chrome or Edge browser** | Cloud Console works best here. |
| **Antigravity IDE access** | The AI-powered code editor that generates the app for you. |

> 💡 **Beginner tip:** If you've never used Google Cloud, start at [console.cloud.google.com](https://console.cloud.google.com) and follow the new-account wizard first. You get $300 in free credits.

### How This Guide Works

Every step offers **two paths** — pick whichever feels comfortable:

| Label | Meaning |
|---|---|
| **[GUI]** | Point-and-click inside the Google Cloud Console (great for visual learners). |
| **[CLI]** | Copy-paste commands into **Cloud Shell** — the free terminal built into the Console (faster once you're comfortable). |

> 💡 **Beginner tip:** To open Cloud Shell, click the **`>_`** icon in the top-right corner of the Cloud Console. A terminal panel will slide up at the bottom of your screen.

---

## ⏱️ Sprint Overview

| Phase | What happens | Time |
|---|---|---|
| [Phase 1](#-phase-1--project--api-setup) | Project & API Setup | ~5 min |
| [Phase 2](#-phase-2--database--secrets) | Database & Secrets | ~15 min |
| [Phase 3](#-phase-3--antigravity-code-generation) | Code Generation with Antigravity | ~20 min |
| [Phase 4](#-phase-4--deploy-via-github) | Deploy via GitHub & Cloud Run | ~15 min |

---

## 🛠️ Phase 1 — Project & API Setup

**⏱️ Target time: 5 minutes**

### Step 1.1 — Create a Google Cloud Project

A *project* is Google Cloud's way of grouping all resources (databases, APIs, etc.) together.

**[GUI]**
1. Open [console.cloud.google.com](https://console.cloud.google.com).
2. Click the **project dropdown** at the top left (it may say "Select a project").
3. Click **New Project**.
4. Name it: **`todo-workshop`**
5. Click **Create**.
6. Make sure the dropdown now shows **todo-workshop** as active.

**[CLI]**
```bash
gcloud projects create todo-workshop --set-as-default
```

> ✅ **Checkpoint:** The top-left dropdown in the Console shows **todo-workshop**.

---

### Step 1.2 — Enable APIs

APIs are Google Cloud "services" your app will talk to. They're off by default — you need to turn them on.

| API | What it does |
|---|---|
| Cloud SQL Admin | Creates & manages your PostgreSQL database. |
| Cloud Run | Hosts your app so anyone can visit it via a URL. |
| Secret Manager | Safely stores sensitive info like database passwords. |
| Cloud Build | Builds your app from source code automatically. |

**[GUI]**
1. In the Console search bar type **APIs & Services** and click the result.
2. Click **+ ENABLE APIS AND SERVICES**.
3. Search for and enable each of the four APIs listed above, one at a time.

**[CLI]**
```bash
gcloud services enable \
  sqladmin.googleapis.com \
  run.googleapis.com \
  secretmanager.googleapis.com \
  cloudbuild.googleapis.com
```

> ✅ **Checkpoint:** Visiting **APIs & Services → Enabled APIs** shows all four APIs in the list.

---

## 🗄️ Phase 2 — Database & Secrets

**⏱️ Target time: 15 minutes**

### Step 2.1 — Create a PostgreSQL Instance

Cloud SQL gives you a fully-managed PostgreSQL database — no installation required.

**[GUI]**
1. Search **SQL** in the Console search bar → click **SQL**.
2. Click **Create Instance** → choose **PostgreSQL**.
3. Fill in:
   - **Instance ID:** `todo-db`
   - **Password:** `workshop-pass-2026`
4. Under *Choose a Cloud SQL edition*, select **Enterprise**.
5. Under *Choose availability*, select **Zonal** (lower cost — perfect for a workshop).
6. Click **Create Instance**.

> ⏳ This takes **3–5 minutes**. Don't wait — move on to reading Step 2.2 while it provisions.

**[CLI]**
```bash
gcloud sql instances create todo-db \
  --database-version=POSTGRES_15 \
  --tier=db-f1-micro \
  --region=us-central1

gcloud sql users set-password postgres \
  --instance=todo-db \
  --password=workshop-pass-2026
```

> ✅ **Checkpoint:** The SQL page shows **todo-db** with a green checkmark (status: *Runnable*).

---

### Step 2.2 — Create the Tables

We need two tables: one for **users**, one for **tasks**.

**[GUI]**
1. Click on your **todo-db** instance.
2. In the left sidebar, click **SQL Editor** (you may need to click **Open SQL Editor**).
3. **Copy and paste** the entire SQL block below into the editor.
4. Click **Run**.

**SQL Script:**

```sql
-- Create the users table
CREATE TABLE "users" (
  "id"    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  "email" TEXT UNIQUE NOT NULL
);

-- Create the tasks table
CREATE TABLE "tasks" (
  "id"          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  "user_id"     UUID REFERENCES "users"("id") ON DELETE CASCADE,
  "title"       TEXT NOT NULL,
  "description" TEXT,
  "due_date"    TIMESTAMP WITH TIME ZONE,
  "status"      TEXT DEFAULT 'pending',
  "created_at"  TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- (No extra grants needed — we connect as the root postgres user)
```

> ✅ **Checkpoint:** The SQL Editor shows **"Statement executed successfully"** (or similar).

---

### Step 2.3 — Store the Database Connection String as a Secret

Instead of pasting a password into your code (never do that!), we store it in **Secret Manager** and fetch it at runtime.

**[GUI]**
1. Search **Secret Manager** in the Console → click it.
2. Click **Create Secret**.
3. Fill in:
   - **Name:** `DATABASE_URL`
   - **Secret value:** (paste the string below, replacing `[PROJECT_ID]` with your actual project ID)

```
postgresql://postgres:workshop-pass-2026@localhost/postgres?host=/cloudsql/[PROJECT_ID]:us-central1:todo-db
```

> 💡 **How to find your Project ID:** Look at the top-left of the Console right next to the project name, or run `gcloud config get-value project` in Cloud Shell.

4. Click **Create Secret**.

**[CLI]**
```bash
PROJECT_ID=$(gcloud config get-value project)

echo -n "postgresql://postgres:workshop-pass-2026@localhost/postgres?host=/cloudsql/$PROJECT_ID:us-central1:todo-db" | \
  gcloud secrets create DATABASE_URL --data-file=-
```

> ✅ **Checkpoint:** Secret Manager lists **DATABASE_URL** with 1 version.

---

## 💻 Phase 3 — Antigravity Code Generation

**⏱️ Target time: 20 minutes**

### Step 3.1 — Open Antigravity IDE

Open the **Antigravity IDE** (the AI-powered editor). If you don't have it open already, your workshop facilitator will provide the link.

### Step 3.2 — Generate the App with the Master Prompt

**Copy the entire prompt below** and paste it into the Antigravity editor's chat panel.

```
Create a Next.js 15 application for a personal to-do list with a calendar view.

=== UI LAYOUT ===
- Full-page calendar using 'react-big-calendar' showing the user's tasks
- A slide-in or modal form for adding and editing tasks

=== AUTHENTICATION ===
Build a simple mock login page:
- User enters their email
- If email exists in the 'users' table, log them in
- If not, create a new user and log them in
- Store the user's UUID in a secure HTTP-only cookie
- Redirect to the main calendar page after login

=== DATABASE & ORM ===
Use Drizzle ORM with PostgreSQL:
- Use @google-cloud/secret-manager to fetch 'DATABASE_URL' at runtime
- Schema for 'users' table: id (UUID), email (TEXT)
- Schema for 'tasks' table: id (UUID), user_id (UUID FK), title (TEXT),
  description (TEXT), due_date (TIMESTAMP), status (TEXT), created_at (TIMESTAMP)

=== TASK MANAGEMENT ===
Users manage tasks through the UI:
- Add Task: Button opens a form with title, description, due date, status fields
- Edit Task: Click a task on the calendar to open the edit form
- Delete Task: Delete button on the edit form
- Tasks display on the calendar based on due_date
- Filter bar to show tasks by status: all / pending / in-progress / completed

=== CRITICAL - BUILD-TIME CONFIGURATION ===
- Add 'export const dynamic = "force-dynamic"' to all pages that
  query the database to prevent static generation.
- Database connection ONLY initialized at REQUEST time, never at build time.
- In next.config.js, set output: 'standalone' for Cloud Run.
- Add a build-time check: if DATABASE_URL is missing, skip database
  initialization (Cloud Build won't have it, but Cloud Run will).

=== TESTING ===
- Write a quick connection test script to verify the Drizzle DB connection works
- Ensure the calendar renders without hydration errors
```

> 💡 **What just happened?** Antigravity read your prompt and scaffolded an entire Next.js project with:
> - A calendar for viewing tasks
> - Forms for adding, editing, and deleting tasks
> - User login backed by the database
> - Drizzle ORM wired up to your Cloud SQL instance

### Step 3.3 — Review the Generated Code

Take a minute to look through the generated files. Key things to notice:

| File / Folder | Purpose |
|---|---|
| `app/page.tsx` | The main calendar page. |
| `app/login/page.tsx` | The mock login page. |
| `app/api/tasks/route.ts` | API routes for creating, reading, updating, and deleting tasks. |
| `db/schema.ts` | Drizzle ORM schema defining `users` and `tasks` tables. |
| `lib/secrets.ts` | Fetches `DATABASE_URL` from Secret Manager. |
| `components/TaskForm.tsx` | Form for adding/editing tasks. |
| `next.config.js` | Must include `output: 'standalone'` for Cloud Run. |
| `db/client.ts` | Should check if `DATABASE_URL` exists before connecting. |

> 💡 **Why the build-time check?** During Cloud Build, the database isn't accessible — the `DATABASE_URL` secret only becomes available when the container runs on Cloud Run. Pages that query the database must use `export const dynamic = 'force-dynamic'` to skip static generation, and the DB client should gracefully handle missing credentials at build time.

### Step 3.4 — Push to GitHub

1. Click the **Source Control** icon in the left sidebar (it looks like a branch).
2. Click **Initialize Repository**.
3. Type a commit message (e.g., `initial workshop app`).
4. Click **Commit**.
5. Click **Publish Branch** and follow the prompts to push to GitHub.

> ✅ **Checkpoint:** Your GitHub repo shows the full project with all generated files.

---

## 🚢 Phase 4 — Deploy via GitHub

**⏱️ Target time: 15 minutes**

### Step 4.1 — Create a Cloud Run Service

Cloud Run will host your app and automatically re-deploy every time you push to GitHub.

**[GUI]**
1. Search **Cloud Run** in the Console → click it.
2. Click **Create Service**.
3. Check **✅ Continuously deploy from a repository**.
4. Click **Set up with Cloud Build**.
5. Select your **GitHub repo** from the list (you may need to authorize Cloud Build to access GitHub).
6. Set **Build Type** to: **Next.js**
7. Click **Save**.

> 💡 **Important:** During the build process, Cloud Build does NOT have access to your database or secrets. This is why the Antigravity prompt included instructions to:
> - Use `export const dynamic = 'force-dynamic'` on database-querying pages
> - Check for the existence of `DATABASE_URL` before initializing connections
> 
> These patterns ensure Next.js skips static generation for those pages and only connects to the database at request time when the app is running on Cloud Run.

---

### Step 4.2 — Configure Database & Secrets

Still on the Create Service page, scroll down:

**Database Connection:**
1. Find the **Connections** section (under *Container, Networking, Security*).
2. Click **Add Connection**.
3. Select your **todo-db** Cloud SQL instance.

**Environment Variable from Secret:**
1. Find the **Variables & Secrets** section.
2. Click **Reference a Secret**.
3. Fill in:
   - **Environment variable name:** `DATABASE_URL`
   - **Secret:** `DATABASE_URL`
   - **Version:** `latest`

4. Click **Create** to deploy the service.

> ⏳ The first deploy takes **3–5 minutes**. Cloud Build pulls your code from GitHub, builds it, and launches it on Cloud Run.

> ✅ **Checkpoint:** Cloud Run shows your service with a green checkmark and a URL at the top.

---

## ✅ Final Verification

You did it! Let's confirm everything works end to end.

| Step | Action |
|---|---|
| **1. Open your app** | Click the **URL** shown at the top of the Cloud Run service page. |
| **2. Log in** | On the login page, enter: `workshop@test.com` |
| **3. Add a task** | Click the add button and fill in: *"Project meeting"*, tomorrow's date at 10 AM, status *pending*. |
| **4. Check the calendar** | Your task should appear on the calendar on tomorrow's date. |
| **5. Edit the task** | Click the task on the calendar and update its description. |
| **6. Mark it done** | Change the status to *completed* and save. |

> 🎉 **Congratulations!** You just built and deployed a full-stack to-do calendar app with user login, task management, and continuous deployment — all in under an hour.

---

## 🧹 Clean Up (Optional)

To avoid any charges after the workshop, delete the project:

```bash
gcloud projects delete todo-workshop
```

Or via the Console: **IAM & Admin → Settings → Shut Down**.

---

## 🔧 Troubleshooting

| Problem | Fix |
|---|---|
| **Cloud SQL instance stuck creating** | Wait up to 10 minutes. If still stuck, delete and recreate. |
| **Secret Manager says "not found"** | Verify the secret name is exactly `DATABASE_URL` (case-sensitive). |
| **Cloud Run deploy fails** | Check **Cloud Build → History** for error logs. Common fix: ensure `package.json` has a `build` script. |
| **"Can't connect to database" during build** | Make sure pages use `export const dynamic = 'force-dynamic'` and the DB client checks if `DATABASE_URL` exists before connecting. Database is only available at runtime, not build time. |
| **Calendar shows no tasks after adding** | Refresh the page. Check browser console for errors and that the `app/api/tasks` routes are returning 200. |
| **Login doesn't redirect** | Check that the cookie is being set correctly and the `users` table was created successfully in Step 2.2. |

---

## 📚 Glossary for Beginners

| Term | Plain-English Definition |
|---|---|
| **API** | A way for two pieces of software to talk to each other. |
| **Cloud Build** | Google's service that compiles and packages your code automatically when you push to GitHub. |
| **Cloud Run** | Google's service that runs your app in a container and gives it a public URL. |
| **Cloud SQL** | A managed database service — Google handles backups, updates, and security. |
| **Drizzle ORM** | A lightweight library that lets you talk to a database using TypeScript instead of raw SQL. |
| **Next.js** | A popular React framework for building web applications. |
| **Secret Manager** | A vault for storing passwords and API keys securely. |