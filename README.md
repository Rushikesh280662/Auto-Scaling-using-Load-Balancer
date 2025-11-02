# Hands-on Practice: AWS Auto Scaling **With** a Load Balancer

**Goal:**  
Create a resilient, auto-scaling web setup on AWS using EC2, NGINX, AMI, Launch Templates, an Application Load Balancer (ALB), and an Auto Scaling Group (ASG) with a target-tracking scaling policy.

---

## Prerequisites
- An AWS account with permissions to manage EC2, AMIs, Load Balancers, Launch Templates, and Auto Scaling.  
- Basic knowledge of SSH and Linux terminal commands.  
- A locally stored key pair (PEM) for SSH access.  
- AWS Console access. Work within a single region for the lab.

---

## Architecture Summary
1. Launch and configure a single EC2 instance (Ubuntu) with NGINX.  
2. Create an AMI from the configured instance.  
3. Create a Launch Template that references the AMI.  
4. Create an Auto Scaling Group (ASG) and attach an Application Load Balancer (ALB).  
5. Configure a target-tracking scaling policy based on CPU utilization.  
6. Validate scaling by generating CPU load and observing ASG behavior.

> Note: This hands-on lab demonstrates a typical production-like autoscaling pattern. Ensure you monitor costs and clean up resources after testing.

---

## Detailed Step-by-Step

### 1) Launch base EC2 instance (Ubuntu)
1. Sign in to AWS Console → EC2 → Launch Instance.  
2. Choose an OS (e.g., **Ubuntu Server 22.04 LTS**).  
3. Instance type: `t2.micro` (or as required).  
4. Create or select a **Key pair**. Download and store the `.pem` file securely.  
5. Network settings:  
   - Edit network settings → **Auto-assign Public IP** → Enable.  
   - Create a new security group (e.g., `launch-wizard-2`).  
6. Security Group Inbound rules:  
   - **HTTP** (port 80) → `0.0.0.0/0` (Anywhere)  
   - **SSH** (port 22) → your IP (recommended) or `0.0.0.0/0` for quick testing (less secure)  
7. Launch the instance and note its **Public IP**.

--- 

### 2) Connect and configure NGINX
From your machine (PowerShell / Terminal):

```bash
ssh -i .\path\to\public-server.pem ubuntu@<PUBLIC_IP>
```

Once connected:

```bash
sudo apt-get update
sudo apt-get install nginx -y
sudo service nginx start
sudo systemctl enable nginx.service
sudo service nginx status
```

Configure the webroot:

```bash
cd /var/www/html
ls
sudo rm index.nginx-debian.html
sudo nano index.html
```

Add the following content:

```html
<!doctype html>
<html>
  <body>
    <h1>This is Auto-Scaling With Using Load Balancer</h1>
  </body>
</html>
```

Save and exit (Ctrl+S, Ctrl+X). Verify the page at `http://<PUBLIC_IP>`.

--- 

### 3) Create AMI from configured instance
1. In EC2 console, select the instance → **Actions → Image and templates → Create Image**.  
2. Name the AMI (e.g., `asg-ubuntu-nginx-ami-lb`).  
3. Click **Create Image**.  
4. Go to **AMIs** and wait until AMI `State` is **Available**.  
5. Optionally terminate the original instance to save costs.

--- 

### 4) Create Launch Template
1. EC2 console → **Launch Templates** → **Create launch template**.  
2. Enter name and description.  
3. For **AMI**, choose your created AMI (`asg-ubuntu-nginx-ami-lb`).  
4. Choose Instance Type (e.g., `t2.micro`).  
5. Choose Key Pair and Security Group (`launch-wizard-2`).  
6. (Optional) Add **User Data** to bootstrap instances at launch.  
7. Create the Launch Template.

--- 

### 5) Create Auto Scaling Group (ASG) & Application Load Balancer (ALB)
1. EC2 console → **Auto Scaling Groups** → **Create Auto Scaling group**.  
2. Select the **Launch Template** you created.  
3. Give the ASG a name (e.g., `asg-with-lb-demo`).  
4. **Network**: Select 2 or more public subnets in different Availability Zones. (ALB will be internet-facing.)  
5. For **Load balancer** step, choose **Attach to a new load balancer** → **Application Load Balancer**.  
   - Name the ALB (e.g., `alb-asg-demo`).  
   - Scheme: **Internet-facing**.  
   - Listener: HTTP (port 80).  
