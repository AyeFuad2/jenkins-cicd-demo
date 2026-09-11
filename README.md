# jenkins-cicd-demo
# Jenkins CI/CD Pipeline — GitHub → Docker → AWS ECR

A hands-on CI/CD project that demonstrates how a source-code change can move from **GitHub → Jenkins → Docker → Amazon ECR** using an AWS EC2 build server.

The goal of this project was to build the **continuous integration foundation of a future Kubernetes/EKS deployment pipeline**. Every source change can be converted into a traceable, build-numbered Docker image, while Jenkins uses temporary AWS credentials from an EC2 IAM role instead of storing long-term AWS access keys.

---

## 📌 Project Overview

This project started with a simple static web application and evolved into a working cloud-based CI pipeline.

The application is packaged into a Docker container using NGINX. Jenkins runs on an Amazon Linux 2023 EC2 instance and retrieves the source code from GitHub. Jenkins then builds the Docker image, authenticates to Amazon ECR using the EC2 instance role, tags the image with the Jenkins build number, and publishes it to a private ECR repository.

The pipeline was later extended to automatically detect source changes using Jenkins SCM polling.

A smoke-test quality gate was also added to prevent a broken container image from being published.

### Final workflow

```text
Developer
    │
    │ git push
    ▼
 GitHub
    │
    │ Jenkins SCM polling
    ▼
 Jenkins
    │
    ├── Verify Git
    ├── Verify Docker
    ├── Verify AWS CLI
    │
    ▼
 Docker Build
    │
    ▼
 Smoke Test
    │
    ├── PASS ────────────────┐
    │                        │
    │                        ▼
    │                  Amazon ECR
    │                        │
    │                  Build # / Tag
    │
    └── FAIL → Stop pipeline
```

---

# 🏗️ Architecture

```text
                         ┌─────────────────────┐
                         │      Developer      │
                         │                     │
                         │  Modify index.html  │
                         └──────────┬──────────┘
                                    │
                                    │ git push
                                    ▼
                         ┌─────────────────────┐
                         │       GitHub        │
                         │                     │
                         │  Source Repository  │
                         │  index.html         │
                         │  Dockerfile         │
                         │  Jenkinsfile        │
                         └──────────┬──────────┘
                                    │
                              SCM Polling
                                    │
                                    ▼
                    ┌─────────────────────────────┐
                    │       Amazon EC2             │
                    │                              │
                    │       Jenkins Server         │
                    │                              │
                    │  ┌────────────────────────┐  │
                    │  │ Jenkins Pipeline       │  │
                    │  │                        │  │
                    │  │ Verify Tools           │  │
                    │  │ Build Docker Image     │  │
                    │  │ Smoke Test             │  │
                    │  │ Push Image             │  │
                    │  └───────────┬────────────┘  │
                    │              │               │
                    │       IAM Instance Role       │
                    └──────────────┼────────────────┘
                                   │
                                   │ Temporary AWS Credentials
                                   ▼
                         ┌─────────────────────┐
                         │    Amazon ECR       │
                         │                     │
                         │ Private Repository  │
                         │                     │
                         │ :1                  │
                         │ :2                  │
                         │ :3                  │
                         │ ...                 │
                         └─────────────────────┘
```

---

# 🎯 Project Goals

The project was designed to demonstrate practical experience with:

* AWS infrastructure
* IAM roles
* EC2
* AWS Systems Manager Session Manager
* Amazon ECR
* Docker
* Jenkins
* Jenkins Declarative Pipelines
* Git/GitHub
* Linux administration
* AWS CLI
* Container image versioning
* CI/CD automation
* SCM polling
* Build troubleshooting
* Smoke testing
* Infrastructure security
* Artifact traceability

The project also serves as the **CI foundation for a future Amazon EKS/Kubernetes deployment**.

---

# 🧰 Technologies Used

