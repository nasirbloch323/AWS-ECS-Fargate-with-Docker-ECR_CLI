# Deploying a Three-Tier App (React + Node.js + MongoDB) on AWS ECS (Fargate) with Docker & ECR

This guide walks through containerizing a full-stack application (React frontend, Node.js backend, MongoDB database) and deploying it on **AWS ECS using the Fargate launch type**, with images stored in **Amazon ECR**.

---

## 📐 Architecture

| Component | Role | Port | Source |
|---|---|---|---|
| Frontend (React) | Public UI | 3000 | Custom image → ECR |
| Backend (Node.js) | API server | 5000 | Custom image → ECR |
| MongoDB | Database | 27017 | `mongo:latest` from Docker Hub |

All three containers run inside **one ECS Fargate Task**, communicating internally using container names (`backend`, `mongodb`).

---

## 🧰 Prerequisites

| Tool | Requirement |
|---|---|
| AWS Account | Access to ECS, ECR, IAM, VPC |
| Docker | Installed & running |
| AWS CLI | Installed & configured |
| VPC | At least 1 public subnet |
| IAM Role | `ecsTaskExecutionRole` with ECR + CloudWatch permissions |

---

## 🔐 Permissions You Need to Set Up

Before starting, make sure these permissions are in place — most deployment failures happen here, not in the actual ECS config.

1. **IAM User / Access Keys (for your local machine or EC2)**
   You need an IAM user (or role) with permissions to push images to ECR and manage ECS/IAM resources. At minimum, attach:
   - `AmazonEC2ContainerRegistryFullAccess`
   - `AmazonECS_FullAccess`
   - `IAMReadOnlyAccess` (to attach/verify roles)

   Generate credentials:
   - AWS Console → IAM → Users → your username → **Security credentials** → **Create access key**
   - Copy the **Access Key ID** and **Secret Access Key**

