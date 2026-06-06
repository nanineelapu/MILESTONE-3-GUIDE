# STEP-BY-STEP COMMANDS — Full Runbook

> Follow this file **top to bottom**. Commands are grouped by VM.
> OS = Amazon Linux 2023 → use `dnf` (NOT `apt`).
> Login user = `ec2-user` | Key = `devops-key.ppk` (use in PuTTY).
>
> 📸 = take a screenshot here (details in `SCREENSHOTS-GUIDE.md`)

---

# STEP 1 — Launch 3 EC2 Instances (AWS Console, browser)

1. Go to https://console.aws.amazon.com → search **EC2** → set your Region (e.g. Mumbai `ap-south-1`).
2. Click **Launch Instance** and create these 3:

| Name | AMI | Type | Storage | Ports to open | Key pair |
|------|-----|------|---------|---------------|----------|
| `Jenkins-Server` | Amazon Linux 2023 | t2.medium | 20 GB | 22, 8080 | `devops-key` (.ppk) |
| `SonarQube-Server` | Amazon Linux 2023 | t2.medium | 20 GB | 22, 9000 | `devops-key` |
| `App-Server` | Amazon Linux 2023 | t2.micro | 10 GB | 22, 8080 | `devops-key` |

- Create the key pair **once** (name `devops-key`, format **.ppk** for PuTTY) and reuse it for all 3.
- In **Network settings → Edit**, add the listed ports as Custom TCP, source `0.0.0.0/0`.

📸 **Screenshot 1:** EC2 Instances page showing all 3 *Running*.

---

# STEP 2 — Connect with PuTTY

1. Install PuTTY from https://www.putty.org (includes PuTTYgen).
2. If your key is `.pem`, convert it: open **PuTTYgen → Load → select .pem → Save private key → devops-key.ppk**.
3. Get a VM's **Public IPv4** from the EC2 console.
4. In **PuTTY**:
   - Host Name: `ec2-user@<PUBLIC-IP>`
   - Port: `22`
   - Left panel → **Connection → SSH → Auth → Credentials** → Browse → select `devops-key.ppk`
   - Back to **Session** → save name (e.g. `Jenkins-Server`) → **Open** → **Accept**.
5. Repeat for all 3 VMs (save 3 sessions).

📸 **Screenshot 2:** PuTTY terminal logged into `Jenkins-Server` (showing `[ec2-user@ip-... ~]$`).

---

# STEP 3 — VM1 (Jenkins-Server): Install Java, Jenkins, Git, Maven, Ansible
nano setup.sh

chmod +x setup.sh

```bash

#!/bin/bash
set -e

echo "=== Step 1: Updating System ==="
dnf update -y

echo "=== Step 2: Installing Java 21 ==="
dnf install java-21-amazon-corretto -y
java -version

echo "=== Step 3: Adding Jenkins Repo ==="
wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key

echo "=== Step 4: Installing Jenkins ==="
dnf install jenkins -y

echo "=== Step 5: Setting JAVA_HOME for Jenkins ==="
mkdir -p /etc/systemd/system/jenkins.service.d
cat > /etc/systemd/system/jenkins.service.d/override.conf << EOF
[Service]
Environment="JAVA_HOME=/usr/lib/jvm/java-21-amazon-corretto.x86_64"
EOF

echo "=== Step 6: Starting Jenkins ==="
systemctl daemon-reload
systemctl start jenkins
systemctl enable jenkins
systemctl status jenkins

echo "=== Step 7: Installing Git ==="
dnf install git -y
git --version

echo "=== Step 8: Installing Maven ==="
dnf install maven -y
mvn -version

echo "=== Step 9: Installing Ansible ==="
dnf install ansible -y
ansible --version

echo "========================================"
echo "=== ALL DONE! Jenkins Unlock Password ==="
echo "========================================"
cat /var/lib/jenkins/secrets/initialAdminPassword

```bash


Open in browser: `http://<Jenkins-Server-IP>:8080` → "Unlock Jenkins" page.

📸 **Screenshot 3:** `systemctl status jenkins` showing *active (running)*.

---

# STEP 4 — Set Up Jenkins (browser)

1. Paste the unlock password → **Continue**.
2. Click **Install suggested plugins** → wait.
3. Create admin user (REMEMBER these):
   - Username: `admin` | Password: `Admin@123` | Email: `thewebcros@gmail.com`
4. **Save and Finish → Start using Jenkins**.
5. Install extra plugins: **Manage Jenkins → Plugins → Available plugins**, tick:
   - `SonarQube Scanner`
   - `Maven Integration`
   - `Ansible`
   - `SSH Agent`
   - (`Git` and `Pipeline` are usually already installed)
