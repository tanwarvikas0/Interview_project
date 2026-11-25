# AWS ECS Fargate Multi-Tier Application Deployment



---


## Step 1: Networking (VPC & Security Groups)

### 1.1 Create VPC & Subnets
Create a VPC named `ritual-roast-vpc` (`10.0.0.0/16`) across **2 Availability Zones** (`us-east-1a`, `us-east-1b`):

| Tier | Subnet Names | CIDR Blocks | Routing |
| :--- | :--- | :--- | :--- |
| **Public** | Public Subnet 1a / 1b | `10.0.1.0/24`, `10.0.2.0/24` | Internet Gateway (`rr-igw`) |
| **App Private** | App Subnet 1a / 1b | `10.0.10.0/24`, `10.0.11.0/24` | NAT Gateway (`rr-nat` in 1a) |
| **Data Private** | Data Subnet 1a / 1b | `10.0.20.0/24`, `10.0.21.0/24` | NAT Gateway (`rr-nat` in 1a) |

### 1.2 Create Security Groups
Create three security groups inside `ritual-roast-vpc`:

| Security Group | Purpose | Inbound Protocol / Port | Allowed Source |
| :--- | :--- | :--- | :--- |
| **`rr-alb-sg`** | Application Load Balancer | HTTP (`80`) | `0.0.0.0/0` (Internet) |
| **`rr-app-sg`** | ECS Fargate Tasks | HTTP (`80`) | `rr-alb-sg` |
| **`rr-data-sg`** | RDS MySQL Database | MySQL/Aurora (`3306`) | `rr-app-sg` and `rr-data-sg` *(self-referencing for Secrets rotation Lambda)* |

---

## Step 2: Container Registry (Amazon ECR)

1. Navigate to **Amazon ECR → Repositories → Create repository**.
2. Set visibility to **Private** and name it `ritual-roast`.
3. Copy and save the **Repository URI**:
   ```text
   <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/ritual-roast
   ```

---

## Step 3: Build & Push Docker Image

### 3.1 Create Build Server IAM Role
1. Create an IAM role named `iam-role-grant-ec2-ssm-ecr-access` with **EC2** as the trusted entity.
2. Attach the following AWS managed policies:
   * `AmazonSSMManagedInstanceCore`
   * `EC2InstanceProfileForImageBuilderECRContainerBuilds`

### 3.2 Launch Build Instance & Push Image
1. Launch an **Amazon Linux 2023 (`t2.micro`)** EC2 instance inside a private app subnet with the IAM role above attached (no SSH key pair required).
2. Connect via **AWS Systems Manager (Session Manager)** and execute:

```bash
sudo su - ec2-user
sudo yum update -y && sudo yum install -y docker unzip
sudo service docker start
sudo usermod -a -G docker ec2-user
# Re-authenticate session to apply docker group permissions
sudo su - ec2-user

# Download and extract source code
wget <YOUR_SOURCE_CODE_ZIP_URL>
unzip <ARCHIVE_NAME>.zip && cd <PROJECT_DIRECTORY>

# Build Docker image
docker build -t <ECR_REPOSITORY_URI> .

# Authenticate Docker with Amazon ECR and push
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com
docker push <ECR_REPOSITORY_URI>:latest
```

---

## Step 4: Database Setup (Amazon RDS MySQL)

1. **Create DB Subnet Group:**
   * Name: `ritual-roast-db-subnet-group`
   * VPC: `ritual-roast-vpc`
   * Subnets: Select both **Data Private Subnets** (`10.0.20.0/24` and `10.0.21.0/24`).
