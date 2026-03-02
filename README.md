# 🚀 Workshop: Build a Multi-Tenant AI To-Do Agent in 60 Minutes

> **What you'll build:** A full-stack AI-powered to-do app with a calendar view, a chat sidebar that talks to a Vertex AI agent, and a PostgreSQL database — deployed live on Cloud Run — in under an hour.

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
| [Phase 3](#-phase-3--vertex-ai-agent-builder) | Vertex AI Agent Builder | ~10 min |
| [Phase 4](#-phase-4--antigravity-code-generation) | Code Generation with Antigravity | ~15 min |
| [Phase 5](#-phase-5--deploy-via-github) | Deploy via GitHub & Cloud Run | ~15 min |

---

## 🛠️ Phase 1 — Project & API Setup

**⏱️ Target time: 5 minutes**

### Step 1.1 — Create a Google Cloud Project

A *project* is Google Cloud's way of grouping all resources (databases, APIs, etc.) together.

**[GUI]**
1. Open [console.cloud.google.com](https://console.cloud.google.com).
2. Click the **project dropdown** at the top left (it may say "Select a project").
3. Click **New Project**.
4. Name it: **`agent-todo-workshop`**
5. Click **Create**.
6. Make sure the dropdown now shows **agent-todo-workshop** as active.

**[CLI]**
```bash
gcloud projects create agent-todo-workshop --set-as-default
```

> ✅ **Checkpoint:** The top-left dropdown in the Console shows **agent-todo-workshop**.

---

### Step 1.2 — Enable APIs

APIs are Google Cloud "services" your app will talk to. They're off by default — you need to turn them on.

| API | What it does |
|---|---|
| Cloud SQL Admin | Creates & manages your PostgreSQL database. |
| Vertex AI | Powers the AI agent that understands natural language. |
| Cloud Run | Hosts your app so anyone can visit it via a URL. |
| Secret Manager | Safely stores sensitive info like database passwords. |
| Cloud Build | Builds your app from source code automatically. |

**[GUI]**
1. In the Console search bar type **APIs & Services** and click the result.
2. Click **+ ENABLE APIS AND SERVICES**.
3. Search for and enable each of the five APIs listed above, one at a time.

**[CLI]**
```bash
gcloud services enable \
  sqladmin.googleapis.com \
  aiplatform.googleapis.com \
  run.googleapis.com \
  secretmanager.googleapis.com \
  cloudbuild.googleapis.com
```

> ✅ **Checkpoint:** Visiting **APIs & Services → Enabled APIs** shows all five APIs in the list.

---

### Step 1.3 — Grant Permissions (IAM)

Your app runs under a **Service Account** — think of it as a robot employee. You need to give that robot permission to access the database, the AI, and the secret store.

**[GUI]**
1. In the Console search bar type **IAM** and click **IAM & Admin → IAM**.
2. Find the row whose email ends in **`@developer.gserviceaccount.com`**.
3. Click the **pencil icon** (Edit) on that row.
4. Click **+ ADD ANOTHER ROLE** and add each of these roles (one at a time):
   - `Cloud SQL Client`
   - `Vertex AI User`
   - `Secret Manager Secret Accessor`
5. Click **Save**.

**[CLI]**
```bash
PROJECT_ID=$(gcloud config get-value project)
PROJECT_NUM=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')
SA="${PROJECT_NUM}-compute@developer.gserviceaccount.com"

for ROLE in cloudsql.client aiplatform.user secretmanager.secretAccessor; do
  gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:$SA" \
    --role="roles/$ROLE"
done
```

> ✅ **Checkpoint:** The service account row in IAM now shows three roles.

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

### Step 2.2 — Create the App User & Tables

We need a dedicated database user for the app and two tables: one for **users**, one for **tasks**.

**[GUI]**
1. Click on your **todo-db** instance.
2. In the left sidebar, click **Users** → **Add User Account**.
   - **Username:** `todo_app_user`
   - **Password:** `app-secret-123`
   - Click **Add**.
3. In the left sidebar, click **SQL Editor** (you may need to click **Open SQL Editor**).
4. **Copy and paste** the entire SQL block below into the editor.
5. Click **Run**.

**SQL Script:**

```sql
-- Create the users table
CREATE TABLE users (
  id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email TEXT UNIQUE NOT NULL
);

-- Create the tasks table
CREATE TABLE tasks (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     UUID REFERENCES users(id) ON DELETE CASCADE,
  title       TEXT NOT NULL,
  description TEXT,
  due_date    TIMESTAMP WITH TIME ZONE,
  status      TEXT DEFAULT 'pending',
  created_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Row-Level Security: each user can only see their own tasks
ALTER TABLE tasks ENABLE ROW LEVEL SECURITY;

CREATE POLICY task_isolation_policy ON tasks
  USING (user_id::text = current_setting('app.current_user_id'));

-- Grant permissions to our app user
GRANT ALL ON tasks, users TO todo_app_user;
```

> 💡 **What is Row-Level Security?** It's a PostgreSQL feature that automatically filters rows so User A can never see User B's tasks — even if a bug in your code forgets to add a `WHERE` clause. This is the "multi-tenant" part of the workshop.

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
postgresql://todo_app_user:app-secret-123@localhost/postgres?host=/cloudsql/[PROJECT_ID]:us-central1:todo-db
```

> 💡 **How to find your Project ID:** Look at the top-left of the Console right next to the project name, or run `gcloud config get-value project` in Cloud Shell.

4. Click **Create Secret**.

**[CLI]**
```bash
PROJECT_ID=$(gcloud config get-value project)

echo -n "postgresql://todo_app_user:app-secret-123@localhost/postgres?host=/cloudsql/$PROJECT_ID:us-central1:todo-db" | \
  gcloud secrets create DATABASE_URL --data-file=-
```

> ✅ **Checkpoint:** Secret Manager lists **DATABASE_URL** with 1 version.

---

## 🧠 Phase 3 — Vertex AI Agent Builder

**⏱️ Target time: 10 minutes**

### Step 3.1 — Create the Agent

An **Agent** is a natural-language AI that can be connected to *tools* (like your database) so it can take real actions on behalf of the user.

**[GUI]**
1. Search **Vertex AI** in the Console → click **Vertex AI**.
2. In the left sidebar find **Agent Builder** → click it.
3. Click **Create App** → select **Agent**.
4. Name it: **`TaskAgent`**
5. Click **Create**.

---

### Step 3.2 — Connect the Agent to Your Database

This gives the agent the ability to read and write tasks.

1. Inside your agent, click the **Tools** tab.
2. Click **Create** → choose **Cloud SQL**.
3. Select your **todo-db** instance.
4. In the **Instructions** field, paste **exactly** this:

```
Before every query, you MUST run:
SET LOCAL app.current_user_id = '{{user_uuid}}';
```

> 💡 **Why this instruction?** Remember Row-Level Security from Phase 2? This command tells PostgreSQL *which user is making the request* so it can filter tasks automatically. The `{{user_uuid}}` placeholder is replaced by your app at runtime.

> ✅ **Checkpoint:** The Tools tab shows a Cloud SQL tool connected to **todo-db**. Copy and save the **Agent ID** from the top of the page — you'll need it in Phase 4.

---

## 💻 Phase 4 — Antigravity Code Generation

**⏱️ Target time: 15 minutes**

### Step 4.1 — Open Antigravity IDE

Open the **Antigravity IDE** (the AI-powered editor). If you don't have it open already, your workshop facilitator will provide the link.

### Step 4.2 — Generate the App with the Master Prompt

**Copy the entire prompt below** and paste it into the Antigravity editor's chat panel. Replace `[YOUR_AGENT_ID]` with the Agent ID you saved from Phase 3.

```
Create a Next.js 15 application.

UI: Use 'react-big-calendar' for a full-page schedule on the left.
    AI Chat sidebar on the right.

Auth: Build a mock login page that saves a User UUID from the
      'users' table into a secure cookie.

Database & ORM: Use Drizzle ORM for simple and succinct database
    interactions. Use @google-cloud/secret-manager to fetch
    'DATABASE_URL' at runtime. Ensure the Drizzle schema includes
    description and created_at fields for tasks.

Agent: Connect to Vertex AI Agent ID [YOUR_AGENT_ID].
       Ensure every message sends the user_uuid.

Testing & Validation: Test your work at the end! Write a quick
    connection test script to verify the Drizzle DB connection works,
    and ensure the calendar renders without hydration errors.
```

> 💡 **What just happened?** Antigravity read your prompt, scaffolded an entire Next.js project, wired up the database layer, created the UI components, and wrote tests — all automatically.

### Step 4.3 — Review the Generated Code

Take a minute to look through the generated files. Key things to notice:

| File / Folder | Purpose |
|---|---|
| `app/page.tsx` | The main page with the calendar + chat layout. |
| `app/login/page.tsx` | The mock login page. |
| `db/schema.ts` | Drizzle ORM schema defining `users` and `tasks` tables. |
| `lib/secrets.ts` | Fetches `DATABASE_URL` from Secret Manager. |
| `scripts/test-connection.ts` | Tests that the DB connection works. |

### Step 4.4 — Push to GitHub

1. Click the **Source Control** icon in the left sidebar (it looks like a branch).
2. Click **Initialize Repository**.
3. Type a commit message (e.g., `initial workshop app`).
4. Click **Commit**.
5. Click **Publish Branch** and follow the prompts to push to GitHub.

> ✅ **Checkpoint:** Your GitHub repo shows the full project with all generated files.

---

## 🚢 Phase 5 — Deploy via GitHub

**⏱️ Target time: 15 minutes**

### Step 5.1 — Create a Cloud Run Service

Cloud Run will host your app and automatically re-deploy every time you push to GitHub.

**[GUI]**
1. Search **Cloud Run** in the Console → click it.
2. Click **Create Service**.
3. Check **✅ Continuously deploy from a repository**.
4. Click **Set up with Cloud Build**.
5. Select your **GitHub repo** from the list (you may need to authorize Cloud Build to access GitHub).
6. Set **Build Type** to: **Next.js**
7. Click **Save**.

---

### Step 5.2 — Configure Database & Secrets

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
| **3. Talk to the agent** | In the chat sidebar, type: *"I have a meeting tomorrow at 10 AM regarding the project launch. Please add that description."* |
| **4. Check the calendar** | Refresh the page (or wait for auto-update). Your new task should appear on the calendar with its title, time, and description! |

> 🎉 **Congratulations!** You just built and deployed a multi-tenant AI-powered to-do agent with a calendar UI, a chat interface, row-level security, and continuous deployment — all in under an hour.

---

## 🧹 Clean Up (Optional)

To avoid any charges after the workshop, delete the project:

```bash
gcloud projects delete agent-todo-workshop
```

Or via the Console: **IAM & Admin → Settings → Shut Down**.

---

## 🔧 Troubleshooting

| Problem | Fix |
|---|---|
| **Cloud SQL instance stuck creating** | Wait up to 10 minutes. If still stuck, delete and recreate. |
| **"Permission denied" errors** | Re-check Step 1.3 — make sure all three IAM roles are assigned. |
| **Secret Manager says "not found"** | Verify the secret name is exactly `DATABASE_URL` (case-sensitive). |
| **Cloud Run deploy fails** | Check **Cloud Build → History** for error logs. Common fix: ensure `package.json` has a `build` script. |
| **Calendar shows no tasks after chatting** | Refresh the page. Check that the Agent ID in your code matches the one in the Console. |
| **"Quota exceeded" error** | You may need to enable billing or request a quota increase for Vertex AI in your region. |

---

## 📚 Glossary for Beginners

| Term | Plain-English Definition |
|---|---|
| **API** | A way for two pieces of software to talk to each other. |
| **Cloud Run** | Google's service that runs your app in a container and gives it a public URL. |
| **Cloud SQL** | A managed database service — Google handles backups, updates, and security. |
| **Drizzle ORM** | A lightweight library that lets you talk to a database using TypeScript instead of raw SQL. |
| **IAM** | *Identity and Access Management* — Google Cloud's permission system. |
| **Next.js** | A popular React framework for building web applications. |
| **Row-Level Security (RLS)** | A database feature that restricts which rows a user can see. |
| **Secret Manager** | A vault for storing passwords and API keys securely. |
| **Service Account** | A special Google Cloud account used by apps (not humans). |
| **Vertex AI** | Google's AI/ML platform — it powers the chatbot agent in this workshop. |