6. **Target Group**: Create a new target group (e.g., `tg-asg-demo`) with protocol HTTP and target type `instance`.  
7. Continue through the wizard and set **Health check grace period** to **30 seconds** (instead of default 300).  
8. Set desired capacity values:  
   - Min: 1  
   - Desired: 1  
   - Max: 3  
9. Under **Scaling policies**, choose **Target tracking scaling policy**:  
   - Metric: **Average CPU utilization**  
   - Target value: **30%**  
   - Instance warm-up: **30 seconds**  
   - (Optional) Enable **Disable scale-in for scale-out policy** if you want to prevent scale-in after traffic drops.  
10. Enable monitoring and review all settings. Click **Create Auto Scaling group**.

--- 

### 6) Validate ASG + ALB behavior
- After ASG creation, an instance will be launched and registered with the ALB target group. Note the ALB DNS name (found in the Load Balancer details).  
- Visit `http://<ALB-DNS>` requests should route to one of the instances and display the index page.

--- 

### 7) Generate load to trigger scaling (stress test)
1. SSH into the currently running instance(s):  
   ```bash
   ssh -i .\path\to\public-server.pem ubuntu@<INSTANCE_PUBLIC_IP>
   ```  
2. Install `stress` utility (on Debian/Ubuntu):  
   ```bash
   sudo apt-get update
   sudo apt-get install stress -y
   ```  
3. Run a CPU stress for testing (example: 2 workers for 120 seconds):  
   ```bash
   stress --cpu 2 --timeout 120
   ```  
   - `--cpu 2` creates two CPU-bound workers.  
   - `--timeout 120` runs the test for 120 seconds.  
   - During this time CPU utilization on that instance will spike and may trigger ASG scaling if the average across instances exceeds 30%.

> Warning: Running `stress` can make the instance unresponsive. Use a short timeout and monitor via CloudWatch and the ASG activity history.

--- 

### 8) Monitor scaling via CloudWatch & ASG activity
- In EC2 console, select the instance and open **Monitoring** → view **CPU utilization** graph. Adjust time range as you wish for granular view.
![Traffic 1](https://github.com/user-attachments/assets/1acb0a63-67fd-494d-93ba-a2837b622d59)

- In **Auto Scaling Groups** → select your ASG → **Activity history** to see scale-out and scale-in events.  
- Watch for new instances launching when the average CPU crosses the target threshold and for instance termination when load subsides (unless you disabled scale-in).
![Cpu utilization](https://github.com/user-attachments/assets/61757cc7-1f62-49c5-a259-063029f15e22)

--- 

## Testing Notes & Behavior
- With **Max = 3** and **target CPU = 30%**, ASG will launch instances one-by-one as the target is breached. Instance warm-up delays ensure metrics are stable before further scaling.  
- If **Disable scale-in** is checked, ASG will not reduce instance count automatically after traffic subsides; otherwise, ASG will scale in based on policy and metric thresholds.  
- Health checks via ALB ensure traffic only goes to healthy instances. If an instance fails health checks, the ASG or ALB will replace/unregister it accordingly.

--- 

## Troubleshooting
- **No response from ALB:** Ensure security groups allow HTTP on ALB and target instances; target group shows healthy targets.  
- **Scaling not triggering:** Verify CloudWatch metric is firing, target tracking policy is attached, and instance warm-up is reasonable.  
- **High launch latency:** Check AMI, user-data scripts, and network configuration; slow boot can delay instance readiness.  
- **Permissions issues:** Ensure IAM role/policies allow ASG to launch instances and create/modify required resources.

--- 

## Cost and Clean-up
- Delete the Auto Scaling Group, ALB, and target group.  
- Terminate any leftover instances and delete related EBS snapshots.  
- Deregister and delete AMIs if not needed.  
- Delete Launch Templates.  
- Monitor billing EC2 instances, ALB hours, and EBS snapshots can accrue charges.

--- 

## Flowchart for this Practice
<img width="977" height="669" alt="EC2 Instance and Auto Scaling Group Setup Flow" src="https://github.com/user-attachments/assets/eeba8d45-b3b9-47eb-ab41-89dee5e5d800" />

--- 

## Author

**Rushikesh Dase**  
Cloud Computing Enthusiast | AWS Learner  
📧 *daserushikesh@gmail.com*

### 🔗 Connect with Me
If you’d like to check out more of my cloud-related hands-on practices and projects, connect with me on [LinkedIn](https://www.linkedin.com/in/rushi-dase).