2. **EC2 Instance Role (if you're building/pushing images from an EC2 machine)**
   If you're using an EC2 instance to build Docker images and push to ECR, attach an **IAM Instance Profile** to that EC2 instance with the same ECR push permissions above. This avoids hardcoding access keys on the instance.

3. **ECS Task Execution Role (`ecsTaskExecutionRole`)**
   This is what ECS itself uses to:
   - Pull your container images from ECR
   - Send logs to CloudWatch

   Attach the AWS managed policy: `AmazonECSTaskExecutionRolePolicy`
   If this role doesn't already exist in your account, create it in IAM → Roles → **Elastic Container Service Task** as the trusted entity.

4. **Execute Permission on Installer/Scripts**
   Some downloaded files (like the AWS CLI installer) need execute permission before you can run them:
   ```bash
   chmod +x ./aws/install
   ```
   If you write your own deploy/build scripts (e.g. `deploy.sh`), always give them execute permission first:
   ```bash
   chmod +x deploy.sh
   ./deploy.sh
   ```

5. **Security Group Permissions**
   Your ECS Service's security group must allow inbound traffic on the ports your containers expose (3000, 5000, 27017) — covered in detail in Step 9 below.

---

## 🧹 Step 0: Clean Up Docker Disk Usage (Optional)

```bash
sudo docker system prune -af
sudo docker volume prune -f
```

Check current usage:
```bash
docker system df
```

---

## 🛠 Step 1: Install AWS CLI

```bash
# Update packages
sudo apt update

# Install dependencies
sudo apt install unzip curl -y

# Download AWS CLI v2 bundle
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

# Unzip the installer
unzip awscliv2.zip

# Give execute permission and run the installer
chmod +x ./aws/install
sudo ./aws/install

# Verify installation
aws --version
```

---

## 🔑 Step 2: Configure AWS CLI

```bash
aws configure
```

You'll be prompted for:
- AWS Access Key ID
- AWS Secret Access Key
- Default region (e.g. `us-east-1`)
- Default output format (e.g. `json`)

---

## ✅ Step 3: Clone the Application


## 🐳 Step 4: Build Docker Images Locally (or on EC2)

```bash
# Backend
docker build -t three-tier-backend -f backend/Dockerfile .

# Frontend
docker build -t three-tier-frontend -f frontend/Dockerfile .
```

---

## 📤 Step 5: Push Docker Images to Amazon ECR

**5.1 — Create ECR Repositories**
```bash
aws ecr create-repository --repository-name three-tier-backend
aws ecr create-repository --repository-name three-tier-frontend
```

**5.2 — Authenticate Docker to ECR**
```bash
aws ecr get-login-password --region us-east-1 | \
docker login --username AWS --password-stdin <your-account-id>.dkr.ecr.us-east-1.amazonaws.com
```

**5.3 — Tag & Push Images**
```bash
# Backend
docker tag three-tier-backend:latest <your-account-id>.dkr.ecr.us-east-1.amazonaws.com/three-tier-backend
docker push <your-account-id>.dkr.ecr.us-east-1.amazonaws.com/three-tier-backend

# Frontend
docker tag three-tier-frontend:latest <your-account-id>.dkr.ecr.us-east-1.amazonaws.com/three-tier-frontend
docker push <your-account-id>.dkr.ecr.us-east-1.amazonaws.com/three-tier-frontend
```

---

## 🧱 Step 6: Create the ECS Cluster

- AWS Console → **ECS** → **Clusters** → **Create Cluster**
- Type: **Fargate (Networking Only)**
- Name: `three-tier-cluster`

---

## 📋 Step 7: Create the Task Definition

| Setting | Value |
|---|---|
| Launch type | Fargate |
| Task Role | `ecsTaskExecutionRole` |
| Network Mode | `awsvpc` |
| Task Size | 1 vCPU, 2–4 GB Memory |

**Add 3 containers to the same Task Definition:**

**🟡 MongoDB**
| Field | Value |
|---|---|
| Name | `mongodb` |
| Image | `mongo:latest` |
| Port | 27017 |
| Essential | ✅ Yes |

**🔵 Backend**
| Field | Value |
|---|---|
| Name | `backend` |
| Image | `<your-account-id>.dkr.ecr.us-east-1.amazonaws.com/three-tier-backend` |
| Port | 5000 |
| Essential | ✅ Yes |
| Env Var | `MONGO_URL = mongodb://mongodb:27017/notes` |

> ✅ Use the container name `mongodb` for internal networking — no IP addresses needed.

**🟢 Frontend**
| Field | Value |
|---|---|
| Name | `frontend` |
| Image | `<your-account-id>.dkr.ecr.us-east-1.amazonaws.com/three-tier-frontend` |
| Port | 3000 |
| Essential | ✅ Yes |

> ✅ Set `REACT_APP_API_URL=http://backend:5000` in the frontend's `.env` before building the image.

---

## 🚀 Step 8: Create the ECS Service

| Setting | Value |
|---|---|
| Cluster | `three-tier-cluster` |
| Task Definition | the one created above |
| Launch type | Fargate |
| Number of tasks | 1 |
| Platform version | LATEST |

The **ECS Service** keeps your desired task count running, restarts failed tasks, and manages rolling deployments — you don't have to do this manually.

---

## 🌐 Step 9: Configure Networking & Security Group

| Field | Value |
|---|---|
| VPC | Your default/custom VPC |
| Subnets | Public subnet |
| Assign public IP | ✅ Enabled |
| Security Group | Open ports 3000, 5000, 27017 |

**Inbound Rules:**
| Type | Port | Source |
|---|---|---|
| Custom TCP | 3000 | 0.0.0.0/0 |
| Custom TCP | 5000 | 0.0.0.0/0 |
| Custom TCP | 27017 | 0.0.0.0/0 (optional, restrict in production) |

> ✅ In production, restrict the MongoDB port (27017) to internal VPC traffic only — don't expose it publicly.

---

## 🧪 Step 10: Test the Application

1. Go to **ECS → Clusters → Services → Running Task**
2. Copy the **public IP**
3. Open it in a browser:
   ```
   http://<public-ip>
   ```

Expected behavior:
- ✅ React frontend loads
- ✅ React calls the Node backend internally (`http://backend:5000`)
- ✅ Node connects to MongoDB (`mongodb://mongodb:27017/notes`)

**Sample CloudWatch logs:**
```
Node started on port 5000
MongoDB connected
React app served on port 3000
```

---

## ✅ Best Practices Followed

| Practice | Applied | Why |
|---|---|---|
| Slim/base Docker images | ✅ | Smaller, more secure images |
| Non-root container user | ✅ | Reduces attack surface |
| No hardcoded secrets | ✅ | Use environment variables |
| Private ECR repos | ✅ | Secure image storage |
| Internal networking | ✅ | Containers talk via container name |
| Expose only needed ports | ✅ | Controlled via Security Groups |

---

## 🏁 Summary

By following this guide, you will have:
- Built secure Docker images for the Node.js and React apps
- Pushed both images to private ECR repositories
- Created an ECS **Cluster**, a **Task Definition** (with all 3 containers), and an ECS **Service** to run them on **Fargate**
- Exposed the app publicly while keeping internal container communication secure

This mirrors a real-world microservices deployment pipeline used in production SaaS environments.