| Technology          | Purpose                                   |
| ------------------- | ----------------------------------------- |
| AWS EC2             | Jenkins build server                      |
| Amazon Linux 2023   | EC2 operating system                      |
| Jenkins             | CI/CD automation                          |
| Docker              | Containerization                          |
| Amazon ECR          | Private Docker image registry             |
| IAM                 | AWS permissions and temporary credentials |
| AWS Systems Manager | Secure EC2 access                         |
| Git                 | Source control                            |
| GitHub              | Source repository                         |
| NGINX               | Web server                                |
| AWS CLI             | AWS/ECR interaction                       |
| Bash                | Server and pipeline commands              |
| Groovy              | Jenkinsfile pipeline definition           |
| HTML/CSS            | Demo application                          |

---

# 🚀 Step 0 — Define the CI/CD Objective

The first objective was to understand the difference between having working source code and having a repeatable packaged artifact.

The project was designed so that a GitHub commit eventually becomes a numbered container image.

The intended flow was:

```text
Git Commit
     ↓
GitHub
     ↓
Jenkins
     ↓
Docker Build
     ↓
Smoke Test
     ↓
Build Number
     ↓
Amazon ECR
```

This creates a traceable relationship between source changes and container artifacts.

---

# ☁️ Step 1 — Provision the AWS Build Environment

The first major stage was creating the AWS infrastructure required to run Jenkins.

The environment consisted of:

* IAM role
* Amazon ECR repository
* EC2 instance
* Security group
* Systems Manager Session Manager

The purpose was to give Jenkins a secure environment from which it could build and publish Docker images.

---

## 🔐 IAM Role

I created an EC2 IAM role named:

```text
jenkins-ec2-role
```

The role was configured with:

```text
AmazonSSMManagedInstanceCore
AmazonEC2ContainerRegistryPowerUser
```

### Why?

The Systems Manager policy allowed the EC2 instance to be managed through Session Manager.

The ECR policy allowed the instance to interact with Amazon ECR.

This meant Jenkins could obtain temporary AWS credentials through the EC2 instance role instead of storing permanent AWS access keys.

```text
EC2
 │
 └── IAM Instance Role
        │
        ├── Session Manager access
        │
        └── ECR permissions
```

This was an important security decision because credentials were not hard-coded into the Jenkins pipeline.

---

# 📦 Amazon ECR

I created a private ECR repository:

```text
jenkins-cicd-demo
```

The repository was configured with:

* Private visibility
* Immutable image tags
* AES-256 encryption

### Why immutable tags?

Each Jenkins build uses its build number as the Docker image tag.

For example:

```text
jenkins-cicd-demo:1
jenkins-cicd-demo:2
jenkins-cicd-demo:3
```

With immutable tags, a later build cannot overwrite an existing image version.

This makes it easier to trace:

```text
Git Commit
    ↓
Jenkins Build #2
    ↓
ECR Image :2
```

The project specifically used immutable tags so each successful build remained traceable.

---

# 🖥️ EC2 Jenkins Build Server

I launched an EC2 instance to host Jenkins.

Configuration included:

```text
Instance:
Amazon Linux 2023
t3.medium
16 GiB gp3 storage
```

The instance was assigned:

```text
jenkins-ec2-role
```

I also created a security group:

```text
jenkins-lab-sg
```

Jenkins was exposed on:

```text
TCP 8080
```

with access restricted to my IP.

SSH inbound access was not required because I used AWS Systems Manager Session Manager.

---

# 🔗 AWS Systems Manager Session Manager

Instead of relying on SSH keys, I connected to the EC2 instance through:

```text
AWS Systems Manager → Session Manager
```

This allowed me to open a shell on the EC2 instance without maintaining an SSH key pair or opening inbound SSH access.

I verified the AWS identity using:

```bash
aws sts get-caller-identity
```

The returned identity confirmed that the instance was using the expected IAM role.

This proved the server could receive temporary AWS credentials through its instance profile.