2. **Create RDS Database:**
   * **Engine:** MySQL | **Template:** Free Tier (`db.t3.micro`)
   * **DB Instance Identifier:** `ritual-roast-db`
   * **Master Username:** `admin` *(set a strong password)*
   * **VPC:** `ritual-roast-vpc` | **Public Access:** `No` | **Security Group:** `rr-data-sg`
   * **Initial Database Name:** `ritualroastdb` *(must match your application's DB config)*
3. Once available, note the **RDS Endpoint**.

---

## Step 5: Credentials Management (AWS Secrets Manager)

1. Verify `rr-data-sg` allows inbound **Port 3306** from itself (`rr-data-sg`) so the rotation Lambda can connect to RDS.
2. Navigate to **AWS Secrets Manager → Store a new secret**.
3. Select **Credentials for Amazon RDS database**, enter the `admin` username and password, and select `ritual-roast-db`.
4. Name the secret `ritual-roast-db-secret`.
5. Enable **Automatic rotation** (e.g., every `7` days) and let AWS create a new rotation Lambda function.
6. Copy the **Secret ARN** and **Secret Name**.

---

## Step 6: Load Balancing (Target Group & ALB)

1. **Create Target Group (`ritual-roast-tg`):**
   * **Target Type:** `IP addresses` *(required for Fargate)*
   * **Protocol / Port:** `HTTP : 80` | **VPC:** `ritual-roast-vpc`
   * **Health Check Path:** `/health.html`
2. **Create Application Load Balancer (`ritual-roast-alb`):**
   * **Scheme:** Internet-facing | **IP Address Type:** IPv4
   * **Network Mapping:** `ritual-roast-vpc` → Select both **Public Subnets** (`1a` and `1b`)
   * **Security Group:** `rr-alb-sg`
   * **Listener:** Forward `HTTP : 80` to `ritual-roast-tg`

---

## Step 7: ECS IAM Permissions

1. **Create Custom Policy (`ritual-roast-allow-read-db-secret-policy`):**
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": "secretsmanager:GetSecretValue",
         "Resource": "<YOUR_SECRET_ARN>"
       }
     ]
   }
   ```
2. **Create IAM Role (`ritual-roast-ecs-task-role`):**
   * **Trusted Entity:** `Elastic Container Service Task`
   * **Attached Policies:**
     * `AmazonECSTaskExecutionRolePolicy`
     * `ritual-roast-allow-read-db-secret-policy`

---

## Step 8: ECS Cluster, Task Definition & Service

### 8.1 Create ECS Cluster
* **Cluster Name:** `ritual-roast-ecs-cluster`
* **Infrastructure:** AWS Fargate (serverless)

### 8.2 Create Task Definition
* **Family:** `ritual-roast-task-def`
* **Launch Type:** `AWS Fargate` | **OS/Architecture:** `Linux/X86_64`
* **Task Size:** `0.5 vCPU` | `1 GB` Memory
* **Task Role & Execution Role:** `ritual-roast-ecs-task-role`
* **Container Configuration:**
  * **Name:** `ritual-roast`
  * **Image URI:** `<ECR_REPOSITORY_URI>:latest`
  * **Port Mapping:** Container Port `80` (`TCP` / `HTTP`)
  * **Environment Variables:**

| Key | Value |
| :--- | :--- |
| `DB_SERVER` | `<YOUR_RDS_ENDPOINT>` |
| `DB_DATABASE` | `ritualroastdb` |
| `SECRET_NAME` | `ritual-roast-db-secret` |
| `AWS_REGION` | `us-east-1` |

### 8.3 Create ECS Service
* **Cluster:** `ritual-roast-ecs-cluster`
* **Compute:** Fargate (`LATEST`) | **Service Name:** `ritual-roast-service` | **Desired Tasks:** `1`
* **Networking:**
  * **VPC:** `ritual-roast-vpc`
  * **Subnets:** Select **only App Private Subnets** (`10.0.10.0/24`, `10.0.11.0/24`)
  * **Security Group:** `rr-app-sg`
  * **Public IP:** `Turned OFF`
* **Load Balancing:**
  * Use existing ALB `ritual-roast-alb` (`80:HTTP`) and Target Group `ritual-roast-tg`.
* **Auto Scaling (Optional):**
  * Min tasks: `1` | Max tasks: `4` | Target tracking on `ECSServiceAverageCPUUtilization` at `70%`.

---

## Step 9: Verification & Cleanup

### Verification
1. Confirm the ECS service task status reaches **Running**.
2. Confirm the Target Group (`ritual-roast-tg`) registered target reports **Healthy**.
3. Open the **ALB DNS Name** in a browser to verify the application loads and connects to RDS.

### Resource Cleanup Order
To avoid ongoing AWS charges after testing, delete resources in the following order:
1. ECS Service & ECS Cluster
2. Application Load Balancer & Target Group
3. RDS MySQL Database Instance
4. Secrets Manager Secret & Rotation Lambda Function
5. EC2 Build Instance & ECR Repository
6. NAT Gateway & Release Elastic IP
7. VPC (`ritual-roast-vpc`)
