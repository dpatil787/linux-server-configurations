# Jenkins Server Configuration

![Linux](https://img.shields.io/badge/Linux-Jenkins-blue?style=for-the-badge&logo=linux)

> **Please note:** This is not a complete Jenkins CI/CD Course or Tutorial. Here we are learning Installation and Configuration of Jenkins Server from a Linux Server admin's point of view.

---

## What is Jenkins?

Jenkins is an open-source, web-based automation tool used for Continuous Integration (CI) and Continuous Delivery (CD).

It automates building, testing, and deploying software.

Jenkins has more than 1500+ plugins which integrate with almost every tool in the DevOps ecosystem — Git, Docker, Kubernetes, and more. For example, if you want to use Git, you install the Git plugin and connect it.

- Jenkins runs on **Port 8080** over TCP and is managed through a web browser
- Port can be changed from `/etc/sysconfig/jenkins`
- Jenkins Agent to Master communication uses **Port 50000**

---

## Why Jenkins is Used

**a) Continuous Integration:**
Developers push code frequently. Jenkins automatically builds and tests that code. If something breaks, the team knows immediately — no manual checking needed.

**b) Continuous Delivery:**
After tests pass, Jenkins can automatically deploy the application to staging or production environments.

**c) Automation:**
Repetitive tasks like running scripts, sending emails, and generating reports can be fully automated through Jenkins.

**d) Extensions via Plugins:**
Jenkins has 1500+ plugins. This means it can connect with Git, Docker, Kubernetes, Slack, and almost every tool used in DevOps.

---

## Jenkins Architecture

**a) Master:**
The central Jenkins server. It manages jobs, schedules builds, stores configurations, and serves the web UI. The master handles lightweight tasks and delegates actual build work to agents.

**b) Agent** (previously called Slave):
A separate machine that executes build jobs. Agents allow Jenkins to scale horizontally. The master assigns jobs to agents and collects results. Agents can be physical machines, VMs, or containers.

**c) Node:**
Any machine — Master or Agent — that can execute Jenkins jobs. The master itself is also a node.

**d) Executor:**
A slot on a node that runs one job at a time. A single node can have multiple executors to run parallel builds.

---

## Key Concepts

**a) Job (Project):**
A task that Jenkins performs. Job types include Freestyle, Pipeline, Multi-configuration, and Folders.

**b) Build:**
An execution of a job. Each build has a number, logs, and a status — Success, Failure, or Unstable.

**c) Workspace:**
The directory on a node where Jenkins downloads source code and runs builds.

**d) Plugin:**
An add-on that extends Jenkins functionality. Examples: Git Plugin, Docker Plugin.

---

## Types of Jobs in Jenkins

**a) Freestyle Job:**
The simplest type of job. Build steps, post-build actions, and triggers are configured through the Web UI. Good for simple tasks like running shell scripts.

**b) Pipeline:**
Defines the entire CI/CD process as code inside a Jenkinsfile. Supports stages, parallel execution, and conditional logic.
- Declarative Pipeline — simpler and structured
- Scripted Pipeline — more flexible, Groovy-based

**c) Multi-Branch Pipeline:**
Automatically creates a pipeline for each branch in a Git repository. The main branch runs production pipelines; feature branches run test pipelines.

**d) Folder:**
Organizes related jobs by project or team. Keeps the Jenkins dashboard clean and manageable.

---

## Jenkins Pipeline

A Pipeline defines the entire CI/CD flow as code.

```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                echo 'Building...'
            }
        }
        stage('Test') {
            steps {
                echo 'Testing...'
            }
        }
        stage('Deploy') {
            steps {
                echo 'Deploying...'
            }
        }
    }
}
```

**Key Pipeline Directives:**

| Directive | Description |
|-----------|-------------|
| `agent` | Specifies where the pipeline or a stage runs — agent any, agent none, or a labeled node |
| `stages` | Container that holds all stage blocks. Defines the sequence of work. Makes the flow visible in the UI |
| `stage` | One logical phase — compile, test, deploy. Can have its own agent, environment, or conditions |
| `steps` | Actual commands inside a stage — shell commands, built-in Jenkins functions, or plugin steps |
| `post` | Actions after the pipeline completes — success, failure, or always |
| `environment` | Sets environment variables available throughout the pipeline |
| `when` | Controls whether a stage runs based on a condition. Skips the stage if condition is false |

---

## Common Use Cases

**a) Build and Test:**
Every time a developer pushes code to Git, Jenkins pulls the code, compiles it, runs tests, and creates deployable artifacts.

- Detects code changes via webhook or polling
- Checks out source code from Git
- Compiles using build tools like Maven for Java or npm for Node.js
- Runs unit tests to catch bugs early
- Produces artifacts like JAR files, Docker images, or binaries
- Stores artifacts for deployment

**b) Deploy:**
After tests pass, Jenkins deploys the application to development, staging, or production.

- Takes artifacts from the build stage
- Copies files to target servers via scp or rsync
- Restarts services like Tomcat, Nginx, or systemd services
- For containers: builds Docker images, pushes to registry, and updates Kubernetes manifests
- Supports rolling updates and blue-green deployments