### Screenshot

> Add screenshot here:
>
> `screenshots/aws-session-manager.png`

---

# 🐳 Step 2 — Containerize the Application

Before introducing Jenkins, I built and tested the application locally.

The application is a small static web page called:

```text
GitHub to ECR CI Pipeline
```

It contains a visible version badge:

```text
Version 1
```

The page was intentionally simple because the purpose of the project was to demonstrate the CI/CD workflow rather than build a complex frontend.

---

# 📄 index.html

The application consists of a single HTML file.

The page displays:

```text
Cloud-native CI foundation

GitHub to ECR CI Pipeline

This page was packaged by Docker and published
by a Jenkins CI/CD pipeline on AWS.

Version 1
```

Later, the page was changed to:

```text
Version 2
```

This provided a visible way to prove that a new Git commit produced a new container image.

The project notes specifically used the Version 1 → Version 2 change as the observable application revision.

---

# 🐋 Dockerfile

The Dockerfile uses a version-pinned NGINX Alpine image:

```dockerfile
FROM nginx:stable-alpine3.24

COPY index.html /usr/share/nginx/html/index.html
```

This provides a lightweight web server and copies the application into NGINX's default web root.

### Why NGINX?

The application is static, so a lightweight NGINX container is enough to serve it.

---

# 🚫 .dockerignore

The project also includes:

```text
.git
Jenkinsfile
```

This prevents Git metadata and the Jenkins pipeline definition from being included in the Docker build context/image.

---

# 🧪 Local Docker Testing

Before involving Jenkins, I tested the Docker image locally.

### Build the image

```bash
docker build -t jenkins-cicd-demo:local .
```

This builds the Docker image using the local Dockerfile.

The `.` represents the current directory as the Docker build context.

---

## Run the container

```bash
docker run --name jenkins-cicd-demo -d -p 8081:80 jenkins-cicd-demo:local
```

This maps:

```text
Windows Port 8081
       ↓
Container Port 80
```

The application could then be accessed at:

```text
http://localhost:8081
```

The local test confirmed that the application could successfully run from the Docker image before introducing Jenkins.

### Screenshot

> Add screenshot here:
>
> `screenshots/local-docker-app.png`

---

# 🧹 Container Cleanup

After testing:

```bash
docker rm -f jenkins-cicd-demo
```

Then:

```bash
docker ps
```

confirmed the temporary container was removed while the Docker image remained available locally.

---

# 🗂️ Step 3 — Create the GitHub Repository

I created a public GitHub repository:

```text
jenkins-cicd-demo
```

The repository initially started empty so the local project files and Jenkinsfile could be committed together.

The local project was initialized with Git:

```bash
git init -b main
```

Files were staged:

```bash
git add .
```

Then committed:

```bash
git commit -m "First commit"
```

The GitHub remote was configured and the first commit was pushed:

```bash
git remote add origin <repository>
git push -u origin main
```

The final repository contains the application and CI/CD configuration together.

---

# ☕ Step 4 — Build the Jenkins Server

The EC2 instance was then converted into the Jenkins build server.

The server needed:

* Java
* Jenkins
* Git
* Docker
* AWS CLI

---

# ☕ Java 21

Jenkins requires Java to run.

During setup I initially encountered a package installation problem with:

```bash
java-21-openjdk
```

The package was not available under that name on the Amazon Linux environment.

I diagnosed the package issue and used the Amazon Corretto package instead:

```bash
sudo dnf install -y java-21-amazon-corretto-devel
```

Then verified Java:

```bash
java -version
```

This was a useful Linux troubleshooting exercise because it required understanding the difference between the generic package name and the Amazon Linux package repository.

---

# 🐳 Docker Permission Troubleshooting

I also encountered:

```text
permission denied
```

when working with Docker.

The issue was related to the Linux user attempting to communicate with the Docker daemon.

For the Jenkins service user, I added Jenkins to the Docker group:

