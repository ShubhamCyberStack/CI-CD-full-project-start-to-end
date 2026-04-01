
---

```markdown
# CI/CD Pipeline Project – Demo APK

This project sets up a complete **Continuous Integration (CI)** and **Continuous Deployment (CD)** pipeline using AWS services.  
The goal: every time you commit code to CodeCommit, it automatically builds with CodeBuild, stores artifacts in S3, and deploys to EC2 via CodeDeploy.

---

## 📖 AWS Services Used

- **AWS CodeCommit** – Private Git repository service to store your source code.  
- **AWS IAM** – Identity and Access Management, used to create users, roles, and attach policies.  
- **AWS CodeBuild** – Fully managed build service that compiles source code, runs tests, and produces build artifacts.  
- **Amazon S3** – Storage service used to store build artifacts (ZIP files).  
- **AWS CodeDeploy** – Automates deployment of applications to EC2/on‑premises servers.  
- **Amazon EC2** – Virtual servers where your application runs.  
- **Amazon EventBridge** – Event bus that triggers pipeline runs automatically when commits are pushed.

---

## 🛠️ Step 1: Setup CodeCommit Repository

1. Create a new repository in **CodeCommit** (e.g., `demo-apk`).  
2. Configure Git credentials:
   - Install AWS CLI on your Ubuntu VM:
     ```bash
     sudo apt update && sudo apt upgrade -y
     sudo curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
     sudo apt install unzip -y
     unzip awscliv2.zip
     sudo ./aws/install
     ```
   - Configure credentials:
     ```bash
     aws configure
     aws sts get-caller-identity --region ap-south-1
     git config --global credential.helper '!aws codecommit credential-helper $@'
     git config --global credential.UseHttpPath true
     ```
3. Clone the repo:
   ```bash
   git clone https://git-codecommit.ap-south-1.amazonaws.com/v1/repos/demo-apk
   cd demo-apk
   ```
4. Add sample file:
   ```bash
   echo "<h1>My Demo App</h1>" > index.html
   git add .
   git commit -m "Initial commit"
   git push origin master
   ```

---

## 🛠️ Step 2: Setup CodeBuild Project

1. Create a new **CodeBuild project** (`demo-apk-build`).  
2. Source: CodeCommit → `demo-apk` → branch `master`.  
3. Environment: Ubuntu, managed image, standard runtime.  
4. Service role: let CodeBuild create a new role.  
5. Add `buildspec.yml` in repo root:

```yaml
version: 0.2

phases:
  install:
    commands:
      - echo Installing NGINX
      - sudo apt update -y
      - sudo apt install nginx -y

  build:
    commands:
      - echo Build started on `date`
      - cp -r * /var/www/html/

artifacts:
  files:
    - '**/*'
  base-directory: /var/www/html
  discard-paths: no
```

6. Configure **Artifacts**:
   - Type: ZIP  
   - S3 bucket: `ci-cd-demo-bucket-<account>-ap-south-1`  
   - Path: `code_output_folder/`  
   - Name: `artifacts.zip`

---

## 🛠️ Step 3: Setup CodeDeploy Application

1. Create a **CodeDeploy application** (`demo-apk`).  
   - Compute platform: EC2/on‑premises.  
2. Create a **Deployment group** (`demo-apk-deployment-group`).  
   - Environment: EC2 instances (tagged with `Name=demovm`).  
   - Deployment type: In‑place.  
   - Service role: attach `AWSCodeDeployRole`.  
3. Install CodeDeploy agent on EC2:
   ```bash
   sudo apt-get update
   sudo apt-get install ruby wget -y
   cd /home/ubuntu
   wget https://aws-codedeploy-ap-south-1.s3.ap-south-1.amazonaws.com/latest/install
   chmod +x ./install
   sudo ./install auto
   sudo service codedeploy-agent start
   sudo service codedeploy-agent status
   ```

---

## 🛠️ Step 4: Add AppSpec File and Scripts

In your repo root, create `appspec.yml`:

```yaml
version: 0.0
os: linux
files:
  - source: /
    destination: /var/www/html

hooks:
  AfterInstall:
    - location: scripts/install_nginx.sh
      timeout: 300
      runas: root
  ApplicationStart:
    - location: scripts/start_nginx.sh
      timeout: 300
      runas: root
```

Create `scripts/` folder with:

**install_nginx.sh**
```bash
#!/bin/bash
sudo apt update -y
sudo apt install nginx -y
```

**start_nginx.sh**
```bash
#!/bin/bash
sudo systemctl start nginx
```

Push changes:
```bash
git add .
git commit -m "Added appspec and scripts"
git push origin master
```

---

## 🛠️ Step 5: Setup CodePipeline

1. Create new pipeline (`demo-apk-pipeline`).  
2. **Category** → Deployment  
3. **Template** → Build custom pipeline  
4. Stages:
   - **Source**: CodeCommit (`demo-apk`, branch `master`).  
     - Enable **EventBridge rule** → triggers pipeline automatically on commits.  
   - **Build**: CodeBuild (`demo-apk-build`).  
     - Input artifact: `SourceArtifact`.  
     - Output artifact: `BuildArtifact`.  
   - **Deploy**: CodeDeploy (`demo-apk`, `demo-apk-deployment-group`).  
     - Input artifact: `BuildArtifact`.  

---

## 🛠️ Step 6: IAM Roles

- **CodeBuild role** → `AWSCodeCommitReadOnly`, `AmazonS3FullAccess`.  
- **CodeDeploy service role** → `AWSCodeDeployRole`.  
- **EC2 instance role** → `AmazonEC2RoleforAWSCodeDeploy`, `AmazonS3ReadOnlyAccess`.

---

## ✅ Testing the Pipeline

1. Edit `index.html`:
   ```html
   <h1>My Demo App</h1>
   <p>Updated content!</p>
   ```
2. Push changes:
   ```bash
   git add .
   git commit -m "Updated HTML"
   git push origin master
   ```
3. Pipeline runs automatically (triggered by EventBridge).  
4. CodeDeploy deploys new artifact to EC2.  
5. Visit EC2 public IP → updated website is live.

---

## 🎯 Key Notes
- **Enable EventBridge rule** → instant pipeline trigger on commits.  
- **Artifacts must be ZIP** → include `appspec.yml` at root.  
- **CodeDeploy agent** must be running on EC2.  
- **IAM roles** must have correct policies for S3, CodeDeploy, and CodeCommit.

---

# 🚀 Conclusion
This README provides a complete CI/CD pipeline setup using AWS CodeCommit, CodeBuild, CodeDeploy, and EC2. With EventBridge enabled, every commit automatically updates your website.
```

---
