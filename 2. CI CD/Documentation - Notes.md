
---

## SDLC - Software Development Life Cycle


SDLC is a structured methodology used by development teams to build, deliver and manage high-quality and cost-effective software systems. It's a roadmap designed to make the process of delivering modular and easier to digest. Each part has its own objectives and helps stay on track, guiding to the next step. 

There are many SDLC models, each has its own pros and cons.  
These are the 7 phases agreed by every model:

| **Phase**          | **Key activities**                             | **Deliverables**                         |
| ------------------ | ---------------------------------------------- | ---------------------------------------- |
| 1.     Planning    | Identify project scope, goals and requirements | Initial project plan                     |
| 2.     Analysis    | Gather and review data on project requirements | Fully detailed requirement documentation |
| 3.     Design      | Define project architecture                    | Software design document (SDD)           |
| 4.     Coding      | Write initial code                             | Functional software prototype            |
| 5.     Testing     | Review code and eliminate bugs                 | Refined, optimized software              |
| 6.     Deployment  | Deploy code to production environment          | Software available to end users          |
| 7.     Maintenance | Continual fixes and improvements               | Updated and optimized code               |

### Stage 1: Planning & Feasibility Analysis

This stage determines whether the project is technically, financially, and operationally feasible.

- ****Activities:**** Feasibility analysis, cost estimation, scheduling, resource planning
- ****Output:**** Project Plan, Feasibility Report
- ****Key Roles:**** Project Managers, Senior Engineers, Stakeholders

### Stage 2: Requirement Specification (SRS)

In this stage, detailed functional and non-functional requirements are documented clearly and approved by stakeholders.

- ****Activities:**** Requirement gathering, validation, documentation
- ****Output:**** Software Requirement Specification (SRS)
- ****Key Roles:*** Business Analysts, Product Owners


### Stage 3: System Design

In this stage, the approved requirements are transformed into a technical blueprint for implementation.

- ****High-Level Design**** (HLD): Defines system architecture, technology stack, database design, and major modules.
- ****Low-Level Design (LLD):**** Specifies component logic, APIs, data structures, and workflows.
- ****Output:**** Design Document Specification (DDS)


### Stage 4: Development (Coding)

Developers build the software based on the approved design.

- ****Activities:**** Coding, code reviews, unit testing, version control management
- ****Tools:**** IDEs, version control systems, debuggers
- ****Output:**** Source code, executable application
- ****Key Roles:**** Frontend, Backend, Full Stack Developers

### Stage 5: Testing

Testing ensures the software meets requirements and is free from defects before release.

****Types of Testing include:****

- ****Unit Testing****: Verifies individual components
- ****Integration Testing****: Ensures modules work together
- ****System Testing:**** Validates the complete system
- ****User Acceptance Testing (UAT)****: Confirms business requirements are met

****Output:**** Test cases, defect reports, and quality metrics.


### Stage 6: Deployment

The tested software is released to users.

- ****Activities:**** Production setup, deployment, smoke testing
- ****Modern Approach:**** Continuous Integration and Continuous Deployment (CI/CD) pipelines for faster and reliable releases
- ****Output:**** Live application
- ****Key Roles:**** DevOps Engineers, Release Managers


### Stage 7: Maintenance

Post-deployment support ensures long-term usability.

- ****Activities:**** Bug fixes, performance tuning, updates, feature enhancements
- ****Output:**** Patches, updates, new versions
- ****Key Roles:**** Support Engineers, Developers

### Common Models