```bash
sudo usermod -a -G docker jenkins
```

Then restarted Jenkins:

```bash
sudo systemctl restart jenkins
```

This allowed the Jenkins process to interact with Docker without requiring the pipeline to run Docker as root.

The project troubleshooting notes specifically identified this as a Docker daemon socket permission issue for the Jenkins service user.

---

# ⚙️ Jenkins Installation

I installed Jenkins on the EC2 instance and configured it as a system service.

The build server was configured with:

```text
Jenkins
Java 21
Git
Docker
AWS CLI
```

The services were enabled so they could start with the system:

```bash
sudo systemctl enable docker
sudo systemctl enable jenkins
```

and started:

```bash
sudo systemctl start docker
sudo systemctl start jenkins
```

---

# 🔍 Build Server Verification

I verified each component individually.

### Jenkins

```bash
sudo systemctl status jenkins
```

### Docker

```bash
docker ps
```

### Git

```bash
git --version
```

### AWS CLI

```bash
aws --version
```

### AWS identity

```bash
aws sts get-caller-identity
```

These checks verified that the EC2 host had the core components required to execute the pipeline.

The Jenkins dashboard was made available on:

```text
Port 8080
```

The Pipeline and Git plugins were also verified in Jenkins.

### Screenshot

> Add screenshot here:
>
> `screenshots/jenkins-dashboard.png`

---

# 🔧 Step 5 — Create the Jenkins Pipeline

The pipeline logic was stored in a file called:

```text
Jenkinsfile
```

Instead of configuring the pipeline entirely inside Jenkins, the pipeline definition was stored with the application source code.

This gives the project a source-controlled CI configuration.

---

# Jenkins Pipeline Structure

The pipeline includes:

```text
Pipeline
│
├── Options
│
├── Triggers
│
├── Environment
│
└── Stages
    │
    ├── Verify Tools
    │
    └── Build and Push Image
```

---

# 🧪 Verify Tools Stage

The first stage checks whether the EC2 build environment is ready.

```groovy
stage('Verify Tools') {
    steps {
        sh 'git --version'
        sh 'docker ps'
        sh 'aws --version'
    }
}
```

This verifies:

```text
Git       → source control available
Docker    → container daemon available
AWS CLI   → AWS integration available
```

This was intentionally placed before the build stage so a missing dependency could be identified early.

---

# 🏗️ Build and Push Image Stage

The second stage builds and publishes the Docker image.

The pipeline constructs the ECR registry:

```text
AWS_ACCOUNT_ID
       +
AWS_DEFAULT_REGION
       +
Amazon ECR registry
```

The image URI is then created using:

```text
Repository + Jenkins BUILD_NUMBER
```

Conceptually:

```text
jenkins-cicd-demo:BUILD_NUMBER
```

---

# 🔐 ECR Authentication

Jenkins authenticates to ECR using:

```bash
aws ecr get-login-password
```

The authentication token is passed into Docker:

```bash
docker login
```

This is important because the pipeline does not need a long-term AWS access key stored inside the Jenkinsfile.

The EC2 instance's IAM role provides the AWS identity.

The project documentation describes this as temporary AWS credentials supplied through the EC2 instance role.

---

# 🐳 Docker Build

Jenkins builds the image:

```bash
docker build -t "${IMAGE_URI}" .
```

The Docker image contains:

```text
NGINX
+
index.html
```

The resulting image receives the Jenkins build number as its tag.

---

# 📤 Push to Amazon ECR

After the image is built:

```bash
docker push "${IMAGE_URI}"
```

Jenkins then outputs the exact image URI:

```text
Published <ECR repository>/<image>:<BUILD_NUMBER>
```

This makes it possible to match the Jenkins build with the ECR artifact.

The publishing stage uses the registry, repository name, and Jenkins build number to create a traceable image URI.

---

# 🔗 Step 6 — Connect Jenkins to GitHub

