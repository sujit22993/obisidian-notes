#deployment #devops #cicd #blue-green-deployment #rolling-deployment #canary-deployment #infrastructure #cloud-architecture #best-practices 

### 1️⃣ **Rolling Update**

- **How it works**: Gradually replaces old tasks with new ones, keeping the service running.
    
- **Pros**:
    - No downtime.
    - Simple and built-in with ECS.
        
- **Cons**:
    - If new version has bugs, rollback is slower.
        
- **When to use**:
    - Most standard production workloads where instant rollback isn't critical.
        
---

### 2️⃣ **Blue/Green Deployment (via CodeDeploy)**

- **How it works**:
    
    - **Blue** = Current version.
        
    - **Green** = New version.
        
    - Route traffic to green only after validation.
        
- **Pros**:
    
    - Zero downtime.
        
    - Easy rollback by switching traffic back to blue.
        
- **Cons**:
    
    - Requires double capacity temporarily.
        
- **When to use**:
    
    - Critical production apps, where rollback speed matters.
        
    - Highly available microservices.
        

---

### 3️⃣ **Canary Deployment**

- **How it works**:
    
    - Roll out to a **small % of traffic/users** first.
        
    - Monitor before full rollout.
        
- **Pros**:
    
    - Early bug detection in production with minimal risk.
        
- **Cons**:
    
    - More complex routing rules (ALB, API Gateway, or service mesh needed).
        
- **When to use**:
    
    - Large user base, mission-critical apps.
        

---

### 4️⃣ **Shadow Deployment**

- **How it works**:
    
    - New version runs alongside the old one but **receives a copy of real traffic**.
        
    - Not exposed to users.
        
- **Pros**:
    
    - Great for performance/load testing in real conditions.
        
- **Cons**:
    
    - Extra infra cost.
        
- **When to use**:
    
    - Validating big refactors before full rollout.
        

---

## ✅ Best Practices (Widely Used)

1. **Default for most teams** → Rolling Updates (simplest + built-in).
    
2. **For high-availability production** → Blue/Green with CodeDeploy.
    
3. **For risky features** → Canary first, then Rolling/Blue-Green.
    
4. **Always use Health Checks**:
    
    - ECS task health + ALB health.
        
    - Avoid sending traffic to unhealthy containers.
        
5. **Automate Rollbacks**:
    
    - Integrate alarms in CloudWatch → trigger rollback.
        
6. **Immutable Deployments**:
    
    - Never update containers in-place; always launch new tasks.
        
7. **Tag & Version Images**:
    
    - Use semantic version tags (`v1.0.1`) + commit hashes.
        
8. **CI/CD Integration**:
    
    - Common stack: **CodePipeline + CodeBuild + ECS**.
        
    - Alternative: GitHub Actions → ECS Deploy.
        
9. **Capacity Planning**:
    
    - For Blue/Green or Canary, ensure you have extra compute capacity during rollout.
        

---

💡 **Rule of Thumb:**

- **Start simple** with ECS Rolling Updates.
    
- **Move to Blue/Green** when uptime and rollback speed are critical.
    
- **Use Canary** for risky feature rollouts in big systems.