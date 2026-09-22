
# CI/CD Assignment 4 – Declarative Pipeline for Spring3Hibernate (Java)

**Submitted by jeetendra singh**

A declarative Jenkins pipeline for the Spring3Hibernate Java project that checks out code, runs stability/quality/coverage checks in parallel, feeds results into a SonarQube Quality Gate, generates a combined report, pauses for manual approval before publishing, and sends Slack + Email notifications at every outcome (failure, and the approval decision itself).

## Setup

### Installed the SonarQube Scanner plugin in Jenkins

Needed this so the pipeline can trigger a SonarQube analysis and wait on its Quality Gate result as part of the "code quality analysis" stage.

<img width="1440" height="900" alt="Screenshot 2026-09-22 at 4 36 37 PM" src="https://github.com/user-attachments/assets/b69e80b3-c7fe-4979-b300-a2c9d73309c7" />


### Installed SonarQube and logged in

Set up a local SonarQube server and logged in as Administrator, ready to create the project that this pipeline will analyze.


<img width="1440" height="900" alt="Screenshot 2026-09-22 at 7 29 53 PM" src="https://github.com/user-attachments/assets/f9f6d249-6e12-416f-b3e2-c9f13973cce0" />


### Generated a SonarQube token for Jenkins to authenticate with

Created a user token (jenkins-token) in SonarQube so Jenkins can push analysis results without using a personal login.

<img width="1440" height="900" alt="Screenshot 2026-09-22 at 7 52 50 PM" src="https://github.com/user-attachments/assets/b79930b0-b115-4cab-954d-1ff087172f12" />


### Added the SonarQube token as a Jenkins credential

Stored that token as sonar-token in Jenkins credentials, alongside the GitHub, Slack, and Gmail credentials already set up from earlier assignments, so this pipeline can reuse the same notification setup.

<img width="1440" height="900" alt="Screenshot 2026-09-22 at 7 59 36 PM" src="https://github.com/user-attachments/assets/74f819ad-d272-4711-ad2d-c2a3e8eafe93" />


### Configured the SonarQube server in Jenkins

**Manage Jenkins → System → SonarQube installations**

**Name:** `MySonarQube`

**Server URL:** `http://localhost:9000`

**Server authentication token:** `sonar-token`

This is what lets the pipeline's `withSonarQubeEnv` step know which server to talk to.

<img width="1440" height="900" alt="Screenshot 2026-09-22 at 8 10 59 PM" src="https://github.com/user-attachments/assets/a85bc716-17a3-4134-943b-c93d9e7360c0" />


### Created a webhook in SonarQube pointing back to Jenkins

Without this, SonarQube analysis would run but Jenkins would have no way of knowing whether the Quality Gate passed or failed - the pipeline's `waitForQualityGate` step depends entirely on this webhook firing back to `/sonarqube-webhook/`. Last delivery shows success.

![SonarQube Webhook](screenshots/sonarqube-webhook.png)

## Building the Pipeline Job

Created a new Pipeline item named `spring3hibernate-ci` with the declarative Jenkinsfile defining all the required stages: code checkout, a parallel block for stability/quality/coverage checks, Quality Gate wait, report generation, a manual input/approval step, and conditional artifact publishing - each notified via Slack and Email.

![Pipeline Job Configuration](screenshots/pipeline-job-configuration.png)

## Full stage view of the pipeline in action

This is the clearest picture of the whole pipeline working end to end. The stage table shows every stage exactly as required: Code Checkout → Build → Code Stability / Code Quality Analysis / Code Coverage Analysis (running in parallel) → Quality Gate → Generate Report → Approval for Publish → Publish Artifacts, with per-build timing for each stage.

Build history also shows the pipeline being run multiple times with intentional failures (builds #1-9 mostly red) before getting a clean pass (#10 green) - useful for proving the failure-notification path actually works, not just the happy path. The SonarQube Quality Gate for Spring3HibernateApp shows Passed.

![Full Pipeline Stage View](screenshots/pipeline-stage-view.png)

## SonarQube project dashboard after analysis

Confirms the analysis actually ran and pushed real results into SonarQube - bugs, vulnerabilities, code smells, coverage, and duplication numbers for Spring3HibernateApp, with an overall Passed Quality Gate.

![SonarQube Project Dashboard](screenshots/sonarqube-project-dashboard.png)

## Notifications

### Slack notifications across multiple runs

Each message includes not just the build status but also a Publish Decision field - N/A on the failed builds (since the pipeline never reached the approval stage) and Approve on the successful build #10, where a human actually approved the publish step. This is the part of the pipeline that satisfies the "notify the user post approval/denial" requirement.

![Slack Notifications](screenshots/slack-notifications.png)

### Matching email notifications

Same information reflected in email - Status and Publish Decision in the subject line, with the full build.log attached to each notification for debugging failed runs without needing to open Jenkins.

![Email Notifications](screenshots/email-notifications.png)

## How the Pipeline Satisfies Each Requirement

| Requirement                                | How it's implemented                                                                                                         |
| ------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------- |
| Code checkout                              | Code Checkout stage pulls from GitHub using `github-creds`                                                                   |
| Parallel stability/quality/coverage checks | Code Stability, Code Quality Analysis, Code Coverage Analysis run inside a `parallel {}` block                               |
| Skip scans optionally                      | Build parameters (booleans/choices) gate each parallel branch with a `when` condition, so any scan can be skipped per run    |
| Report generation                          | Generate Report stage combines the scan/coverage outputs into a single HTML report published via HTML Publisher              |
| Publish artifacts                          | Publish Artifacts stage archives the build output, but only runs after approval                                              |
| Approval gate before publish               | Approval for Publish stage uses a Jenkins `input` step, pausing the pipeline until a human approves or denies                |
| Notify on success/failure of publish       | Slack and Email post-build steps read the approval outcome and report it as the Publish Decision field in every notification |
