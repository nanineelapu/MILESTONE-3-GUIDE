# THEORY & DELIVERABLES — Ecommerce CI/CD Pipeline

> The case study asks for **7 written deliverables**. Each is answered below.
> Copy these into your final submission document (add your screenshots).

---

## Deliverable 1: CI/CD Architecture Diagram

```
   DEVELOPER
      | (git push)
      v
   GITHUB  (https://github.com/Msocial123/EcommerceApp.git)
      | (clone)
      v
  +-----------------------------------------------------+
  |  VM1 - JENKINS  (CI/CD + Build + Config Management)  |
  |    Git  ->  Maven (build)  ->  Ansible (deploy)      |
  +-----------------------------------------------------+
      |  analyse                         |  ssh deploy
      v                                   v
  VM2 - SONARQUBE                     VM3 - TOMCAT
  (Code Quality + Gate)               (Application Server)
  http://VM2:9000                     http://VM3:8080
```

**Components**
- **Developer** – writes code, pushes to GitHub.
- **GitHub** – central source code repository (SCM).
- **Jenkins (VM1)** – CI/CD orchestrator; also hosts Maven and Ansible.
- **SonarQube (VM2)** – static code analysis & Quality Gate.
- **Maven** – build tool that compiles code and produces the `.war`.
- **Ansible** – configuration management / deployment automation (runs from VM1).
- **Tomcat (VM3)** – application server that hosts the live Ecommerce app.

---

## Deliverable 2: Complete Pipeline Workflow

| Stage | Action | Tool | Pass condition |
|-------|--------|------|----------------|
| 1. Source Code Management | Connect to GitHub, clone repo, checkout branch | Git | Code pulled |
| 2. Code Quality Validation | Static analysis: bugs, vulnerabilities, security, code smells | SonarQube | Analysis completes |
| 3. Quality Gate | Evaluate results against quality rules | SonarQube | **PASS** → continue; **FAIL** → abort |
| 4. Application Build | Resolve dependencies, compile, package `.war` | Maven | Build success |
| 5. Deployment | Stop Tomcat → copy `.war` → start → verify | Ansible | App live on :8080 |

**Trigger:** Manual *Build Now* (can be extended to a GitHub webhook for auto-trigger on push).
**Core rule:** deployment happens **only if all earlier stages pass** — enforced by `abortPipeline: true` on the Quality Gate and Jenkins' fail-fast stage execution.

---

## Deliverable 3: Tool Integration Design

| Integration | How it is wired |
|-------------|-----------------|
| **Jenkins ↔ GitHub** | `git` step in the pipeline clones the repo over HTTPS. |
| **Jenkins ↔ SonarQube** | SonarQube Scanner plugin + `withSonarQubeEnv('SonarQube')`; auth via a **Secret text** token credential (`sonar-token`). |
| **SonarQube ↔ Jenkins (Quality Gate)** | A **webhook** in SonarQube posts the gate result back to `…/sonarqube-webhook/`; `waitForQualityGate` reads it. |
| **Jenkins ↔ Maven** | Maven configured as a Jenkins global tool (`Maven`); invoked via `tools { maven 'Maven' }`. |
| **Jenkins ↔ Ansible** | Ansible installed on VM1; pipeline calls `ansible-playbook`; uses inventory `/etc/ansible/hosts`. |
| **Ansible ↔ App Server** | Password-less SSH using the `jenkins` user's key copied into the App-Server `authorized_keys`. |

---

## Deliverable 4: Deployment Strategy

- **Type:** Automated **recreate** deployment (stop → replace → start).
- **Artifact:** A single `.war`, renamed to `ROOT.war` so the app serves at the root context (`/`).
- **Steps (Ansible):**
  1. Stop the Tomcat service (clean state).
  2. Remove the previous deployment (`ROOT` dir + `ROOT.war`).
  3. Copy the freshly built `.war` from the Jenkins workspace to `/opt/tomcat/webapps/`.
  4. Start Tomcat.
  5. **Verify** the app is reachable (`wait_for` port 8080).
- **Idempotent & repeatable:** every run produces the same clean end-state.
- **Future improvement:** blue-green or rolling deployment across two Tomcat nodes behind a load balancer for zero downtime.

---

## Deliverable 5: Infrastructure Automation Approach

- **Configuration Management:** Ansible (agentless, over SSH) handles all server operations — no manual login.
- **What is automated:** service stop/start, artifact transfer, deployment, and health verification.
- **Inventory-driven:** target servers defined in `/etc/ansible/hosts`; adding a server = adding a line.
- **Eliminates manual actions** the case study prohibits:
  - ❌ Manual server login → ✅ Ansible SSH
  - ❌ Manual service restart → ✅ `systemd` module
  - ❌ Manual file copy → ✅ `copy` module
  - ❌ Manual deployment → ✅ full playbook in the pipeline
- **Repeatable & version-controllable:** the playbook is plain YAML that can live in Git.

---

## Deliverable 6: Security & Credential Management Approach

- **No hard-coded secrets** in the pipeline script.
- **SonarQube token** stored in **Jenkins Credentials** (Secret text, ID `sonar-token`) and injected only at runtime via `withSonarQubeEnv`.
- **SSH access** uses **key-based authentication** (no passwords); the private key lives only on VM1 under the `jenkins` user with `700`/`600` permissions.
- **Least privilege:** SonarQube runs as a dedicated non-root `sonar` user; Tomcat runs as a non-root `tomcat` user.
- **Network security:** Security Groups expose only required ports (22, 8080, 9000); everything else is closed.
- **Best-practice extensions:** restrict SSH source to the Jenkins IP, use HTTPS, rotate tokens, and store secrets in a vault (e.g., Ansible Vault / AWS Secrets Manager).

---

## Deliverable 7: Failure Handling & Rollback Strategy

**Failure handling**
- **Fail-fast:** any stage that errors stops the pipeline; later stages never run.
- **Quality Gate enforcement:** `waitForQualityGate abortPipeline: true` stops a bad build before it is ever deployed.
- **Timeouts:** the Quality Gate stage has a 3-minute timeout to avoid hanging.
- **Verification:** the deploy playbook's `wait_for` confirms the app is actually up; if not, the run fails.
- **Notifications:** the `post { success / failure }` block reports the outcome (extendable to email/Slack).

**Rollback strategy**
- Because the last known-good `.war` is the previous successful build artifact, rollback = **re-run the pipeline pinned to the previous Git commit/tag**, or redeploy the archived artifact.
- The recreate flow (stop → remove → copy → start) means a rollback is just deploying the older `.war` the same way.
- **Improvement:** keep the previous `ROOT.war` as `ROOT.war.bak` so Ansible can instantly restore it on a failed health check.

---

## Summary Table (tools → role)

| Tool | Role in pipeline |
|------|------------------|
| GitHub | Source code repository |
| Jenkins | CI/CD orchestration server |
| SonarQube | Code quality + Quality Gate |
| Maven | Build & artifact (`.war`) creation |
| Ansible | Deployment / configuration automation |
| Tomcat | Application hosting server |