In Jenkins, I created a Pipeline job:

```text
jenkins-cicd-demo
```

The job was configured using:

```text
Definition:
Pipeline script from SCM

SCM:
Git

Branch:
main

Script Path:
Jenkinsfile
```

This means Jenkins retrieves the pipeline definition directly from GitHub instead of relying on an inline script stored only in Jenkins.

That keeps the CI configuration inside the same Git history as the application.

The project specifically used Pipeline script from SCM so the source revision and pipeline definition remain connected.

---

# ▶️ Step 7 — First Pipeline Run

The first build was triggered manually using:

```text
Build Now
```

I inspected the:

```text
Console Output
```

The pipeline executed:

```text
Verify Tools
      ↓
Docker Build
      ↓
ECR Authentication
      ↓
Docker Push
```

The console output showed the published image URI.

The first successful image used the Jenkins build number as its tag:

```text
:1
```

The image was then visible in Amazon ECR.

This proved the complete:

```text
GitHub → Jenkins → Docker → ECR
```

workflow worked.

### Screenshot

> Add screenshot here:
>
> `screenshots/jenkins-build-success.png`

### Screenshot

> Add screenshot here:
>
> `screenshots/ecr-build-1.png`

---

# 🔄 Step 8 — Automate Source Changes

The initial pipeline required:

```text
Git Push
   ↓
Jenkins
   ↓
Build Now
```

I then removed that manual step by adding SCM polling.

The Jenkinsfile was updated with:

```groovy
triggers {
    pollSCM('H/2 * * * *')
}
```

This tells Jenkins to periodically check the configured GitHub repository.

When Jenkins detects a new commit, it starts another build automatically.

The project used a two-minute hashed polling cadence.

---

# 🆕 Step 9 — Version 2 Release

To prove the automated pipeline worked, I changed the application from:

```html
<span class="badge">Version 1</span>
```

to:

```html
<span class="badge">Version 2</span>
```

Then committed the change:

```bash
git add .
git commit -m "Release version 2"
git push origin main
```

The GitHub push created a new source revision.

Jenkins detected the change through SCM polling.

No additional manual build trigger was required.

The project documentation describes this as closing the manual feedback gap.

---

# 🔢 Build Number → ECR Tag

The automatic Jenkins build used:

```text
BUILD_NUMBER
```

as the image tag.

For example:

```text
Jenkins Build #2
       ↓
ECR Image :2
```

This creates artifact traceability:

```text
GitHub Commit
      ↓
Jenkins Build
      ↓
Docker Image
      ↓
ECR Tag
```

The Jenkins console output and ECR image list can therefore be compared to verify that the correct build produced the correct artifact.

### Screenshot

> Add screenshot here:
>
> `screenshots/ecr-versioned-images.png`

---

# 💎 Step 10 — Smoke-Test Quality Gate

The final enhancement was adding a smoke-test quality gate.

The problem:

A Docker image can successfully build even if the application inside it is broken or contains the wrong content.

A successful Docker build alone does not prove the application works.

The smoke test was designed to validate the application before publishing the image.

The intended logic is:

```text
Docker Build
     ↓
Start Test Container
     ↓
HTTP Request
     ↓
Check Expected Page Content
     │
     ├── PASS → Push to ECR
     │
     └── FAIL → Stop Pipeline
```

The expected page content includes:

```text
GitHub to ECR CI Pipeline
```

This means the pipeline is no longer simply:

```text
Build → Push
```

It becomes:

```text
Build → Test → Push
```

This creates an additional quality-control step before an artifact reaches the registry.

The project notes describe the smoke-test gate as blocking a broken image before publication, with failed builds producing no matching ECR tag.

---

# 🧪 Failure Scenario

One of the most important lessons from the project was that a green Docker build does not necessarily mean the application is healthy.

For example:

```text
Docker Build = SUCCESS
Application = BROKEN
```

Without a smoke test:

```text
Broken Image
     ↓
Amazon ECR
```

With the quality gate:

```text
Broken Image
     ↓
Smoke Test
     ↓
FAIL
     ↓
Pipeline Stops
     ↓
No Image Published
```

This makes the CI pipeline more reliable.

---

# 🔐 Security Considerations

Several security decisions were intentionally included in the project.

### IAM role instead of AWS access keys

The EC2 instance uses an IAM instance role to obtain temporary AWS credentials.

No long-term AWS access keys are required in the Jenkinsfile.

### Private ECR repository

The Docker images are stored in a private Amazon ECR repository.

### Immutable tags

Build artifacts cannot simply be overwritten with another image using the same tag.

### Restricted Jenkins access

Port 8080 is restricted to my IP through the EC2 security group.

### No inbound SSH requirement

The EC2 instance is accessed through Systems Manager Session Manager instead of opening SSH.

### Credentials

Sensitive credentials such as Jenkins administrator passwords and AWS credentials should never be committed to GitHub.

---

# 🛠️ Troubleshooting Experience

This project also involved troubleshooting the actual build environment instead of only following a happy-path deployment.

## Java package issue

Initial package:

```bash
sudo dnf install -y java-21-openjdk
```

Result:

```text
Unable to find a match
```

Resolution:

```bash
sudo dnf install -y java-21-amazon-corretto-devel
```

Then:

```bash
java -version
```

verified the installation.

---

## Docker permission issue

Problem:

```text
permission denied
```

Resolution:

```bash
sudo usermod -a -G docker jenkins
```

Then:

```bash
sudo systemctl restart jenkins
```

This allowed the Jenkins service user to communicate with Docker.

---

## Jenkins service troubleshooting

I used:

```bash
sudo systemctl status jenkins
```

and:

```bash
sudo journalctl -u jenkins --no-pager | tail -n 50
```

to investigate service failures.

For Docker:

```bash
sudo systemctl status docker
```

and:

```bash
sudo journalctl -u docker --no-pager | tail -n 50
```

These troubleshooting steps reinforced the importance of checking:

```text
Service status
      ↓
System logs
      ↓
Dependency
      ↓
Permissions
      ↓
Configuration
```

instead of blindly restarting services.

---

# 📁 Repository Structure

```text
jenkins-cicd-demo/
│
├── index.html
│
├── Dockerfile
│
├── .dockerignore
│
├── Jenkinsfile
│
└── screenshots/
    ├── aws-session-manager.png
    ├── jenkins-dashboard.png
    ├── local-docker-app.png
    ├── jenkins-build-success.png
    ├── ecr-build-1.png
    └── ecr-versioned-images.png
```

---

# 📸 Screenshots

Screenshots can be added to document the major milestones of the project.

## 1. AWS Session Manager

```text
screenshots/aws-session-manager.png
```

Shows:

* EC2 Session Manager
* AWS CLI
* `aws sts get-caller-identity`
* IAM role identity

---

## 2. Jenkins Dashboard

```text
screenshots/jenkins-dashboard.png
```

Shows:

* Jenkins running on EC2
* Pipeline job
* Jenkins dashboard

---

## 3. Local Docker Application

```text
screenshots/local-docker-app.png
```

Shows:

```text
GitHub to ECR CI Pipeline
Version 1
```

running from the Docker container.

---

## 4. Jenkins Successful Build

```text
screenshots/jenkins-build-success.png
```

Shows:

* Verify Tools stage
* Docker build
* ECR authentication
* Docker push
* Published image URI

---

## 5. ECR Build #1

```text
screenshots/ecr-build-1.png
```

Shows the first build-numbered Docker image.

---

## 6. Automatic Build

```text
screenshots/ecr-versioned-images.png
```

Shows multiple build-numbered images in ECR.

Example:

```text
1
2
3
```

---

# 📊 What I Built

