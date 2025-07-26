### Overview

- Package your Java app (Spring Boot) as a .jar

- Upload it to S3 (or access from GitHub directly in EC2)

- Create a Launch Template that installs Java & runs your app

- Create an Auto Scaling Group (ASG) using that template

- Attach a Load Balancer

### Step-by-Step Deployment 

Step 1: Package Your Java App
It's a Spring Boot app:

> mvn clean package

Output: target/demo-0.0.1-SNAPSHOT.jar

Step 2: Upload .jar to S3

> aws s3 cp target/demo-0.0.1-SNAPSHOT.jar s3://your-bucket-name/demo.jar

Make sure:

- S3 bucket is in the same region as your ASG

- You have access from EC2 (via IAM role)

Step 3: Create an IAM Role for EC2

Create a role (e.g., EC2S3ReadOnlyRole) with the following:

AWS managed policy: AmazonS3ReadOnlyAccess

Trust entity: EC2

You’ll attach this to the instances in the launch template.

Step 4: Write EC2 User Data Script
This script will:

Install Java

Download the JAR from S3

Run it

```

#!/bin/bash
yum update -y
amazon-linux-extras enable corretto8
yum install java-1.8.0-amazon-corretto -y

# Download JAR from S3
aws s3 cp s3://your-bucket-name/demo.jar /home/ec2-user/demo.jar

# Run the app
nohup java -jar /home/ec2-user/demo.jar > /home/ec2-user/app.log 2>&1 &

```

Step 5: Create Launch Template
In AWS Console:

Go to EC2 → Launch Templates → Create Launch Template

Name: demo-java-template

Choose AMI: Amazon Linux 2

Choose instance type: t2.micro or similar

Key pair: (optional)

Security group: allow port 8080 & 22

IAM role: Select EC2S3ReadOnlyRole

Paste your User Data script

Save template

Step 6: Create Auto Scaling Group
Go to EC2 → Auto Scaling Groups → Create Auto Scaling Group

Choose launch template: demo-java-template

Name the group: java-asg-demo

Choose VPC & subnets (at least 2 for redundancy)

Attach ALB if needed

Set capacity:

Desired: 1

Min: 1

Max: 3

Scaling policy: use default for now (or CPU > 70%)

Review and create

Step 7: Access Your App
After instance launches:

Go to EC2 → Instances → get public IP

Visit:

http://<Public-EC2-IP>:8080/

You should see:
Hello from Auto Scaling Group!

Step 8: Add Load Balancer
Attach an Application Load Balancer:

Create ALB → Target Group → Add ASG as target

Add health check path: /

Update security group to allow port 80

Access via: http://<ALB-DNS>:80/

==============================================================
### Issues

EC2 Instances were showing as “Unhealthy” in the Target Group

The application was listening on port 8080, but the Target Group was created with port 80

Therefore, health checks failed with Request timed out

The ALB can't successfully reach the EC2 instance(s) in the target group.

The health checks are failing, so traffic isn't being routed

Your Java app is listening on port 8080, but the Target Group health check (and port forwarding) is set to port 80.

So the load balancer is checking http://<EC2_IP>:80/, but nothing is running on port 80 → resulting in health check failure → “Not reachable”.

=======================
### Fix Issue:
Option 1:
Created a new Target Group with port 8080

Updated ALB Listener Rule to forward traffic to the new Target Group

Configured Health Checks on path / and port traffic-port

Option 2: Change Only Health Check Port
If you want to keep port 80 for traffic but fix the health check only:

Go to EC2 → Target Groups → [your group] → Health checks

Click Edit

Change:

Port from 80 → traffic port or manually set to 8080

Save


==============================================================

### Optional Improvements to implement:
- Add CloudWatch alarms for scale-in/scale-out
- Use a custom domain with Route53
- Configure SSL (HTTPS via ACM + ALB listener on 443)



==============================================================

