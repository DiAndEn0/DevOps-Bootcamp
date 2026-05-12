# CI/CD & Software Development Life Cycle Bootcamp

## 📖 Phase 1: Theoretical Foundation
* **SDLC (Software Development Life Cycle):** Understand the phases of software creation, from planning and coding to testing, deployment, and maintenance.
* **What is CI/CD?** Grasp the concepts of Continuous Integration (automating builds and tests) and Continuous Deployment/Delivery (automating release to production).
* **CI/CD Tools Comparison:** * Research **GitLab CI** (Primary focus).
    * Understand the landscape by comparing it with Jenkins, GitHub Actions, and Travis CI.
* **Continuous Deployment (CD) Models:** Read about GitOps and modern CD tools like ArgoCD, as well as deployment strategies using GitHub Actions or Jenkins.
* **Semantic Versioning (SemVer):** Learn the MAJOR.MINOR.PATCH versioning standard and how it applies to software releases and Docker image tags.

---

## 💻 Phase 2: Practical Lab - App Development & CI
**Objective:** Create an application and automate its build process.

1.  **Application Development:**
    * Develop a simple web application (using Node.js, Python, Go, or your preferred language).
    * Ensure the app has basic tests.
2.  **Continuous Integration (CI) Pipeline:**
    * Set up a CI pipeline (using GitHub Actions, GitLab CI, or Jenkins).
    * **Linting:** Add a step to run a linter against your codebase to enforce code quality.
    * **Testing:** Add a step to execute your unit tests automatically. If tests fail, the pipeline should block.
    * **Build & Push:** Containerize your app with a `Dockerfile`. Add a pipeline step to build the Docker image and push it to a container registry (e.g., Docker Hub, GitHub/GitLab Container Registry).
3.  **Implement SemVer:**
    * Automate semantic versioning in your pipeline. Ensure that when you release a new version, the Docker image is tagged accordingly (e.g., `v1.0.0`, `v1.0.1`) rather than just `latest`.

---

## 🚀 Phase 3: Practical Lab - Continuous Deployment
**Objective:** Automate the delivery of your application to a Kubernetes cluster.

1.  **Target Environment:** Use the K3s VM cluster you built in the Virtualization Bootcamp.
2.  **Continuous Deployment (CD) Pipeline:**
    * Extend your pipeline (or create a new CD process) to deploy the newly pushed Docker image into your K3s cluster.
    * Update your Kubernetes deployments automatically when a new SemVer tag is pushed.
    * Try setting up ArgoCD to pull the changes directly from your repository (GitOps style) instead of pushing from the CI server.