6. Click **Install**, tick **Restart Jenkins when installation is complete**.
7. Log back in.

📸 **Screenshot 4:** Jenkins Dashboard after login.

---

# STEP 5 — VM2 (SonarQube-Server): Install SonarQube

```bash
#!/bin/bash
set -e

echo "=== Update System ==="
dnf update -y

echo "=== Install Java 17 ==="
dnf install -y java-17-amazon-corretto
java -version

echo "=== Configure vm.max_map_count ==="
sysctl -w vm.max_map_count=524288

grep -q "vm.max_map_count=524288" /etc/sysctl.conf || \
echo "vm.max_map_count=524288" >> /etc/sysctl.conf

echo "=== Install Required Packages ==="
dnf install -y wget unzip

echo "=== Download SonarQube 10.4 ==="
cd /tmp
wget -O sonarqube.zip https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-10.4.1.88267.zip

echo "=== Extract SonarQube ==="
rm -rf /opt/sonarqube
unzip -q sonarqube.zip
mv sonarqube-10.4.1.88267 /opt/sonarqube

echo "=== Create Sonar User ==="
id sonar >/dev/null 2>&1 || useradd sonar
chown -R sonar:sonar /opt/sonarqube

echo "=== Start SonarQube ==="
sudo -u sonar /opt/sonarqube/bin/linux-x86-64/sonar.sh start

echo "=== Waiting for startup ==="
sleep 90

echo "=== Status ==="
sudo -u sonar /opt/sonarqube/bin/linux-x86-64/sonar.sh status

echo "=== Access ==="
echo "http://YOUR-PUBLIC-IP:9000"
echo "Username: admin"
echo "Password: admin"
```

Wait ~2-3 min. Open: `http://<SonarQube-Server-IP>:9000`
Login `admin` / `admin` → set new password (e.g. `Admin@123`).

> If it won't start: `cat /opt/sonarqube/logs/sonar.log`

📸 **Screenshot 5:** SonarQube Dashboard after login.

---

# STEP 6 — VM3 (App-Server): Install Tomcat 9

```bash
#!/bin/bash
set -e

echo "=== Step 1: Update System ==="
dnf update -y

echo "=== Step 2: Install Java 21 ==="
dnf install java-21-amazon-corretto -y
java -version

echo "=== Step 3: Download Tomcat ==="
cd /opt
wget https://archive.apache.org/dist/tomcat/tomcat-9/v9.0.85/bin/apache-tomcat-9.0.85.tar.gz

echo "=== Step 4: Extract Tomcat ==="
tar -xvf apache-tomcat-9.0.85.tar.gz
mv apache-tomcat-9.0.85 tomcat

echo "=== Step 5: Set Permissions ==="
chmod +x /opt/tomcat/bin/*.sh

echo "=== Step 6: Start Tomcat ==="
/opt/tomcat/bin/startup.sh

echo "========================================"
echo "Tomcat is running!"
echo "Open browser: http://<VM3-IP>:8080"
echo "========================================"```

Open: `http://<App-Server-IP>:8080` → Tomcat welcome page.

> If `wget` 404s, the version was removed — get the current v9 number from https://tomcat.apache.org

📸 **Screenshot 6:** Tomcat welcome page in browser.

---

# STEP 7 — Password-less SSH: Jenkins → App-Server + Ansible inventory

### On VM1 (Jenkins-Server):
```bash
sudo su -

# Create SSH key for the 'jenkins' user
mkdir -p /var/lib/jenkins/.ssh
chown jenkins:jenkins /var/lib/jenkins/.ssh
chmod 700 /var/lib/jenkins/.ssh
sudo -u jenkins ssh-keygen -t rsa -b 2048 -f /var/lib/jenkins/.ssh/id_rsa -N ""

# Show public key (COPY the whole line)
cat /var/lib/jenkins/.ssh/id_rsa.pub
```

### On VM3 (App-Server) — paste the copied key:
```bash
echo "PASTE_THE_COPIED_PUBLIC_KEY_HERE" >> /home/ec2-user/.ssh/authorized_keys
```

### Back on VM1 (Jenkins-Server):
```bash
# Test login (use App-Server PUBLIC IP)
sudo -u jenkins ssh -o StrictHostKeyChecking=no ec2-user@<App-Server-IP> "echo SUCCESS"

# Create Ansible inventory
mkdir -p /etc/ansible
cat > /etc/ansible/hosts <<EOF
[appserver]
<App-Server-IP> ansible_user=ec2-user ansible_ssh_private_key_file=/var/lib/jenkins/.ssh/id_rsa
EOF

# Test Ansible connectivity
sudo -u jenkins ansible -i /etc/ansible/hosts appserver -m ping     # expect: pong / SUCCESS
```

