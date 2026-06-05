# SCREENSHOTS GUIDE — What to Capture & Submit

> Take these in order as you do `STEP-BY-STEP-COMMANDS.md`.
> Save each as `Screenshot-XX-description.png`. These prove every part of the pipeline works.

---

## Setup Phase

| # | When | What the screenshot must show |
|---|------|-------------------------------|
| 1 | After Step 1 | **AWS EC2 → Instances** page with all **3 instances Running** (Jenkins-Server, SonarQube-Server, App-Server) |
| 2 | After Step 2 | **PuTTY terminal** logged into Jenkins-Server (prompt `[ec2-user@ip-... ~]$`) |
| 3 | After Step 3 | `systemctl status jenkins` showing **active (running)** in green |
| 4 | After Step 4 | **Jenkins Dashboard** (main page after admin login) |
| 5 | After Step 5 | **SonarQube Dashboard** (after login, in browser) |
| 6 | After Step 6 | **Tomcat welcome page** in browser (`http://App-Server-IP:8080`) |
| 7 | After Step 7 | Ansible **ping** result showing `"ping": "pong"` / SUCCESS |
| 8 | After Step 8 | `ansible-playbook --syntax-check` showing **no errors** |
| 9 | After Step 9 | Jenkins **SonarQube servers** config saved (Manage Jenkins → System) |

## Pipeline Run Phase (the important ones)

| # | When | What the screenshot must show |
|---|------|-------------------------------|
| 10 | After Build | **Jenkins Stage View** — all **5 stages GREEN** |
| 11 | After Build | **SonarQube** project `EcommerceApp` with **Quality Gate** result (Passed) + bugs/vulnerabilities counts |
| 12 | After Build | **Console Output** showing Ansible tasks (`ok` / `changed`) and `Pipeline SUCCESS` |
| 13 | After Build | **Ecommerce app running** in browser at `http://App-Server-IP:8080` |

---

## Optional bonus screenshots (extra marks)

| # | What |
|---|------|
| 14 | Jenkins **installed plugins** list (showing SonarQube Scanner, Ansible, Maven Integration) |
| 15 | SonarQube **webhook** config pointing to Jenkins |
| 16 | Jenkins **credentials** page showing `sonar-token` (Secret text) |
| 17 | EC2 **Security Group** rules showing ports 22 / 8080 / 9000 open |

---

## Submission checklist
- [ ] Screenshots 1–13 captured (mandatory)
- [ ] Each clearly readable (full window, not cropped)
- [ ] Saved with clear names
- [ ] Bundled into your report alongside `THEORY-DELIVERABLES.md`