- [Waterfall Model](https://www.geeksforgeeks.org/software-engineering/waterfall-model/)
- [Agile Model](https://www.geeksforgeeks.org/software-engineering/software-engineering-agile-development-models/)
- [V-Model](https://www.geeksforgeeks.org/software-engineering/software-engineering-sdlc-v-model/)
- [Spiral Model](https://www.geeksforgeeks.org/software-engineering/software-engineering-spiral-model/)
- [Incremental Model](https://www.geeksforgeeks.org/software-engineering/software-engineering-incremental-process-model/)
- [RAD Model](https://www.geeksforgeeks.org/software-engineering/software-engineering-rapid-application-development-model-rad/)

## Importance of SDLC

- Provides a clear and organized development framework.
- Improves planning, cost control, and project management.
- Ensures better quality through defined testing phases.
- Helps deliver software that meets user and business needs.


 ---
 
## CI/CD 
[Source:](https://www.geeksforgeeks.org/devops/what-is-ci-cd/) 

### Introduction

CI/CD (Continuous Integration and Continuous Delivery/Deployment) is a modern software development practice, which automates the building, testing and release of applications. It plays a key role in DevOps for streamlining collaboration between development and operations teams.
- Automates code integration, testing, and deployment workflows.
- Reduces manual effort while improving software quality.
- Smaller, frequent updates replace large risky releases.

![[before_ci_cd.webp]]![[after_ci_cd.webp]]

### 1. Continuous Integration (CI)

Continuous Integration focuses on integrating code changes frequently to avoid conflicts and ensure code stability.

- ****Goal:**** Prevent “integration hell” caused by late code merging.
- ****Process:**** Developers merge code changes into the main branch frequently (often daily).
- ****Automation:**** Each commit triggers an automated build and unit tests.
- ****Outcome:**** If tests fail, the build is rejected and developers are notified immediately.

### 2. Continuous Delivery (CD)

Continuous Delivery ensures that the application is always ready for release, with minimal manual effort.

- ****Goal:**** Keep the codebase in a release-ready state at all times.
- ****Process:**** After CI passes, code is deployed to a staging or testing environment.
- ****Automation:**** Integration, system, and performance tests are executed automatically.
- ****Release:**** Deployment to production is ****manual****, typically triggered when needed.

### 3. Continuous Deployment (CD)

Continuous Deployment takes automation a step further by removing manual intervention in releases.

- ****Goal:**** Enable fully automated and faster production releases.
- ****Process:**** After all tests pass, code is automatically deployed to production.
- ****Automation:**** End-to-end pipeline runs without human involvement.
- ****Requirement:**** Requires highly reliable and comprehensive automated testing.


### CI Workflow

It starts once a developer commits his code and ends with the status of the build. 
- Developer writes and commits code
- CI tool builds the application
- Automated tests are executed
- If issues occur -> “Problem detected” -> developer fixes code
- If successful -> “Everything OK” -> code is merged
- Application becomes ready for deployment

### CI/CD Workflow

![[tgb.webp]]


---

## Gitlab CI/CD

Gitlab aims to be the DevOps platform for CI/CD pipelines. Gitlab executes the new code a developer commits, based on how it was configured and then it releases the new release to the user. 

There are many tools for CI/CD.

> Biggest advantages of Gitlab:
> 	Almost all projects that use Gitlab CI/CD already exist as a repository on their servers, meaning that it acts as an extension of tools provided to the developers by Gitlab. It makes it seamless, starts without any setup effort, and the pipeline is part of the code, instead of configuring it separately.


### Comparisons: Gitlab CI/CD vs Jenkins vs GitHub Actions

| Choose This        | If You Want                    | But Be Aware                                 |
| ------------------ | ------------------------------ | -------------------------------------------- |
| **GitHub Actions** | Simplicity + GitHub ecosystem  | Mostly paid to use, free tier has limits     |
| **GitLab CI**      | All-in-one DevOps platform     | Steeper learning curve for complex workflows |
| **Jenkins**        | Ultimate flexibility + control | Requires significant DevOps expertise        |

**Things about each CI/CD Tool:**
Gitlab CI/CD:
- Shared, group or project-specific runners(Some are provided by Gitlab)
- Is easy to setup 
- Does not require self-host
- Has an automatic CI/CD pipeline generation
- Temporary environments for every merge request

Jenkins:
- 

GitHub Actions:
- Runners hosted by GitHub but can be self-hosted
- Has a marketplace of prebuilt actions
- Has a built in encrypted secrets


[Source for Comparisons](https://sanj.dev/post/github-actions-gitlab-ci-jenkins-comparison-2025/

| Feature              | GitHub Actions             | GitLab CI                     | Jenkins                   |
| -------------------- | -------------------------- | ----------------------------- | ------------------------- |
| **Parallel Jobs**    | 20 (Team), 40 (Enterprise) | Unlimited (self-hosted)       | Limited by infrastructure |
| **Build Time**       | 2-5 minutes typical        | 1-3 minutes typical           | Highly variable           |
| **Caching**          | Actions cache (10GB limit) | Built-in cache with no limits | Plugin-based, unlimited   |
| **Matrix Builds**    | Native support             | Native support                | Plugin required           |
| **Artifact Storage** | 500MB (free), 2GB+ (paid)  | Built-in with GitLab Pages    | Plugin-based, unlimited   |
| **Resource Classes** | Fixed GitHub instances     | Configurable runners          | Fully configurable        |

| Security Feature          | GitHub Actions             | GitLab CI                     | Jenkins                |
| ------------------------- | -------------------------- | ----------------------------- | ---------------------- |
| **Secret Management**     | Environment-scoped secrets | Project/group variables       | Credential plugins     |
| **RBAC**                  | Repository-based           | Project/group/instance levels | Plugin-based, granular |
| **Audit Logging**         | Enterprise only            | All tiers                     | Plugin-based           |
| **Compliance**            | SOC 2, GDPR ready          | SOC 2, ISO 27001              | Depends on deployment  |
| **Supply Chain Security** | Dependency scanning        | SAST/DAST integrated          | Plugin ecosystem       |
| **IP Restrictions**       | Enterprise only            | Available                     | Manual configuration   |
### Gitlab Architecture

#### Core CI/CD Components
[Source](https://dev.to/anusha_kuppili/gitlab-cicd-architecture-core-concepts-a-complete-hands-on-guide-with-examples-1ofn)

| Component    | Description                                                                |
| ------------ | -------------------------------------------------------------------------- |
| **Pipeline** | The full CI/CD process triggered by a push, MR, schedule, or manual action |
| **Stage**    | A logical grouping of jobs (build, test, deploy)                           |
| **Job**      | A single unit of work executed by a GitLab Runner                          |
| **Script**   | The shell commands that actually run inside a job                          |
##### Job -
Smallest unit in GitLab CI. It runs on a Runner and is part of a **Stage**. It executes scripts, sets artifacts, cache, rules and more. GitLab can run commands in order, so it can have the before script procedure and after.
Because Jobs don't share file systems, Artifacts are used to import an output of a job to another job. 

##### Stage - 
A group of jobs(build, test, deploy). Keywords can fine tune ordering.

##### Scripts - 
Executables that can be run by **Jobs**.

#### Architecture

**Gitlab Instance/Server** - hosts application code and pipelines, so it knows what needs to be done. Assigns pipeline jobs to available **Runners**

**Gitlab Runners** - separate machines that are connected to the **Gitlab** **Instance/Sever** that actually execute CI/CD jobs. Gitlab.com offers multiple **Runners** that are maintained directly by **Gitlab**

** You can create and configure your own Gitlab Instance, and create your own Runners to manage your pipeline.

	