📸 **Screenshot 7:** Ansible `ping` returning `"ping": "pong"` (SUCCESS).

---

# STEP 8 — Create the Ansible Deploy Playbook (on VM1)

```bash
sudo su -

cat > /etc/ansible/deploy.yml <<'EOF'
---
- name: Deploy Ecommerce App to Tomcat
  hosts: appserver
  become: yes
  tasks:

    - name: Stop Tomcat service
      systemd:
        name: tomcat
        state: stopped

    - name: Remove old deployment
      file:
        path: "{{ item }}"
        state: absent
      loop:
        - /opt/tomcat/webapps/ROOT
        - /opt/tomcat/webapps/ROOT.war

    - name: Copy new WAR file to Tomcat
      copy:
        src: "{{ item }}"
        dest: /opt/tomcat/webapps/ROOT.war
        owner: tomcat
        group: tomcat
      with_fileglob:
        - /var/lib/jenkins/workspace/EcommerceApp/target/*.war

    - name: Start Tomcat service
      systemd:
        name: tomcat
        state: started

    - name: Wait for application to be available on port 8080
      wait_for:
        port: 8080
        delay: 10
        timeout: 60
EOF

chown jenkins:jenkins /etc/ansible/deploy.yml
chmod 644 /etc/ansible/deploy.yml

# Validate syntax
sudo -u jenkins ansible-playbook /etc/ansible/deploy.yml --syntax-check
```

📸 **Screenshot 8:** `--syntax-check` showing no errors.

---

# STEP 9 — Connect SonarQube + Maven to Jenkins (browser)

> 💡 This is the **classic / pipeline** method. For the simpler point-and-click
>    Freestyle method, see `ALTERNATIVE-FREESTYLE-METHOD.md` instead.

### A. SonarQube → generate token
SonarQube → avatar → **My Account → Security → Generate Tokens**
Name `jenkins-token`, type **Global Analysis Token** → **Generate** → COPY it.

### B. Jenkins → add token as credential
**Manage Jenkins → Credentials → (global) → Add Credentials**
- Kind: `Secret text` | Secret: *paste token* | ID: `sonar-token`

### C. Jenkins → register SonarQube server
**Manage Jenkins → System → SonarQube servers → Add SonarQube**
- Name: `SonarQube`  *(exact)*
- Server URL: `http://<SonarQube-Server-IP>:9000`
- Auth token: select `sonar-token` → **Save**

### D. Jenkins → configure Maven tool
**Manage Jenkins → Tools → Maven installations → Add Maven**
- Name: `Maven`  *(exact)*
- Tick **Install automatically** → version `3.9.6` → **Save**

> ⚠️ Names `SonarQube` and `Maven` must match the Jenkinsfile exactly.

📸 **Screenshot 9:** SonarQube server config saved in Jenkins.

---

# STEP 10 — Create & Run the Pipeline

### A. SonarQube webhook (so Quality Gate reports back)
SonarQube → **Administration → Configuration → Webhooks → Create**
- Name: `Jenkins`
- URL: `http://<Jenkins-Server-IP>:8080/sonarqube-webhook/`  *(keep trailing slash)*

### B. Create the job
Jenkins → **New Item** → name **`EcommerceApp`** *(exact — playbook depends on it)* → **Pipeline** → OK.
Scroll to **Pipeline** section → Definition: **Pipeline script** → paste the contents of `Jenkinsfile` → **Save**.

### C. Run
Click **Build Now** → open the build → **Console Output** / **Stage View**.
Watch all 5 stages turn green.

### D. Verify
Open `http://<App-Server-IP>:8080` → Ecommerce app loads.

📸 **Screenshot 10:** Jenkins Stage View — all 5 stages green.
📸 **Screenshot 11:** SonarQube project `EcommerceApp` + Quality Gate result.
📸 **Screenshot 12:** Console Output showing Ansible tasks ok/changed.
📸 **Screenshot 13:** Ecommerce app running in browser.

---

## Common Fixes
- **Stage 1 fails:** branch is `master` not `main` → edit Jenkinsfile.
- **Stage 2 fails:** check `mvn -version` and SonarQube server name spelling.
- **Stage 3 hangs:** webhook URL wrong/missing (Step 10A).
- **Stage 5 fails:** re-run the Ansible ping (Step 7); confirm port 22 open.
