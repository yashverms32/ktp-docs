---
sidebar_position: 3
---

# Dokploy Implementation

Dokploy is our self-hosted Platform as a Service (PaaS). It runs in its own Proxmox VM and allows us to deploy websites and applications via Docker without manually writing Traefik labels or managing complex Compose files.

## Requirements

- **Dashboard Access:** You must be granted access to the Dokploy dashboard by the Infrastructure Chair.
- **GitHub Permissions:** Ensure you have "Read" access to the repository you intend to deploy within your organization.

## Steps

### 1. Access the Dashboard
Navigate to [https://dokploy.ugaktp.com](https://dokploy.ugaktp.com).
- **Login:** Use the credentials provided to you by the Infrastructure Chair.
- **Projects:** You will see the "Projects" overview. Projects group related services (e.g., "KTP Website," "Internal Tools") to keep the dashboard organized.

### 2. Linking a Repository (Optional)
Dokploy works best with Git automation:
- Navigate to **Settings -> Git** and link your GitHub account/provider if not already done.

### 3. Create or Select a Project
- Click **Create Project** and name it clearly (e.g., `ktp-docs`).
- Inside the project, click **Create Service**. Choose the service type that fits your needs (usually "Application").
- Select the repository and choose the branch (usually `main`).
- **Auto-Deploy:** Enable this to allow Dokploy to automatically redeploy the app every time you push code to GitHub.

### 4. Configure Build Settings
Choose how Dokploy should build your container:
- **Nixpacks (Recommended):** Best for standard apps (Node.js, Python, Go). It automatically detects your language and builds it—no Dockerfile required.
- **Dockerfile:** Use this if you have a custom `Dockerfile` in your repo.
- **Docker Compose:** Use this for complex stacks (e.g., an app + a database).

### 5. Environment Variables
- Paste your `.env` variables in the **Environment** section. 
- **Security:** Never hardcode secrets in the repository; Dokploy injects these securely at runtime.

### 6. Networking & Traefik
Dokploy handles Traefik routing automatically:
- Go to the **Domains** tab.
- **Host:** Enter the desired URL (e.g., `service.ugaktp.com`).
- **Container Port:** Set this to the port your app listens on (e.g., `3000` for Next.js, `8080` for Spring).
- Dokploy will automatically handle SSL certificates and external routing.

### 7. Persistent Storage (If needed)
If your app saves files (like images or a local SQLite DB), you **must** use the **Mounts** tab.
- Docker containers are ephemeral; any file saved inside the container will be deleted on the next deployment unless a persistent volume is mounted.

## Troubleshooting

- **Build Failure:** Check the **Logs** tab under **Deployments**. This is usually due to missing environment variables or a failing build command.
- **502 Bad Gateway:** Double-check the **Container Port** in the Domains tab. If your app runs on port `5000` but Dokploy is looking at `3000`, the connection will fail.
- **Health Check Failed:** If your app takes a long time to "boot up," increase the **Health Check** timeout in the service settings to give it more time before Dokploy marks it as "failed."