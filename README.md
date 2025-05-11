
# Cloud Project: Full-Stack Application Deployment on AWS

This project demonstrates the deployment of a full-stack web application on AWS, following a production-like, secure architecture. It includes a frontend, a backend API, a relational database, and file storage, all leveraging various AWS services.

## Architecture Diagram

![AWS Deployment Architecture](https://raw.githubusercontent.com/Mattique20/cloud-project-1/main/Untitled%20design%20(4).png)


## Project Functional Requirements

The application fulfills the following functional criteria:
1.  **User Authentication:** Secure user login and registration.
2.  **CRUD Operations:** Create, Read, Update, and Delete operations on at least one primary entity (e.g., Posts, Products, Tasks).
3.  **Database Operations:** User management and entity persistence using Amazon RDS (PostgreSQL/MySQL).
4.  **File and Image Upload Support:** Upload and retrieval of media assets using an AWS S3 Bucket.

## AWS Cloud Infrastructure Requirements

The system is deployed on AWS cloud services with a focus on security and scalability:
1.  **Frontend (React/Angular/Vue/etc.):**
    *   Deployed on AWS Elastic Beanstalk.
2.  **Backend (Node.js/Flask/Django/etc.):**
    *   Deployed on Amazon EC2 using a Docker container.
    *   EC2 is hosted inside a VPC with proper Security Groups and IAM roles.
3.  **Database Layer:**
    *   Amazon RDS (PostgreSQL/MySQL) for persistent data storage.
4.  **File/Image Storage:**
    *   Amazon S3 for uploading and retrieving media assets.
    *   S3 bucket policies for private and public access separation.
5.  **Security:**
    *   Implemented IAM Roles and Policies for EC2, S3, and RDS access.
    *   Configured Security Groups for EC2 and RDS (opening only necessary ports).
    *   HTTPS access (using ACM with Elastic Beanstalk's Load Balancer).
    *   Adherence to the least-privilege access model across all components.
6.  **Bonus:**
    *   Configured Route 53 for custom domain setup.

## Technology Stack

*   **Frontend:** Frontend: React (with Vite), React Router for navigation, Axios for API calls, Tailwind CSS for styling, Framer Motion for animations
*   **Backend:** Node.js with Express.js and PostgreSQL with Sequelize ORM
*   **Containerization:** Docker
*   **Cloud Provider:** Amazon Web Services (AWS)
    *   **Compute:** Elastic Beanstalk (Frontend), EC2 (Backend)
    *   **Database:** RDS (PostgreSQL/MySQL)
    *   **Storage:** S3
    *   **Networking:** VPC, Application Load Balancer (ALB via Elastic Beanstalk, and optionally for backend), Internet Gateway, NAT Gateway
    *   **Security & Identity:** IAM, Security Groups, ACM (AWS Certificate Manager)
    *   **DNS:** Route 53 (Optional)

## Deployment Guide

This guide outlines the steps to deploy the application on AWS.

**Prerequisites:**
*   AWS Account with necessary permissions.
*   AWS CLI configured locally.
*   Docker installed locally (for building and pushing backend image).
*   Git installed locally.
*   Your application code (frontend and backend) from [https://github.com/Mattique20/cloud-project-1](https://github.com/Mattique20/cloud-project-1). Clone this repository.

```bash
git clone [https://github.com/Mattique20/M.git](https://github.com/Mattique20/MetroArt.git)
cd cloud-project-1
```

---

### Phase 1: Foundational Setup (VPC, IAM, S3)

1.  **Create a VPC:**
    *   Navigate to the AWS VPC console.
    *   Create a VPC (e.g., `my-project-vpc` with CIDR `10.0.0.0/16`).
    *   Create **two Public Subnets** in different Availability Zones (AZs) (e.g., `10.0.1.0/24`, `10.0.2.0/24`).
    *   Create **two Private Subnets** in different AZs (e.g., `10.0.3.0/24`, `10.0.4.0/24`).
    *   Create an **Internet Gateway (IGW)** and attach it to your VPC.
    *   Create a **NAT Gateway** in one of your public subnets (allocate an Elastic IP for it).
    *   **Configure Route Tables:**
        *   **Public Route Table:** Associate with public subnets. Add a route `0.0.0.0/0` to the IGW.
        *   **Private Route Table:** Associate with private subnets. Add a route `0.0.0.0/0` to the NAT Gateway.

2.  **Create IAM Roles and Policies:**
    *   Navigate to the AWS IAM console.
    *   **a. EC2 Instance Role (`EC2AppRole`):**
        *   Trusted entity: `AWS service` -> `EC2`.
        *   Permissions:
            *   Create an inline policy (`EC2S3AccessPolicy`) for S3 access (replace `YOUR_BUCKET_NAME`):
                ```json
                {
                    "Version": "2012-10-17",
                    "Statement": [
                        {
                            "Effect": "Allow",
                            "Action": [
                                "s3:GetObject",
                                "s3:PutObject",
                                "s3:DeleteObject",
                                "s3:ListBucket"
                            ],
                            "Resource": [
                                "arn:aws:s3:::YOUR_BUCKET_NAME",
                                "arn:aws:s3:::YOUR_BUCKET_NAME/*"
                            ]
                        }
                    ]
                }
                ```
            *   Attach AWS managed policy: `AmazonRDSDataFullAccess` (or a more granular custom policy for RDS).
        *   Name: `EC2AppRole`.
    *   **b. Elastic Beanstalk Service Role:** Usually `aws-elasticbeanstalk-service-role`. Ensure it exists and has `AWSElasticBeanstalkEnhancedHealth` and `AWSElasticBeanstalkService` policies.
    *   **c. Elastic Beanstalk Instance Profile Role:** Usually `aws-elasticbeanstalk-ec2-role`.

3.  **Create S3 Bucket:**
    *   Navigate to the S3 console.
    *   Create a bucket with a **globally unique name** (e.g., `yourname-cloud-project-media`). Choose your region.
    *   **Block Public Access settings:** Initially, you might uncheck "Block all public access" to set up a bucket policy, then re-evaluate.
    *   **Bucket Policy:** (Example: public read for `public/` prefix, secure otherwise - replace `YOUR_BUCKET_NAME`)
        ```json
        {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Sid": "PublicReadForPublicFolder",
                    "Effect": "Allow",
                    "Principal": "*",
                    "Action": "s3:GetObject",
                    "Resource": "arn:aws:s3:::YOUR_BUCKET_NAME/public/*"
                }
                // The EC2AppRole will handle uploads to other (private) prefixes
            ]
        }
        ```
    *   User uploads should ideally go to a non-public prefix, accessed via pre-signed URLs generated by the backend.

---

### Phase 2: Database Setup (RDS)

1.  **Create Security Groups (SGs):**
    *   Navigate to VPC console -> Security Groups.
    *   **a. `BackendEC2-SG`:** For the backend EC2 instance.
        *   Inbound: Allow TCP on your backend's port (e.g., 3000, 5000, 8000) from the Security Group of the **Backend ALB** (you'll create this later). Allow SSH (TCP/22) from your IP.
        *   Outbound: Allow all (default).
    *   **b. `RDS-SG`:** For the RDS instance.
        *   Inbound: Allow TCP on DB port (MySQL: 3306, PostgreSQL: 5432) from `BackendEC2-SG` (source will be the ID of `BackendEC2-SG`).
        *   Outbound: Allow all (default).

2.  **Launch RDS Instance:**
    *   Navigate to the RDS console.
    *   Create database:
        *   Engine: MySQL or PostgreSQL.
        *   Templates: "Free tier" or "Dev/Test".
        *   Settings: DB instance identifier, Master username, Master password (store securely!).
        *   Connectivity:
            *   VPC: Select `my-project-vpc`.
            *   DB Subnet Group: Create new, select your **private subnets**.
            *   Public access: **No**.
            *   VPC security group: Choose existing -> `RDS-SG`.
    *   Note the **Endpoint** and **Port** once available.

---

### Phase 3: Backend Deployment (EC2 + Docker)

1.  **Prepare Backend Application (from your `backend` directory):**
    *   Ensure a `Dockerfile` exists.
    *   Configure your application to read database credentials, S3 bucket name, etc., from environment variables.

2.  **Push Docker Image to ECR (Elastic Container Registry) - Recommended:**
    *   Create an ECR private repository (e.g., `my-backend-app`).
    *   Follow "View push commands" in the ECR console to build, tag, and push your image from the `backend` directory.
        ```bash
        # cd into your backend directory
        # aws ecr get-login-password --region YOUR_REGION | docker login ...
        # docker build -t my-backend-app .
        # docker tag my-backend-app:latest YOUR_AWS_ACCOUNT_ID.dkr.ecr.YOUR_REGION.amazonaws.com/my-backend-app:latest
        # docker push YOUR_AWS_ACCOUNT_ID.dkr.ecr.YOUR_REGION.amazonaws.com/my-backend-app:latest
        ```

3.  **Create Application Load Balancer (ALB) for Backend:**
    *   This ALB will make your backend (in a private subnet) accessible to the frontend.
    *   Navigate to EC2 -> Load Balancers -> Create Application Load Balancer.
    *   Name: `backend-api-alb`.
    *   Scheme: `Internet-facing`.
    *   Network mapping: Select your VPC and **public subnets**.
    *   Security Groups: Create a new SG (`Backend-ALB-SG`) allowing HTTP/80 and HTTPS/443 from `0.0.0.0/0`.
    *   Listeners:
        *   HTTP/80: Forward to a new target group.
            *   Target group: `backend-ec2-tg`, Instances, HTTP, Port (your backend app port, e.g., 8000), VPC. Define health checks.
        *   (Optional but Recommended) HTTPS/443: Use an ACM certificate. Forward to `backend-ec2-tg`.
    *   Note the DNS name of this ALB. This is your backend API endpoint.

4.  **Launch EC2 Instance for Backend:**
    *   Navigate to EC2 console -> Launch Instances.
    *   Name: `Backend-API-Server`.
    *   AMI: Amazon Linux 2 (or similar with Docker).
    *   Instance Type: `t2.micro` or `t3.micro`.
    *   Key pair: Select/create.
    *   Network settings:
        *   VPC: `my-project-vpc`.
        *   Subnet: Choose one of your **private subnets**.
        *   Auto-assign public IP: **Disable**.
        *   Firewall (security groups): Select existing -> `BackendEC2-SG`.
    *   Advanced details:
        *   IAM instance profile: Select `EC2AppRole`.
        *   User data (replace placeholders, use your ECR URI):
            ```bash
            #!/bin/bash
            yum update -y
            amazon-linux-extras install docker -y
            systemctl start docker
            systemctl enable docker
            usermod -a -G docker ec2-user

            # Login to ECR
            aws ecr get-login-password --region YOUR_REGION | docker login --username AWS --password-stdin YOUR_AWS_ACCOUNT_ID.dkr.ecr.YOUR_REGION.amazonaws.com

            # Run Docker container (replace placeholders)
            docker run -d -p YOUR_APP_PORT:YOUR_APP_PORT \
              -e DB_HOST="YOUR_RDS_ENDPOINT" \
              -e DB_PORT="YOUR_RDS_PORT" \
              -e DB_USER="YOUR_DB_USER" \
              -e DB_PASSWORD="YOUR_DB_PASSWORD" \
              -e DB_NAME="YOUR_DB_NAME_IF_ANY" \
              -e S3_BUCKET_NAME="YOUR_S3_BUCKET_NAME" \
              -e NODE_ENV="production" \
              # Add any other environment variables your app needs
              YOUR_ECR_IMAGE_URI:latest
            ```
    *   Launch. Once running, register this instance with the `backend-ec2-tg` target group created for the `backend-api-alb`.
    *   **Update `BackendEC2-SG`:** Ensure the inbound rule for your app's port allows traffic *only* from `Backend-ALB-SG`.

---

### Phase 4: Frontend Deployment (Elastic Beanstalk)

1.  **Prepare Frontend Application (from your `frontend` directory):**
    *   Configure your frontend to make API calls to the **DNS name of the `backend-api-alb`**. This is often done via an environment variable (e.g., `REACT_APP_API_URL=http://your-backend-api-alb-dns...`).
    *   Build your frontend application for production.
    *   Zip the *contents* of your build/distribution directory (e.g., `build/` for React, `dist/` for Angular/Vue).

2.  **Create Elastic Beanstalk Application:**
    *   Navigate to Elastic Beanstalk console.
    *   Create application (e.g., `my-frontend-app`).
    *   Create environment:
        *   Web server environment.
        *   Platform: Choose appropriate (e.g., Node.js for serving static React/Vue builds, or your specific frontend platform).
        *   Application code: Upload your ZIP file.
        *   Configure more options:
            *   **Network:**
                *   VPC: `my-project-vpc`.
                *   Load balancer subnets: Select your **public subnets**.
                *   Instance subnets: Select your **public subnets**.
            *   **Security:**
                *   Service role: `aws-elasticbeanstalk-service-role`.
                *   EC2 instance profile: `aws-elasticbeanstalk-ec2-role`.
            *   **Load Balancer:**
                *   Type: Application Load Balancer.
                *   Listener Port 80: Enabled.
                *   (HTTPS) Add Listener for Port 443: Protocol HTTPS, select/request an ACM certificate.
            *   **Software (Environment properties):** Add any environment variables your frontend application needs at runtime.
    *   Create environment. This will provision an ALB, EC2 instances, etc. Note the Beanstalk environment URL.

---

### Phase 5: DNS and Finalization (Bonus)

1.  **Configure Route 53 (Optional):**
    *   If you have a custom domain, go to Route 53 -> Hosted zones.
    *   Create records (Type `A`, Alias) to point your domain/subdomain to:
        *   The Elastic Beanstalk environment's Load Balancer (for frontend).
        *   The `backend-api-alb` (for backend API, if using a separate API subdomain like `api.yourdomain.com`).

2.  **Test Thoroughly:**
    *   Access your frontend via the Elastic Beanstalk URL or your custom domain.
    *   Test user authentication, CRUD operations, and file uploads.
    *   Check browser developer tools for any API call errors.
    *   Check CloudWatch Logs for backend and frontend instances if issues arise.