**c) Notifications:**
Jenkins notifies the team about build status without anyone having to manually check.

- Sends build status, build number, time taken, and log links after pipeline completion
- Supports Teams, Slack, email, and SMS for critical failure alerts

**d) Scheduled Jobs:**
Similar to cron, Jenkins can run jobs on a schedule — useful for maintenance and reporting tasks.

- Configured with cron syntax
- Runs independently without any code change
- Common examples: database backup at 2 AM, report generation at 7 PM

---

## Key Files and Directories

| Path | Description |
|------|-------------|
| `/var/lib/jenkins` | Jenkins home directory |
| `/var/lib/jenkins/secrets/initialAdminPassword` | Initial admin password |
| `/var/lib/jenkins/plugins` | Installed plugins |
| `/var/lib/jenkins/jobs` | Job configurations |
| `/var/log/jenkins/jenkins.log` | Jenkins service logs |

---

# Practical

## Step 1 — Set Hostname and Install Java

```bash
hostnamectl set-hostname JENKINSSERVER
reboot
```

Jenkins requires Java 17. Install it before Jenkins.

> Official Documentation: https://www.jenkins.io/doc/book/installing/linux/

```bash
dnf install java-17-openjdk -y

java --version
# openjdk version "17.0.18" 2026-01-20 LTS
# OpenJDK Runtime Environment (Red_Hat-17.0.18.0.8-2) (build 17.0.18+8-LTS)
# OpenJDK 64-Bit Server VM (Red_Hat-17.0.18.0.8-2) (build 17.0.18+8-LTS, mixed mode, sharing)
```

---

## Step 2 — Add Repository and Install Jenkins

```bash
# Add Jenkins repository so the package manager knows where to download Jenkins from
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/rpm-stable/jenkins.repo

# Update all packages to latest versions
yum upgrade -y

# Import Jenkins GPG key to verify the package is authentic and signed by Jenkins
rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key

# Install Jenkins
dnf install jenkins -y

# Enable Jenkins to start on boot and start it now
systemctl enable jenkins
systemctl start jenkins
systemctl status jenkins
```

---

## Step 3 — Configure Firewall

Jenkins web interface runs on port 8080. Agents communicate with master on port 50000. Both need to be open in the firewall.

```bash
firewall-cmd --permanent --add-service=jenkins --zone=public
# OR explicitly open port 8080
firewall-cmd --add-port=8080/tcp --zone=public --permanent

firewall-cmd --reload
firewall-cmd --list-all
```


---

## Step 4 — Access Jenkins Web UI

```bash
# Get the initial admin password generated during installation
cat /var/lib/jenkins/secrets/initialAdminPassword

# Check VM IP address
ifconfig
```

Open a browser and access Jenkins using the VM IP:

```
http://<your-vm-ip>:8080
```

The Jenkins unlock screen appears and asks for the admin password.

![Jenkins UI](images/2-jenkinsui.png)

After entering the password, Jenkins asks to install plugins. Select **Install suggested plugins** — this installs the most commonly used plugins automatically.

Once plugins are installed, Jenkins asks to create the first admin user. After that, it shows the Jenkins URL.

![Jenkins URL](images/3-jenkinsurl.png)

Jenkins is now accessible to all users on the network at `http://<your-vm-ip>:8080`.

---

# Pipeline

## Step 5 — Create First Pipeline

On the Jenkins Dashboard, click **New Item**, select **Pipeline**, and name it `my-first-pipeline`.

![First Pipeline](images/4-firstpipeline.png)

On the next page, add the pipeline script.

![Pipeline Script](images/5-pipelinescript.png)

Click **Build Now**. A build number appears in the build history on the left side.

![Build History](images/6-buildhistory.png)

Click on the build number (e.g., #1) to open it.

In **Console Output**, the result is visible. The pipeline ran all three stages — Build, Test, Deploy — and printed the echo messages.

![Console Output](images/7-consoleoutput.png)

The stage view shows each stage separately with pass or fail status.

![Pipeline Visualization](images/8-pipelinevisualization.png)

---

## Result

- Jenkins installed and running on port 8080
- Web UI accessible from browser
- Admin user created and authenticated
- First pipeline executed successfully — Build → Test → Deploy stages completed
- Console output and stage visualization verified

**This setup can be extended to:**
- Full CI/CD pipelines triggered by Git webhooks
- Docker image build and push automation
- Kubernetes deployment automation
- Slack or email notifications on build results

---

# Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| Jenkins service fails to start | Wrong Java version | Install Java 17 — `dnf install java-17-openjdk -y` |
| Cannot access port 8080 | Firewall blocking port | `firewall-cmd --add-port=8080/tcp --zone=public --permanent` |
| Admin password not found | First login | Read from `/var/lib/jenkins/secrets/initialAdminPassword` |
| Build fails immediately | Missing plugin or wrong config | Check Console Output for exact error message |

---

*Document prepared as part of DevOps Home Lab — Linux Server Configuration Series*
