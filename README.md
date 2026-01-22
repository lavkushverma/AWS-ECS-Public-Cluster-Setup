# AWS-ECS-Public-Cluster-Setup
AWS-ECS-Public-Cluster-Setup-Guide
This is a classic "Full Stack" DevOps project. We will build a containerized website (ECS) that loads images from storage (S3), pulled from a private registry (ECR), and served to the world via a Load Balancer (ALB).

**The Architecture:**
`User` -> `ALB` -> `ECS Fargate (Nginx)` -> `Reads Docker Image from ECR` & `Displays Image from S3`

---

### Phase 1: S3 (The Static Assets)
We will upload an image to S3 that our website will display.

1.  **Create a Bucket:**
    ```bash
    # Use a unique name
    aws s3 mb s3://my-lab-assets-9988 --region us-east-1
    ```

2.  **Download a sample image & Upload:**
    ```bash
    # Download a dummy image
    curl https://via.placeholder.com/150 -o logo.png

    # Upload to S3 and make it public (For this demo)
    aws s3 cp logo.png s3://my-lab-assets-9988/ --acl public-read
    ```
    *Note: If `--acl public-read` fails, you need to go to S3 Console -> Permissions -> Turn OFF "Block all public access" and try again.*

3.  **Get the URL:**
    Your URL is: `https://my-lab-assets-9988.s3.amazonaws.com/logo.png`
    *(Keep this URL handy).*

---

### Phase 2: The Application (Docker & ECR)
We will create a simple HTML file that references the S3 image, verify it locally, and push it to AWS ECR.

**1. Create `index.html`**
```html
<!DOCTYPE html>
<html>
<head><title>ECS Demo</title></head>
<body style="text-align:center; padding-top:50px;">
    <h1>Hello from AWS ECS + ALB!</h1>
    <p>This image is served from S3:</p>
    <!-- REPLACE THE URL BELOW WITH YOUR S3 URL -->
    <img src="https://my-lab-assets-9988.s3.amazonaws.com/logo.png" alt="S3 Image">
</body>
</html>
```

**2. Create `Dockerfile`**
```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
```

**3. Build and Push to ECR**
```bash
# 1. Create Repo
aws ecr create-repository --repository-name ecs-alb-demo --region us-east-1

# 2. Login
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <YOUR-ACCOUNT-ID>.dkr.ecr.us-east-1.amazonaws.com

# 3. Build
docker build -t ecs-alb-demo .

# 4. Tag
docker tag ecs-alb-demo:latest <YOUR-ACCOUNT-ID>.dkr.ecr.us-east-1.amazonaws.com/ecs-alb-demo:latest

# 5. Push
docker push <YOUR-ACCOUNT-ID>.dkr.ecr.us-east-1.amazonaws.com/ecs-alb-demo:latest
```

---

### Phase 3: The Security Groups (Firewalls)
Before creating the heavy infrastructure, let's set up the security rules.

1.  Go to **EC2 Console** -> **Security Groups**.
2.  **Create SG 1 (For ALB):**
    *   Name: `ALB-SG`
    *   Inbound: Allow **HTTP (80)** from **Anywhere (0.0.0.0/0)**.
3.  **Create SG 2 (For ECS):**
    *   Name: `ECS-SG`
    *   Inbound: Allow **HTTP (80)** from **Source: ALB-SG**.
    *   *(This ensures nobody can bypass the Load Balancer).*

---

### Phase 4: The Load Balancer (ALB)
1.  Go to **EC2 Console** -> **Load Balancers** -> **Create Load Balancer**.
2.  Select **Application Load Balancer**.
3.  **Name:** `my-demo-alb`.
4.  **Scheme:** Internet-facing.
5.  **Network:** Select your default VPC and check **all subnets**.
6.  **Security Group:** Select `ALB-SG`.
7.  **Listeners:** Protocol HTTP, Port 80.
    *   Click **Create target group**.
    *   **Type:** IP addresses (Important for Fargate!).
    *   **Name:** `ecs-target-group`.
    *   Click **Next** -> Click **Create target group** (Don't register targets yet).
8.  Go back to ALB tab, hit refresh, select `ecs-target-group`, and click **Create load balancer**.

---

### Phase 5: The ECS Cluster & Service
Now we wire everything together.

1.  **Go to ECS Console.**
2.  **Create Cluster:**
    *   Name: `web-cluster`.
    *   Infrastructure: **Fargate**.
    *   Click **Create**.

3.  **Create Task Definition:**
    *   Name: `web-task`.
    *   Launch Type: **AWS Fargate**.
    *   OS: Linux.
    *   Memory: .5 GB, CPU: .25 vCPU.
    *   **Container Details:**
        *   Name: `web-container`
        *   Image URI: *(Paste your ECR URI from Phase 2)*
        *   Port Mappings: 80.
    *   Click **Create**.

4.  **Create Service (The final step):**
    *   Go inside `web-cluster`.
    *   Click **Create** (under Services).
    *   **Launch type:** Fargate.
    *   **Task Definition:** `web-task`.
    *   **Service Name:** `web-service`.
    *   **Desired tasks:** 2 (We want 2 copies running).
    *   **Networking:**
        *   VPC: Default.
        *   Subnets: Select all.
        *   Security Group: **Edit** -> Select `ECS-SG`.
        *   **Public IP:** ENABLED (Required for Fargate in public subnets to pull images).
    *   **Load Balancing:**
        *   Type: Application Load Balancer.
        *   Container: `web-container`.
        *   Click **Add to load balancer**.
        *   **Target Group:** Select `ecs-target-group`.
    *   Click **Create**.

---

### Phase 6: Test It!
1.  Go to **EC2 Console** -> **Load Balancers**.
2.  Copy the **DNS name** of your ALB (e.g., `my-demo-alb-123.us-east-1.elb.amazonaws.com`).
3.  Paste it into your browser.

**Success Criteria:**
1.  You see "Hello from AWS ECS + ALB!".
2.  You see the logo displayed (which proves S3 is working).
3.  If you refresh, you are technically hitting different containers (managed by ALB).

---

### Phase 7: Cleanup (Important)
To avoid costs:
1.  **ECS:** Delete Service, then Delete Cluster.
2.  **EC2:** Delete Load Balancer, then Delete Target Group.
3.  **ECR:** Delete Repository.
4.  **S3:** Delete Bucket.
