# WireMock Service Walkthrough

## Overview
This service mocks third-party APIs (Digio, Hunter, Cashfree, NSDL) to allow QA testing without hitting production endpoints.

## Prerequisites
- Docker and Docker Compose installed.

## How to Run
1. Navigate to the `wiremock-service` directory:
   ```bash
   cd wiremock-service
   ```
2. Start the service (if not already running):
   ```bash
   docker-compose up -d
   ```
3. The mock service will be available at `http://localhost:8080`.

## Verified Endpoints & Responses

### Digio KYC
**Request:**
```bash
curl -X POST http://localhost:8080/digio/kyc
```
**Response:**
```json
{"status":"success","message":"KYC Verified Successfully","reference_id":"MOCK_DIGIO_12345"}
```

### Hunter Fraud Check
**Request:**
```bash
curl -X POST http://localhost:8080/hunter/check
```
**Response:**
```json
{"status":"approved","score":0,"message":"No Fraud Detected","transaction_id":"MOCK_HUNTER_67890"}
```

### Cashfree Disbursement
**Request:**
```bash
curl -X POST http://localhost:8080/cashfree/disburse
```
**Response:**
```json
{"status":"SUCCESS","message":"Disbursement Successful","referenceId":"MOCK_CASHFREE_54321","amount":50000}
```

### NSDL PAN Check
**Request:**
```bash
curl -X POST http://localhost:8080/nsdl/pan
```
**Response:**
```json
{"status":"E","message":"Existing and Valid","name":"MOCK USER NAME","pan":"ABCDE1234F"}
```

## Admin Interface
Access the WireMock admin interface at:

## AWS Deployment Guide

This guide outlines how to deploy the `wiremock-service` to AWS using **Amazon ECS (Elastic Container Service)** with **AWS Fargate**.

### Architecture Overview
- **Service:** Docker container running WireMock.
- **Compute:** AWS Fargate (Serverless, no EC2 management).
- **Network:** Private Subnet (Not accessible from the public internet).
- **Access:** Internal Application Load Balancer (ALB) or Service Discovery.

### Deployment Steps

#### 1. Containerize & Push Image (ECR)
First, you need to build your Docker image and push it to Amazon Elastic Container Registry (ECR).

1.  **Create an ECR Repository:**
    ```bash
    aws ecr create-repository --repository-name wiremock-service
    ```
2.  **Authenticate Docker with ECR:**
    ```bash
    aws ecr get-login-password --region <your-region> | docker login --username AWS --password-stdin <your-account-id>.dkr.ecr.<your-region>.amazonaws.com
    ```
3.  **Build and Tag the Image:**
    ```bash
    # From the wiremock-service directory
    docker build -t wiremock-service .
    docker tag wiremock-service:latest <your-account-id>.dkr.ecr.<your-region>.amazonaws.com/wiremock-service:latest
    ```
    *(Note: You'll need to create a `Dockerfile` if you haven't already, or just map volumes in the task definition. Since we have local mappings, baking them into the image is better for deployment.)*
4.  **Push to ECR:**
    ```bash
    docker push <your-account-id>.dkr.ecr.<your-region>.amazonaws.com/wiremock-service:latest
    ```

#### 2. Create ECS Cluster
1.  Go to the **ECS Console**.
2.  Create a new Cluster (Networking only - Fargate).
3.  Name it `qa-cluster`.

#### 3. Create Task Definition
1.  Create a new **Task Definition** (Fargate launch type).
2.  **Name:** `wiremock-task`
3.  **Task Role:** Access to ECR and CloudWatch Logs.
4.  **Container Definitions:**
    - **Image:** URI from ECR (step 1.4).
    - **Port Mappings:** 8080 (TCP).
    - **Environment Variables:** Any needed config.

#### 4. Create Service
1.  Create a **Service** within your cluster.
2.  **Launch Type:** Fargate.
3.  **Task Definition:** `wiremock-task`.
4.  **Service Name:** `wiremock-service`.
5.  **Number of Tasks:** 1 (for mock, 1 is usually enough).
6.  **VPC & Subnets:** Choose your **Private Subnets** to ensure it's not internet-facing.
7.  **Security Group:** Allow Inbound traffic on port 8080 ONLY from your Application's Security Group.

#### 5. Accessing the Service
To let your main application talk to this mock service:

**Option A: Internal Load Balancer (Recommended for HA)**
- Create an **Internal Application Load Balancer**.
- Create a Target Group (IP type) pointing to port 8080.
- Point the ECS Service to this Target Group.
- Your app connects via `http://<internal-alb-dns-name>:80`.

**Option B: Service Discovery (Simpler)**
- Enable **Service Connect** or **Service Discovery** when creating the ECS Service.
- Your app connects via `http://wiremock-service.qa-cluster.local:8080`.

### Dockerfile for Deployment
To make the deployment smoother, create a `Dockerfile` in `wiremock-service` to include your mappings:

```dockerfile
FROM wiremock/wiremock:latest

# Copy local mappings and files into the image
COPY mappings /home/wiremock/mappings
COPY __files /home/wiremock/__files

EXPOSE 8080
```

### Option 2: EC2 with Docker Compose (Simpler Alternative)

If you prefer to manage a server directly or want an experience closer to your local development environment, you can run the service on an EC2 instance using Docker Compose.

#### 1. Launch EC2 Instance
1.  Launch an **Amazon Linux 2023** or **Ubuntu** instance.
2.  Place it in a **Private Subnet** (with NAT Gateway access for installing packages) or Public Subnet (restricted by Security Group).
3.  **Security Group:** Allow Inbound port 8080 from your application's Security Group / VPN.

#### 2. Install Docker & Git
SSH into your instance and run:

**For Amazon Linux 2023:**
```bash
sudo yum update -y
sudo yum install -y docker git
sudo service docker start
sudo usermod -a -G docker ec2-user
# Logout and login again to pick up group changes
```

**Install Docker Compose:**
```bash
mkdir -p ~/.docker/cli-plugins/
curl -SL https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64 -o ~/.docker/cli-plugins/docker-compose
chmod +x ~/.docker/cli-plugins/docker-compose
```

#### 3. Deploy
1.  **Clone the Repository:**
    ```bash
    git clone https://github.com/Vsury-ui/wiremock.git
    cd wiremock
    ```
2.  **Start the Service:**
    ```bash
    docker compose up -d
    ```
3.  **Verify:**
    ```bash
    curl http://localhost:8080/__admin
    ```

#### 4. Connect
- Connect from your main application using the **Private IP** of the EC2 instance: `http://<private-ip>:8080`.
