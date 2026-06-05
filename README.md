# Milestone 3 — Enterprise CI/CD Pipeline for Ecommerce Application

> **DevOps Assessment | Tools: GitHub · Jenkins · SonarQube · Maven · Ansible · Tomcat (AWS EC2)**

This is the master guide. Read this file first, then follow the other files in order.

---

## 1. What You Are Building

An **automated CI/CD pipeline** that takes code from GitHub and deploys it to a live server — with zero manual steps.

**Application repo:** https://github.com/Msocial123/EcommerceApp.git

When a build runs, the pipeline automatically:
1. Pulls the latest code from GitHub
2. Scans it for bugs/vulnerabilities with SonarQube (and stops if quality fails)
3. Builds the application into a deployable `.war` file with Maven
4. Deploys it to a Tomcat server using Ansible — no manual login, copy, or restart

---

## 2. Architecture Diagram

```
                          ┌──────────────────┐
                          │    DEVELOPER      │
                          │  (pushes code)    │
                          └─────────┬─────────┘
                                    │ git push
                                    ▼
                          ┌──────────────────┐
                          │     GITHUB        │
                          │  EcommerceApp.git │
                          └─────────┬─────────┘
                                    │ clone
                                    ▼
        ┌───────────────────────────────────────────────────┐
        │            VM1 : JENKINS (CI/CD Server)            │
        │   ┌─────────┐   ┌────────┐   ┌─────────────────┐  │
        │   │  Git    │ → │ Maven  │ → │ Ansible (deploy)│  │
        │   └─────────┘   └────────┘   └────────┬────────┘  │
        └────────┬───────────────────────────────┼──────────┘
                 │ analyse code                   │ ssh deploy
                 ▼                                 ▼
   ┌──────────────────────────┐      ┌──────────────────────────┐
   │  VM2 : SONARQUBE          │      │  VM3 : TOMCAT (App Server)│
   │  Code Quality + Gate      │      │  Hosts the live app       │
   │  http://VM2-IP:9000       │      │  http://VM3-IP:8080       │
   └──────────────────────────┘      └──────────────────────────┘
```

---

## 3. Infrastructure (3 AWS EC2 Linux VMs)

| VM | Name | Role | Software | Open Ports |
|----|------|------|----------|------------|
| VM1 | `Jenkins-Server` | CI/CD Server + Build + Config Mgmt | Java 17, Jenkins, Git, Maven, Ansible | 22, 8080 |
| VM2 | `SonarQube-Server` | Code Quality Server | Java 17, SonarQube | 22, 9000 |
| VM3 | `App-Server` | Application Server | Java 17, Tomcat 9 | 22, 8080 |

- **OS:** Amazon Linux 2023 (use `dnf`/`yum`, NOT `apt`)
- **Access:** PuTTY from your laptop (user = `ec2-user`, key = `devops-key.ppk`)

---

## 4. Pipeline Flow (5 Stages)

| Stage | Name | Tool | What happens | If it fails |
|-------|------|------|--------------|-------------|
| 1 | Clone Source Code | Git | Pull latest code from GitHub | Stop |
| 2 | Code Quality | SonarQube | Static analysis: bugs, vulnerabilities, security | Stop |
| 3 | Quality Gate | SonarQube | Pass/Fail decision | Stop if FAIL |
| 4 | Build | Maven | Compile → package `.war` | Stop |
| 5 | Deploy | Ansible | Stop Tomcat → copy war → start → verify | Stop |

**Rule:** A version is deployed **only if every stage passes.**

> ℹ️ This is the **classic scripted-pipeline (Jenkinsfile)** version. Prefer a simpler point-and-click setup? See the **`ALTERNATIVE-FREESTYLE-METHOD.md`** file — same use case, no Groovy, no webhook.

---

## 5. How to Use These Files (Order)

1. **`STEP-BY-STEP-COMMANDS.md`** — Do this first. Every command for all 3 VMs in order. Follow top to bottom.
2. **`Jenkinsfile`** — Paste into the Jenkins pipeline job (Step 10).
3. **`deploy.yml`** — The Ansible playbook (created on VM1 in Step 8).
4. **`inventory-hosts.txt`** — The Ansible inventory (created on VM1 in Step 7).
5. **`SCREENSHOTS-GUIDE.md`** — Take every screenshot listed here for submission.
6. **`THEORY-DELIVERABLES.md`** — Copy the 7 written deliverables into your submission document.

> 🔀 **Two ways to build the Jenkins job — pick ONE:**
> - **Classic / Pipeline (default):** Steps 9 & 10 in `STEP-BY-STEP-COMMANDS.md` + the `Jenkinsfile`. Uses token credential, SonarQube server config, webhook, and an automatic Quality-Gate stop.
> - **Alternative / Freestyle (simpler):** `ALTERNATIVE-FREESTYLE-METHOD.md`. Point-and-click, no Groovy, no webhook, token pasted inline. Steps 1–8 are identical for both.

---

## 6. Quick Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Browser page won't load (8080/9000) | Port closed | Open the port in the VM's Security Group |
| Stage 1 fails | Wrong branch name | Change `main` → `master` in Jenkinsfile |
| Stage 3 hangs | Webhook missing | Add SonarQube → Jenkins webhook (Step 10 Part A) |
| Stage 5 fails | SSH not working | Re-run the Ansible `ping` test (Step 7) |
| SonarQube won't start | Memory / limits | Check `vm.max_map_count` + use `t2.medium` |

---

## 7. Definition of Done

- [ ] All 5 pipeline stages green in Jenkins
- [ ] SonarQube shows `EcommerceApp` project with Quality Gate result
- [ ] App loads at `http://<App-Server-IP>:8080`
- [ ] All screenshots taken (see `SCREENSHOTS-GUIDE.md`)
- [ ] Theory document completed (see `THEORY-DELIVERABLES.md`)
