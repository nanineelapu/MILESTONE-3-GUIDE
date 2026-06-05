# ALTERNATIVE METHOD — Freestyle Job (Simpler)

> **Use this instead of Steps 9 & 10 in `STEP-BY-STEP-COMMANDS.md`** if you want the
> simple point-and-click approach (no Groovy/Jenkinsfile, no token credential, no
> SonarQube server config, no webhook). It still satisfies the full use case.
>
> ✅ **Steps 1–8 are EXACTLY THE SAME** for both methods — do them first from
>    `STEP-BY-STEP-COMMANDS.md`. Only Steps 9 & 10 differ. This file replaces those two.

---

## Classic vs Freestyle — what's different

| | Classic (Pipeline) | Freestyle (this file) |
|---|---|---|
| Job type | Pipeline + `Jenkinsfile` (Groovy) | Freestyle (UI build steps) |
| SonarQube auth | Token stored as Jenkins credential | Token pasted inline in the command |
| SonarQube server config | Required (Manage Jenkins → System) | Not needed |
| Maven tool config | Required (Manage Jenkins → Tools) | Not needed (uses system `mvn`) |
| Quality Gate | Auto-stops build (webhook + `waitForQualityGate`) | Checked visually on SonarQube dashboard |
| Extra plugins | SonarQube Scanner, Maven Integration, Ansible | None required (uses shell) |
| Difficulty | Higher | **Lowest** |

Both deploy the same app the same way (Ansible → Tomcat). Steps 1–8 unchanged.

---

# STEP 9 (Freestyle) — Generate a SonarQube Token

This is the **only** prep needed — no Jenkins credential or server config.

1. Open SonarQube `http://<SonarQube-Server-IP>:9000` → log in.
2. Click your **avatar (top-right) → My Account → Security**.
3. Under **Generate Tokens**:
   - Name: `ecommerce-token`
   - Type: **Global Analysis Token**
   - Click **Generate** → 📋 **COPY the token** (looks like `squ_xxxxxxxx`). Save it.

> Maven and Ansible are already installed on VM1 (from Step 3). The Freestyle job
> calls them directly via shell, so there's nothing else to configure in Jenkins.

📸 **Screenshot 9:** SonarQube "Tokens" list showing `ecommerce-token` generated.

---

# STEP 10 (Freestyle) — Create & Run the Freestyle Job

### A. Create the job
1. Jenkins Dashboard → **New Item**.
2. Enter name **`EcommerceApp`**  *(exact — the Ansible playbook reads `/var/lib/jenkins/workspace/EcommerceApp/`)*.
3. Select **Freestyle project** → **OK**.

### B. Source Code Management — Stage 1: Clone
- Under **Source Code Management**, select **Git**.
- **Repository URL:** `https://github.com/Msocial123/EcommerceApp.git`
- **Branch Specifier:** `*/main`  *(if the build later says branch not found, change to `*/master`)*

### C. Build Steps — Stages 2, 3, 4
Scroll to **Build → Add build step → Execute shell**. Add **three** shell steps in this order.

**Build Step 1 — Code Quality (SonarQube).** Replace `<VM2-IP>` and `<TOKEN>`:
```bash
mvn clean verify sonar:sonar \
  -Dsonar.projectKey=EcommerceApp \
  -Dsonar.host.url=http://<VM2-IP>:9000 \
  -Dsonar.login=<TOKEN> \
  -DskipTests
```

**Build Step 2 — Maven Build (package the .war):**
```bash
mvn clean package -DskipTests
```

**Build Step 3 — Ansible Deploy:**
```bash
ansible-playbook -i /etc/ansible/hosts /etc/ansible/deploy.yml
```

Click **Save**.

> 🔎 **About the Quality Gate (Stage 3 in the use case):** SonarQube analyses the code
> in Build Step 1 and shows the Quality Gate result (Passed/Failed) plus all bugs,
> vulnerabilities and security issues on its dashboard. You demonstrate the
> "PASS → continue / FAIL → stop" rule by checking that dashboard. If your trainer
> specifically wants the build to **auto-stop** on a failed gate, use the classic
> method (Steps 9 & 10 + `Jenkinsfile`) instead.

### D. Run it
- Click **Build Now**.
- Open the build number → **Console Output** and watch:
  `Cloning repo → SonarQube ANALYSIS SUCCESSFUL → Maven BUILD SUCCESS → Ansible PLAY RECAP (ok/changed)`.

### E. Verify the app is live
Open `http://<App-Server-IP>:8080` → the Ecommerce app loads.

📸 **Screenshot 10:** Build marked **SUCCESS** (blue/green ball) in Build History.
📸 **Screenshot 11:** SonarQube project `EcommerceApp` + Quality Gate result (bugs/vulnerabilities).
📸 **Screenshot 12:** Console Output showing Ansible `PLAY RECAP` (ok/changed) + `Finished: SUCCESS`.
📸 **Screenshot 13:** Ecommerce app running in browser.

---

## Common Fixes (Freestyle)
- **Clone fails / branch not found:** change `*/main` → `*/master` in SCM.
- **SonarQube step fails:** check `<VM2-IP>` and the token are correct, and port 9000 is open.
- **`mvn` not found:** confirm `mvn -version` works on VM1 (re-run `dnf install maven -y`).
- **Ansible step fails:** re-run the ping test from Step 7; confirm port 22 open on App-Server.
- **`.war` not deployed:** confirm `mvn clean package` produced a file under `target/*.war`.

---

## Note for your theory document
If you submit using **this Freestyle method**, adjust two of the deliverables in
`THEORY-DELIVERABLES.md`:
- **Deliverable 3 (Tool Integration):** SonarQube is invoked via the Maven scanner
  with the token passed on the command line (not a stored credential/webhook).
- **Deliverable 6 (Security):** mention that storing the token as a Jenkins
  credential (the classic method) would be the production best practice.
