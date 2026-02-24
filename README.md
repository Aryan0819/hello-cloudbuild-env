# ☸️ GitOps-style Continuous Delivery for GKE
### Modern CI/CD with Cloud Build, Artifact Registry, and Kustomize

![GCP](https://img.shields.io/badge/Google_Cloud-4285F4?style=for-the-badge&logo=google-cloud&logoColor=white)
![Kubernetes](https://img.shields.io/badge/kubernetes-%23326ce5.svg?style=for-the-badge&logo=kubernetes&logoColor=white)
![GitOps](https://img.shields.io/badge/GitOps-Managed-orange?style=for-the-badge&logo=git)

This repository demonstrates a **production-grade GitOps workflow**. By decoupling application source code from deployment manifests, we achieve a highly auditable, secure, and automated path from commit to cluster.

---

## 🏗️ Architecture Overview

We utilize a **Two-Repository Strategy** to maintain a clean separation of concerns:

* **📦 App Repository:** Contains application source code, Dockerfile, and CI logic (`cloudbuild.yaml`).
* **🛠️ Env Repository:** Acts as the "Source of Truth" for infrastructure. It contains Kubernetes manifests managed via **Kustomize** overlays.

### The Modern Pipeline Flow
1.  **Commit:** A developer pushes code to the App Repo.
2.  **Build (CI):** Cloud Build triggers, builds a container image, and pushes it to **Artifact Registry**.
3.  **Hydrate:** Cloud Build clones the **Env Repo** and uses `kustomize edit set image` to update the deployment manifest with the new image tag.
4.  **Sync (CD):** A separate trigger on the Env Repo applies the updated manifests to **Google Kubernetes Engine (GKE)**.

---

## 🚀 Key Modern Features

* **Declarative State:** No manual `kubectl` commands. Git is the only way to change the cluster state.
* **Kustomize Integration:** Uses native Kubernetes configuration management instead of brittle text-replacement (sed) scripts.
* **Workload Identity:** Secure, keyless authentication between Cloud Build and GKE.
* **Environment Promotion:** * **Candidate Branch:** History of all deployment attempts.
    * **Production Branch:** History of verified, successful deployments.
* **Instant Rollbacks:** Simply re-run a previous Cloud Build job to revert the Git state and the cluster simultaneously.

---

## 🛠️ Implementation Snippet

A modern `cloudbuild.yaml` for the **App Repo** looks like this:

```yaml
steps:
  # 1. Build the container image
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', '$_REGION-docker.pkg.dev/$PROJECT_ID/$_REPO_NAME/app:$SHORT_SHA', '.']

  # 2. Push to Artifact Registry
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', '$_REGION-docker.pkg.dev/$PROJECT_ID/$_REPO_NAME/app:$SHORT_SHA']

  # 3. Update Env Repo using Kustomize
  - name: 'google/cloud-sdk'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        git clone [https://github.com/your-org/env-repo.git](https://github.com/your-org/env-repo.git)
        cd env-repo/overlays/production
        kustomize edit set image gke-app=$_REGION-docker.pkg.dev/$PROJECT_ID/$_REPO_NAME/app:$SHORT_SHA
        git add . && git commit -m "Update image to $SHORT_SHA"
        git push origin candidate
```
