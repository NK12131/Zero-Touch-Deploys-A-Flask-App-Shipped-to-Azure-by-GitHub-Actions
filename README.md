# ☁️ Flask on Azure: CI/CD with GitHub Actions and Azure App Service

> A minimal Flask web app that deploys itself to **Azure App Service (Linux)** through a **GitHub Actions** pipeline. Every push to `master` builds, packages and ships the app, signing in to Azure with OpenID Connect so no passwords or keys are stored in the repo.

![Python](https://img.shields.io/badge/Python-3.10-3776AB?logo=python&logoColor=white)
![Flask](https://img.shields.io/badge/Flask-latest-000000?logo=flask&logoColor=white)
![Azure App Service](https://img.shields.io/badge/Azure-App%20Service-0078D4?logo=microsoftazure&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/CI%2FCD-GitHub%20Actions-2088FF?logo=githubactions&logoColor=white)

---

## Table of Contents

- [Overview](#overview)
- [Pipeline](#pipeline)
- [Run Locally](#run-locally)
- [Deploy Your Own Copy](#deploy-your-own-copy)
- [Project Structure](#project-structure)
- [What I Learned](#what-i-learned)
- [Next Steps](#next-steps)
- [Credits](#credits)

---

## Overview

The app itself is deliberately tiny: one route that returns `Hello, World!`. The point of the project is the **deployment pipeline** around it:

- **Continuous deployment.** Every push to `master` triggers a build and a production deploy.
- **Manual runs.** The workflow can also be started from the Actions tab (`workflow_dispatch`).
- **Separate build and deploy jobs.** The build job produces a versioned artifact, and the deploy job ships exactly that artifact.
- **Keyless authentication.** The workflow signs in to Azure with **OpenID Connect (OIDC)** federated credentials. Only IDs (client, tenant and subscription) are stored as GitHub secrets, not a password or publish profile.
- **GitHub Environments.** Deploys go through the `Production` environment, which shows the live URL on each run and can require approval.

---

## Pipeline

```
 push to master / manual run
            │
            ▼
┌────────────────────────── build (ubuntu-latest) ─────────────────────────┐
│ checkout ─► set up Python 3.10 ─► create venv ─► pip install -r reqs    │
│        ─► zip the app into release.zip ─► upload artifact "python-app"  │
└──────────────────────────────────────────────────────────────────────────┘
            │  needs: build
            ▼
┌──────────────────────── deploy (Production environment) ─────────────────┐
│ download artifact ─► unzip ─► azure/login (OIDC)                         │
│        ─► azure/webapps-deploy ─► live on Azure App Service               │
└──────────────────────────────────────────────────────────────────────────┘
```

The workflow is in [`.github/workflows/master_voyage1500.yml`](.github/workflows/master_voyage1500.yml).

On the Azure side, App Service's Oryx build system finds `app.py` and serves the `app` object with Gunicorn automatically, so no startup command is needed.

---

## Run Locally

Requires Python 3.10 or later.

```bash
git clone <your-repo-url> flask-azure-cicd
cd flask-azure-cicd

python3 -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt

flask run
```

Open **http://127.0.0.1:5000**. You should see `Hello, World!`.

---

## Deploy Your Own Copy

1. **Create a Web App** in the Azure Portal: runtime stack **Python 3.10**, operating system **Linux**.
2. In the Web App, open **Deployment Center**, choose **GitHub** as the source, and pick your repo and branch. Azure then:
   - creates an app registration with a federated credential for your repo,
   - adds the `AZUREAPPSERVICE_CLIENTID_*`, `AZUREAPPSERVICE_TENANTID_*` and `AZUREAPPSERVICE_SUBSCRIPTIONID_*` secrets to the repo,
   - commits a workflow file like the one in this repo.
3. **Push to `master`** (or click **Run workflow** under **Actions**) and watch the build and deploy jobs run.
4. Open the URL shown on the `deploy` job, `https://<app-name>.azurewebsites.net`.

To deploy to a different app, change `app-name` in the workflow and the secret names to match your own.

---

## Project Structure

```
flask-azure-cicd/
├── app.py                                   # Flask app with a single route
├── requirements.txt                         # Flask (unpinned)
├── .github/
│   └── workflows/
│       └── master_voyage1500.yml            # Build and deploy pipeline
├── .gitignore
└── LICENSE                                  # MIT
```

---

## What I Learned

- How Azure App Service builds and runs Python apps (Oryx and Gunicorn)
- Setting up **OIDC federated identity** between GitHub and Azure instead of storing long-lived credentials
- Structuring a CI/CD workflow into **build and deploy jobs** linked by artifacts
- Using **GitHub Environments** for production deploys

---

## Next Steps

- [ ] Update the workflow actions: `actions/upload-artifact` and `actions/download-artifact` from v3 to v4, `setup-python` to v5, `azure/login` to v2 and `webapps-deploy` to v3 (v3 artifact actions no longer run on GitHub)
- [ ] Add a `pytest` step to the build job so a failing test blocks the deploy
- [ ] Add a staging deployment slot and swap it into production after checks pass
- [ ] Pin dependency versions for repeatable builds
- [ ] Add a health-check endpoint and monitoring with Application Insights

---

## Credits

Based on Microsoft's [Azure-Samples/python-docs-hello-world](https://github.com/Azure-Samples/python-docs-hello-world) sample, used with the Azure [Python quickstart](https://learn.microsoft.com/azure/app-service/quickstart-python). The GitHub Actions deployment pipeline was added on top. Licensed under the [MIT License](LICENSE).
