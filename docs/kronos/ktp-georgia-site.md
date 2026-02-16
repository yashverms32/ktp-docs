---
sidebar_position: 1
---

# KTPGeorgia Deployment

## Overview

The **ktpgeorgia.com** website is deployed through **Dokploy** using GitHub-based CI/CD.

⚠️ The deployment previously used a direct Git connection which caused commits not to reliably reflect in production.  
This has now been fixed by switching to a GitHub account integration.

Production now pulls from GitHub automatically.

---

## Repository Source

Deployment uses the GitHub account tied to:
admin@ugaktp.com

This account is connected inside Dokploy and is the **authorized GitHub identity for CI/CD pulls**.

If commits are not appearing in production, verify:

- Repo is accessible to the admin GitHub account
- Dokploy GitHub integration is still active
- Webhook permissions remain enabled

---

## Deployment Flow

1. Developer pushes changes to GitHub repository
2. Dokploy detects new commit
3. Dokploy rebuilds container
4. Updated container is deployed automatically
5. Production site updates

No manual git pulling should be required on the server.

---

## Dokploy Location

The project can be found inside Dokploy under the KTPGeorgia application.

If access is needed:

- Login to Dokploy admin panel
- Navigate to applications
- Select **ktpgeorgia**

---

## Troubleshooting

### Changes pushed but not visible on production

Check:

- GitHub repo permissions for admin@ugaktp.com
- Dokploy build logs
- Deployment status
- Container health

### If deployment is stuck

Try:

- Trigger manual redeploy from Dokploy UI
- Confirm GitHub integration token has not expired

---

## Notes for Infrastructure Committee

- Do NOT revert to manual Git deployment
- Always ensure GitHub integration is used
- The admin@ugaktp.com GitHub account is part of the deployment chain

If credentials ever change, Dokploy GitHub integration must be re-authorized.

---
