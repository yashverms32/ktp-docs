---
sidebar_position: 1
---

# KTPGeorgia Deployment

## Overview

The **ktpgeorgia.com** website is deployed on an isolated LXC container on Kronos (ID: 116).

The deployment uses a self-hosted GitHub runner and Docker to automatically deploy on a push to main.

The workflow is located at `.github/workflows/deploy.yml`.

---

## Deployment Flow

1. Developer pushes changes to GitHub repository
2. GitHub triggers workflow
3. GitHub Runner rebuilds containers
4. Updated container is deployed automatically
5. Production site updates

No manual git pulling should be required on the server.

---

## Troubleshooting

### Changes pushed but not visible on production

Check:

- GitHub repo permissions for admin@ugaktp.com
- GitHub workflow logs
- Deployment status
- Container health

## Notes for Infrastructure Committee

- Do NOT revert to manual Git deployment

---
