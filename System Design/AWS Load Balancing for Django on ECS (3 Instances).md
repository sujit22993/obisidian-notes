
#System-design #Load-balancers #ECS #AWS #ECR #ALB #Deployments

### **1. High-Level Architecture**

```
graph TD
    User[User Browser] --> ALB[AWS Application Load Balancer]
    ALB --> ECS1[Django App - ECS Task 1]
    ALB --> ECS2[Django App - ECS Task 2]
    ALB --> ECS3[Django App - ECS Task 3]
    ECS1 --> RDS[PostgreSQL on AWS RDS]
    ECS2 --> RDS
    ECS3 --> RDS
    ECS1 --> S3[AWS S3 for static/media files]
    ECS2 --> S3
    ECS3 --> S3

```

---

### **2. Step-by-Step Setup**

#### **Step 1: Prepare Your Django App for ECS**

- **Dockerize Django**
    
    - Create a `Dockerfile` and `docker-compose.yml` (optional for local testing).
        
    - Ensure:
        
        - Static files are collected to `/static/`
            
        - Gunicorn or uWSGI is the WSGI server (not `runserver`)
            
- **Example minimal Dockerfile**:
```
    FROM python:3.11-slim
	WORKDIR /app
	COPY requirements.txt .
	RUN pip install -r requirements.txt
	COPY . .
	RUN python manage.py collectstatic --noinput
	CMD ["gunicorn", "myproject.wsgi:application", "--bind", "0.0.0.0:8000"]

```

---

#### **Step 2: Push Image to ECR (Elastic Container Registry)**

```
	aws ecr create-repository --repository-name my-django-app
	aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin <account-id>.dkr.ecr.ap-south-1.amazonaws.com
	docker build -t my-django-app .
	docker tag my-django-app:latest <account-id>.dkr.ecr.ap-south-1.amazonaws.com/my-django-app:latest
	docker push <account-id>.dkr.ecr.ap-south-1.amazonaws.com/my-django-app:latest

```
---

#### **Step 3: ECS Cluster Setup**

- Create **ECS Cluster** (Fargate or EC2 Launch Type — Fargate preferred for serverless scaling).
    
- Create **Task Definition**:
    
    - Use the ECR image.
        
    - Assign **CPU** and **Memory**.
        
    - Configure **Port Mapping**: Container port `8000`.
        

---

#### **Step 4: Create Application Load Balancer (ALB)**

- **Listener**: Port 80 (HTTP) or 443 (HTTPS with ACM Certificate).
    
- **Target Group**:
    
    - Type: `ip` (for Fargate) or `instance` (for EC2 launch type).
        
    - Health Check Path: `/health/` (implement in Django to return HTTP 200).
        
- **Register ECS Service** with ALB:
    
    - Desired Tasks: `3` (will create 3 instances of your app).
        
    - Each new task registers automatically with the target group.
        

---

#### **Step 5: Networking**

- Place ECS tasks in **private subnets**.
    
- ALB in **public subnets**.
    
- Use **Security Groups**:
    
    - ALB SG → Allows inbound HTTP/HTTPS from `0.0.0.0/0`.
        
    - ECS SG → Allows inbound only from ALB SG on port 8000.
        

---

#### **Step 6: Scaling**

- **Manual Scaling**: Change desired count in ECS Service from `3` to N.
    
- **Auto Scaling**:
    
    - Create ECS Service Auto Scaling policy.
        
    - Metric: CPUUtilization > 70% → Scale Out.
        
    - Metric: CPUUtilization < 30% → Scale In.
        

---

### **3. Benefits of This Setup**

- **High Availability**: If one task dies, ALB routes traffic to healthy tasks.
    
- **Elasticity**: Auto scales based on load.
    
- **Central Entry Point**: ALB handles routing, health checks, SSL termination.
    

---

### **4. Optional Enhancements**

- **Sticky Sessions**: If your app stores user sessions in memory (not recommended), ALB can enable session stickiness.
    
- **Caching Layer**: Add CloudFront or Redis for caching.
    
- **Blue/Green Deployments**: Use CodeDeploy with ECS for zero-downtime releases.