| Component       | Result                       |
| --------------- | ---------------------------- |
| AWS IAM         | EC2 role created             |
| EC2             | Jenkins build server         |
| Session Manager | Secure server access         |
| Jenkins         | CI/CD automation             |
| Docker          | Application containerization |
| GitHub          | Source control               |
| ECR             | Private image registry       |
| SCM Polling     | Automatic source detection   |
| Build Numbers   | Artifact versioning          |
| Smoke Test      | Pre-publication quality gate |
| NGINX           | Static application server    |

---

# 🧠 Key Concepts Learned

### Continuous Integration

A source change can automatically trigger a build and produce a new artifact.

### Infrastructure

The CI server itself runs on AWS infrastructure rather than a local workstation.

### IAM

AWS access can be provided through instance roles instead of hard-coded credentials.

### Containers

The application is packaged into a portable Docker image.

### Artifact Traceability

Jenkins build numbers become ECR image tags.

### Source-Controlled Pipelines

The Jenkinsfile lives alongside the application code.

### Automated Feedback

SCM polling removes the need to manually click **Build Now** after every source change.

### Quality Gates

A smoke test can prevent an unhealthy image from being published.

### Troubleshooting

Service status, logs, permissions, package managers, and system dependencies all play a role in operating a CI server.

---

# 🔭 Future Improvements

This project currently establishes the **CI portion** of a larger cloud-native delivery system.

Future improvements could include:

### 1. GitHub Webhooks

Replace SCM polling with GitHub webhooks for event-driven builds.

```text
GitHub Push
     ↓
Webhook
     ↓
Jenkins
```

### 2. Amazon EKS Deployment

Deploy successful ECR images to Kubernetes:

```text
GitHub
   ↓
Jenkins
   ↓
Docker
   ↓
ECR
   ↓
EKS
   ↓
Kubernetes
```

### 3. Infrastructure as Code

Recreate the AWS environment using Terraform:

```text
Terraform
   ↓
IAM
EC2
ECR
Security Groups
```

### 4. Better Automated Testing

Add:

* HTTP health checks
* HTML validation
* Container health checks
* Integration tests

### 5. Monitoring

Integrate:

* CloudWatch
* Jenkins metrics
* Application logs
* Container monitoring

### 6. Deployment Strategies

Eventually implement:

* Rolling deployments
* Blue/green deployments
* Canary deployments

---

# 💡 Final Architecture

The final concept demonstrated by this project is:

```text
                 SOURCE
                   │
                   ▼
                GitHub
                   │
                   │ SCM Polling
                   ▼
                Jenkins
                   │
          ┌────────┴────────┐
          │                 │
     Verify Tools       Build Image
                            │
                            ▼
                       Smoke Test
                            │
                       ┌────┴────┐
                       │         │
                     PASS       FAIL
                       │         │
                       ▼         ▼
                     ECR       STOP
                       │
                       ▼
                Build-Numbered
                   Image
```

---

# 🎯 Project Outcome

The final result is a working CI/CD foundation where a source change can become a tested, versioned Docker image in Amazon ECR.

The pipeline demonstrates:

```text
GitHub
  ↓
Jenkins
  ↓
Docker
  ↓
Smoke Test
  ↓
Amazon ECR
```

Each successful build produces a traceable image based on the Jenkins build number, while the EC2 IAM role provides temporary AWS credentials without requiring long-term AWS access keys.

The project also gave me hands-on experience troubleshooting the underlying Linux build server, including Java package availability, Docker permissions, Jenkins services, AWS identity, and CI pipeline failures.

---

## 🔗 Repository

**jenkins-cicd-demo**

Built as a hands-on DevOps project to understand how application source code becomes a cloud-ready container artifact.

---

## 👨‍💻 Author

**Fuad Aye**

Cloud / DevOps / Technical Support Engineer

Interests:

`AWS` `Docker` `Jenkins` `Kubernetes` `Terraform` `Linux` `CI/CD`
