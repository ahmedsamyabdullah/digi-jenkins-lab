# 🧪 Lab 2: Containerization & Code Quality

> **Topics Covered:** Docker · SonarQube · Private GitHub Repo · Jenkins Credentials · Image Scanning · Deployment

---

## 📋 Prerequisites

Before starting this lab, make sure you have completed **Lab 1** and have the following ready:

- ✅ A GitHub repository with your application code (from Lab 1)
- ✅ A Personal Access Token (PAT) generated in Lab 1
- ✅ Jenkins up and running
- ✅ Docker installed on the Jenkins server/VM
- ✅ A Docker Hub account

---

## 🗂️ Lab Overview

| Task | Description |
|------|-------------|
| [Task 1](#task-1--secure-access-to-private-repository) | Secure access to a private GitHub repo using PAT |
| [Task 2](#task-2--static-code-analysis-sonarqube) | Set up SonarQube for static code analysis |
| [Task 3](#task-3--application-containerization-dockerfile) | Write a Dockerfile to package the app |
| [Task 4](#task-4--secure-docker-hub-registry-access) | Store Docker Hub credentials securely in Jenkins |
| [Task 5](#task-5--finalizing-the-pipeline-build--push) | Add Build & Push stages to the Jenkinsfile |
| [Task 6](#task-6--image-vulnerability-scanning-trivy) | Scan the Docker image for vulnerabilities using Trivy |
| [Task 7](#task-7--resource-cleanup) | Clean up dangling Docker images automatically |
| [Task 8](#task-8--deployment) | Deploy the container to a local VM or AWS EC2 |

---

## Task 1 — Secure Access to Private Repository

> **Goal:** Move your app to a private GitHub repo and configure Jenkins to authenticate using a Personal Access Token (PAT).

---

### Step 1 — Make the repository private

1. Go to your GitHub repository **Settings**
2. Scroll down to the **Danger Zone** section
3. Click **Change visibility** → select **Private** → confirm

> [!WARNING]
> After this change, Jenkins will **immediately fail** to pull your code. This is expected. Complete Steps 2 and 3 to restore access.

---

### Step 2 — Add PAT credentials in Jenkins

Navigate to **Manage Jenkins → Credentials → (global) → Add Credentials** and fill in:

| Field | Value |
|-------|-------|
| Kind | `Username with password` |
| Username | Your GitHub username |
| Password | Your PAT from Lab 1 |
| ID | `github-pat-creds` |
| Description | GitHub PAT for private repo access |

Click **Create**.

---

### Step 3 — Update your Jenkinsfile

Modify the `Checkout` stage to use the new credential ID:

```groovy
stage('Checkout') {
    steps {
        checkout scmGit(
            branches: [[name: 'main']],
            userRemoteConfigs: [[
                url: 'https://github.com/<your-username>/<your-repo>.git',
                credentialsId: 'github-pat-creds'
            ]]
        )
    }
}
```

> [!NOTE]
> Replace `<your-username>/<your-repo>` with your actual GitHub repository path.

---

## Task 2 — Static Code Analysis (SonarQube)

> **Goal:** Detect bugs, code smells, and security vulnerabilities before shipping by integrating SonarQube into the pipeline.

---

### Phase 1 — SonarQube Server Setup

#### Step 1 — Start SonarQube via Docker

Run the following command on your VM:

```bash
docker run -d \
  --name sonarqube \
  -p 9000:9000 \
  sonarqube:lts-community
```

Wait ~60 seconds for SonarQube to fully start, then open:

```
http://<YOUR_VM_IP>:9000
```

> [!NOTE]
> Default credentials are **admin / admin**. You will be forced to change the password on first login.

---

#### Step 2 — Create a SonarQube project

1. Click **Create Project → Manually**
2. Fill in the project details:

| Field | Value |
|-------|-------|
| Project display name | `service-app` |
| Project key | `service-app` |

3. Click **Set Up**
4. Under *"How do you want to analyze your repository?"*, select **With Jenkins**
5. Select your DevOps platform (e.g., **GitHub**)
6. Click **Continue**

---

#### Step 3 — Generate a SonarQube token

1. In the SonarQube UI, go to **My Account → Security → Generate Tokens**
2. Fill in:

| Field | Value |
|-------|-------|
| Token name | `jenkins-sonar-token` |
| Type | `Global Analysis Token` |

3. Click **Generate**

> [!IMPORTANT]
> **Copy the token immediately.** It will never be shown again. Store it somewhere safe before closing the dialog.

---

#### Step 4 — Install SonarQube Scanner plugin in Jenkins

1. Go to **Manage Jenkins → Plugins → Available plugins**
2. Search for **SonarQube Scanner**
3. Select it and click **Install**
4. Restart Jenkins after installation

---

#### Step 5 — Store the SonarQube token in Jenkins

Navigate to **Manage Jenkins → Credentials → (global) → Add Credentials**:

| Field | Value |
|-------|-------|
| Kind | `Secret text` |
| Secret | Paste the token copied from SonarQube |
| ID | `sonar-token` |
| Description | SonarQube analysis token |

Click **Create**.

---

### Phase 2 — Jenkins Global Configuration

#### Step 6 — Register the SonarQube Scanner tool

Go to **Manage Jenkins → Tools** → scroll to **SonarQube Scanner** → click **Add SonarQube Scanner**:

| Field | Value |
|-------|-------|
| Name | `SonarScanner` |
| Install automatically | ✅ Checked |

Click **Save**.

> [!IMPORTANT]
> The name `SonarScanner` is case-sensitive and must match exactly in your Jenkinsfile.

---

#### Step 7 — Configure the SonarQube server in Jenkins

Go to **Manage Jenkins → System** → scroll to **SonarQube servers** → click **Add SonarQube**:

| Field | Value |
|-------|-------|
| Name | `SonarQube-Server` |
| Server URL | `http://<YOUR_VM_IP>:9000` |
| Server authentication token | Select `sonar-token` from the dropdown |

Click **Save**.

> [!WARNING]
> Do **not** use `localhost` or `127.0.0.1` as the server URL if Jenkins is running inside a Docker container. Use the actual VM/host IP address instead.

---

### Phase 3 — Jenkinsfile Configuration

#### Step 8 — Add the SonarQube analysis stage

Add the following stage to your `Jenkinsfile` **after** the Checkout stage:

```groovy
stage('SonarQube Analysis') {
    steps {
        withSonarQubeEnv('SonarQube-Server') {
            sh """
                ${tool 'SonarScanner'}/bin/sonar-scanner \
                  -Dsonar.projectKey=service-app \
                  -Dsonar.sources=./src \
                  -Dsonar.host.url=http://<YOUR_VM_IP>:9000
            """
        }
    }
}
```

> [!NOTE]
> Adjust `-Dsonar.sources` to point to the directory containing your source code.

---

### ✅ Verify SonarQube is Working

1. Run the pipeline — confirm the SonarQube stage **passes**
2. Open `http://<YOUR_VM_IP>:9000` and verify the project analysis appears
3. Now deliberately introduce a bug: open `src/OrderProcessor.php` and **remove the comment hash (`#`)** from a commented-out line
4. Commit and push the change
5. Re-run the pipeline — SonarQube should now **fail**, confirming quality gates are active

---

## Task 3 — Application Containerization (Dockerfile)

> **Goal:** Package the PHP application and its runtime into a portable Docker image.

---

### Step 1 — Create a Dockerfile

In the **root of your repository**, create a file named `Dockerfile` with the following content:

```dockerfile
# Use the official PHP CLI image as the base
FROM php:8.2-cli

# Set the working directory inside the container
WORKDIR /app

# Copy all source code into the container
COPY . .

# Install any required PHP extensions (add as needed)
# RUN docker-php-ext-install pdo pdo_mysql

# Define the command to run the application
CMD ["php", "./src/index.php"]
```

> [!NOTE]
> If your app requires additional PHP extensions (e.g., `pdo`, `mbstring`), uncomment and adjust the `RUN` line above.

---

### Step 2 — Build and test locally (optional but recommended)

```bash
# Build the image
docker build -t service-app:test .

# Run it to verify the output
docker run --rm service-app:test
```

---

### Step 3 — Add a Build stage to your Jenkinsfile

```groovy
stage('Build Docker Image') {
    steps {
        sh 'docker build -t <your-dockerhub-username>/service-app:${BUILD_NUMBER} .'
    }
}
```

> [!NOTE]
> `${BUILD_NUMBER}` is a built-in Jenkins environment variable that tags each image with a unique build number, making rollbacks easy.

---

## Task 4 — Secure Docker Hub Registry Access

> **Goal:** Allow Jenkins to push images to Docker Hub using an Access Token instead of your main password.

---

### Step 1 — Generate a Docker Hub Access Token

1. Log in to [hub.docker.com](https://hub.docker.com)
2. Go to **Account Settings → Security → New Access Token**
3. Fill in:

| Field | Value |
|-------|-------|
| Token name | `jenkins-token` |
| Access permissions | `Read & Write` |

4. Click **Generate** and **copy the token immediately**

---

### Step 2 — Store the token in Jenkins

Navigate to **Manage Jenkins → Credentials → (global) → Add Credentials**:

| Field | Value |
|-------|-------|
| Kind | `Username with password` |
| Username | Your Docker Hub username |
| Password | The Access Token you just generated |
| ID | `dockerhub-creds` |
| Description | Docker Hub access token |

Click **Create**.

---

## Task 5 — Finalizing the Pipeline (Build & Push)

> **Goal:** Add Build and Push stages to your Jenkinsfile so Jenkins produces and publishes a Docker image on every run.


---

## Task 6 — Image Vulnerability Scanning (Trivy)

> **Goal:** Scan the built Docker image for known CVEs before deploying. Fail the pipeline automatically if **CRITICAL** vulnerabilities are found.

---

### Step 1 — Install Trivy on the Jenkins container

```bash
# Add the Trivy apt repository
sudo apt-get install -y wget apt-transport-https gnupg lsb-release

wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -

echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" \
  | sudo tee /etc/apt/sources.list.d/trivy.list

# Install Trivy
sudo apt-get update && sudo apt-get install -y trivy
```

Verify the installation:

```bash
trivy --version
```

---

### Step 2 — Trivy scan stage in Jenkinsfile

Add this stage **after** `Build Docker Image` and **before** `Push to Docker Hub`:

```groovy
stage('Trivy Image Scan') {
    steps {
        sh """
            trivy image \
              --exit-code 1 \
              --severity CRITICAL \
              --no-progress \
              ${IMAGE_TAG}
        """
    }
}
```

| Flag | Meaning |
|------|---------|
| `--exit-code 1` | Fail the pipeline if vulnerabilities are found |
| `--severity CRITICAL` | Only fail on CRITICAL severity issues |
| `--no-progress` | Cleaner output in Jenkins logs |

> [!NOTE]
> To also fail on HIGH severity issues, change the flag to `--severity CRITICAL,HIGH`

---

## Task 7 — Resource Cleanup

> **Goal:** Prevent the Jenkins server from running out of disk space by removing Docker images automatically after each build.

---

### Add a `post` cleanup block to your Jenkinsfile

Use a `post { always { } }` block so cleanup runs **even if the pipeline fails**:

```groovy
post {
    always {
        sh '''
            docker rmi ${IMAGE_TAG} || true
            docker image prune -f
        '''
    }
}
```

Place this block at the **same level as `stages`** inside `pipeline { }`:

| Command | Purpose |
|---------|---------|
| `docker rmi ${IMAGE_TAG}` | Remove the specific image built this run |
| `docker image prune -f` | Remove all dangling (untagged) image layers |
| `\|\| true` | Prevent failure if the image was already removed |

---

## Task 8 — Deployment

> **Goal:** Run the containerized application so it is accessible to users — either on the same VM as Jenkins, or on a remote AWS EC2 instance.

---

### Option A — Local Deployment (Same VM as Jenkins)

#### Step 1 — Ensure Jenkins can access the Docker socket

If Jenkins runs in a Docker container, start it with the Docker socket mounted:

```bash
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts
```

> [!IMPORTANT]
> Mounting `/var/run/docker.sock` allows Jenkins to run Docker commands directly on the host machine.

#### Step 2 — Add a local deploy stage to your Jenkinsfile


> [!NOTE]
> Port `8081` is used to avoid conflict with Jenkins on port `8080`. Change it if needed.

Access the application at: `http://<YOUR_VM_IP>:8081`

---

### Option B — Remote Deployment (AWS EC2)

#### Step 1 — Provision and prepare the EC2 instance

1. Launch an **Ubuntu 22.04** EC2 instance in AWS
2. Allow inbound traffic on **port 80** (HTTP) in the Security Group
3. SSH into the instance and install Docker:

```bash
sudo apt-get update
sudo apt-get install -y docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker ubuntu
```

#### Step 2 — Add the EC2 private key to Jenkins

Navigate to **Manage Jenkins → Credentials → (global) → Add Credentials**:

| Field | Value |
|-------|-------|
| Kind | `SSH Username with private key` |
| ID | `ec2-ssh-key` |
| Username | `ubuntu` |
| Private Key | Paste the full contents of your `.pem` file |

Click **Create**.

#### Step 3 — Install the SSH Agent plugin in Jenkins

1. Go to **Manage Jenkins → Plugins → Available plugins**
2. Search for **SSH Agent**
3. Install and restart Jenkins

#### Step 4 — Add a remote deploy stage to your Jenkinsfile
---

## 🏁 Final Pipeline Stage Order

Your complete Jenkinsfile should have stages in this exact order:

```
1. Checkout
2. SonarQube Analysis
3. Build Docker Image
4. Trivy Image Scan
5. Push to Docker Hub
6. Deploy (Option A or B)
   └── post { always { Cleanup } }
```

---

## 🔑 Credentials Summary

| Credential ID | Kind | Used In | Purpose |
|---------------|------|---------|---------|
| `github-pat-creds` | Username with password | Checkout stage | Pull from private GitHub repo |
| `sonar-token` | Secret text | SonarQube Analysis stage | Authenticate with SonarQube |
| `dockerhub-creds` | Username with password | Push stage | Push image to Docker Hub |
| `ec2-ssh-key` | SSH Username with private key | Deploy stage (Option B only) | SSH into EC2 for remote deploy |