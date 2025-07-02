# ALB with EC2 Autoscaling

## The Business Challenge

A small business I worked for faced instability in its web application delivery as user traffic increased. During peak usage, the system often became unresponsive due to fixed compute resources, leading to degraded user experiences and lost business opportunities. Additionally, inconsistent server configurations and limited visibility into system performance made troubleshooting difficult and delayed response times. The lack of an automated scaling mechanism and clear security boundaries further increased operational risk.

## The Solution: Automated, Scalable, and Monitored Application Deployment

To resolve these issues, my team implemented a dynamic and secure deployment workflow through several AWS services. The solution was designed to ensure high availability, consistent configuration, and responsive scaling to meet fluctuating demand.

### Key Workflow Components:

#### 1. Application Load Balancer (ALB)
An **Application Load Balancer** is deployed to serve as the single point of entry for all HTTP/HTTPS traffic. It distributes incoming requests evenly across available compute resources, ensuring resilience and fault tolerance.

#### 2. Auto Scaling Group (ASG) with Launch Template
An **Auto Scaling Group (ASG)** is configured to maintain a fleet of EC2 instances, starting with a baseline of 1. These instances are provisioned automatically from a **Launch Template**, which standardizes configurations including:
- AMI (Amazon Machine Image)
- Instance type
- Initialization scripts

#### 3. Dynamic Scaling Policies with CloudWatch Alarms
**Amazon CloudWatch** monitors critical performance metrics such as CPU utilization and request count. Alarms are triggered based on predefined thresholds, activating scaling policies that automatically add or remove instances based on real-time demand.

#### 4. Network Security via Security Groups
**Security Groups** are applied to control traffic flow:
- The load balancer's security group permits inbound HTTP traffic.
- The instances within the ASG are restricted to receive only traffic from the load balancer. This ensures secure, controlled access at every layer.

---

## Core Services in the Workflow

| Service                           | Description |
|-----------------------------------|-------------|
| **Amazon EC2 (Elastic Compute Cloud)** | Hosts the application instances that handle business logic and web traffic. |
| **EC2 Launch Templates**           | Define instance configurations like AMI, instance type, key pair, and user data for consistent deployments. |
| **Amazon EC2 Auto Scaling (ASG)**  | Automatically adjusts the number of EC2 instances based on demand and policies. |
| **Elastic Load Balancing (ELB)**   | Distributes incoming HTTP requests across multiple instances for high availability and fault tolerance. |
| **Amazon CloudWatch**              | Monitors system performance and triggers alarms based on predefined metrics. |
| **CloudWatch Alarms**              | Automatically initiate scaling actions or send alerts based on metric thresholds. |
| **Amazon VPC (Virtual Private Cloud)** | Provides a logically isolated network where all components securely communicate. |
| **Security Groups**                | Act as virtual firewalls that control inbound and outbound traffic to resources. |

---

## Steps

### Step 1: Create a Security Group for the Application Load Balancer and Auto Scaling Group

1. Log into the AWS console → EC2 → Security Groups → Create Security Group.
2. Name the ALB security group. For example: `MyALBSG`.
3. Edit the inbound rules to allow traffic from anywhere on the internet via HTTP or HTTPS.

---
![1](https://github.com/user-attachments/assets/0549a0ab-ade9-439c-8ea8-7139384b2e7c)

### Create Another Security Group for the Auto Scaling Group

1. Create a new security group for the **Auto Scaling Group** (ASG) and name it something like `MyASGSG`.
2. Edit the **inbound rules** to allow **All TCP traffic** from the **Application Load Balancer (ALB) security group**.

#### Explanation of Security Group Configuration:
The key concept of this security group configuration is to ensure that **only traffic routed through the ALB** (and not directly from the public internet) can reach the EC2 instances. This setup guarantees that regardless of the port, traffic can only flow securely through the ALB, which acts as the gateway to the EC2 instances. This ensures the EC2 instances within the Auto Scaling Group are only exposed to traffic that comes through the load balancer, maintaining security boundaries and preventing direct access from the public internet.

---

This configuration ensures that **users from around the world** access the website via the ALB, and the ALB in turn accesses the EC2 instances securely through the ASG security group configuration.
![2](https://github.com/user-attachments/assets/890ebc68-fb06-4dfd-90f7-a364838a797c)

### Add an Additional Inbound Rule for SSH Access

1. In the **Auto Scaling Group (ASG) security group (`MyASGSG`)**, add an additional inbound rule to allow **SSH traffic (Port 22)**.
2. Configure the rule to allow SSH access **from the public internet**. Typically, this can be set to `0.0.0.0/0` to allow access from any IP, or a more restricted IP range depending on your security requirements.

#### Explanation of the SSH Rule:
The additional SSH rule allows administrators to remotely access the EC2 instances in the Auto Scaling Group for troubleshooting, maintenance, or configuration purposes. By allowing SSH on Port 22 from the public internet, the instances are accessible for management but remain protected by the security group's other configurations.

**Important Consideration:** 
To ensure security, it’s best to limit SSH access to specific IP addresses (such as the IP range of your office network) rather than allowing it from `0.0.0.0/0`, which would allow anyone from the public internet to attempt an SSH connection. Always follow best practices for SSH key management and access control.

---

This setup allows secure management of EC2 instances in your Auto Scaling Group while maintaining strict access control through the security group configuration.
![3](https://github.com/user-attachments/assets/8c46588a-0bc0-4cb4-a2ab-b156e629a850)

### Create a Security Group for Auto Scaling Group

- Log into the AWS console → EC2 → Security Groups → Create Security Group.
- Name the security group (e.g., `MyASGSG`).
- Edit the inbound rules to allow traffic as required (e.g., HTTP, HTTPS, or SSH).

---

### Step 2: Create a Launch Template for the Auto Scaling Group

- EC2 Dashboard → **Instances** → **Launch Templates** → **Create Launch Template**.
- Name the Launch Template (e.g., `LaunchTemplateForEC2`).
- Under **Launch Template Content**, configure the following settings:
  - **Preferred AMI**: Select the desired AMI for your EC2 instances (e.g., `Ubuntu AMI`).
  - **Instance Type**: Select your preferred instance type (e.g., `T2.nano`). This is the size of the instance that will be deployed by the Auto Scaling Group when it scales horizontally.
  - **SSH Key Pair**: For remote SSH login to EC2 instances, create a keypair compatible with your operating system. If you plan to access instances via **AWS CloudShell**, keypairs may not be necessary.

- Under **Network Settings** → **Security Group**:
  - Select the security group created earlier for the Auto Scaling Group (e.g., `MyASGSG` with security group ID `sg-0369f40b5ced13dbc`).

- Under **Advanced Details**, provide a **User Data Script** that will run automatically on every new EC2 instance launched by the Auto Scaling Group. This script will handle the installation and setup of the application, ensuring that each instance is fully configured and ready to serve traffic as soon as it starts.

  **Example: Apache Web Server Installation Script**
**#!/bin/bash**
Specifies that the script should be executed using the Bash shell.

**apt update -y** - 
Updates the list of available packages and their versions from the configured repositories,
The -y flag automatically answers "yes" to prompts during the update process.

**apt install -y apache2** - 
Installs the Apache2 web server.
The -y flag automatically confirms the installation without requiring user input.

**systemctl start apache2** - 
Starts the Apache2 service, making the web server run and ready to serve HTTP requests.

**systemctl enable apache2** - 
Configures Apache2 to start automatically on boot.
This ensures that the web server remains running even after the instance is restarted.

**echo "<h1>This message from : $(hostname -i)</h1>" > /var/www/html/index.html** - 
Creates a simple HTML page with a message that includes the instance's internal IP address.

**$(hostname -i)** -  retrieves the instance's internal IP address.
This message is written to /var/www/html/index.html, which is the default location where Apache serves its web content.

- Create launch template

  ### Step 3: Create an Auto Scaling Group

1. Go to the **EC2 Dashboard** → **Auto Scaling Groups** → **Create Auto Scaling Group**.

2. Provide a name for the Auto Scaling Group, for example: `AutoScalingForEC2`.

3. Select the **Launch Template** you created earlier from the drop-down menu.

This will create an Auto Scaling Group that automatically launches and scales EC2 instances based on the Launch Template you provided earlier.
![4](https://github.com/user-attachments/assets/d829bf94-4c9a-45b9-a3e3-a741049c5c91)

4. Click on **Next** to edit the networking configuration.

5. Choose the **Availability Zones** where your application will be deployed. It’s not mandatory to choose multiple Availability Zones (AZs) for an Auto Scaling Group (ASG), but doing so is a best practice for production environments to ensure your application is resilient and fault-tolerant.

6. Click on **Next** to set up the load balancing configuration.

7. If there is no prior load balancer, select **Attach a new load balancer**.

8. Under **Load Balancer Type**, choose **Application Load Balancer**.

9. Name the load balancer (e.g., `ALBForASG`).

10. Select **Internet Facing** to allow the load balancer to accept traffic from the internet.

11. Under **Default Routing (Forward to)**, select **Create a target group** to create a target group on the fly.

    - **Defining Traffic Destination**: A Target Group is a logical grouping of backend resources (such as EC2 instances) that the Application Load Balancer (ALB) will send traffic to. When you create an ALB and configure it to forward traffic, it needs to know where to send that traffic. This is where the Target Group comes in—it specifies the EC2 instances (or IP addresses) that should receive the incoming traffic.

12. Under **Health Checks**, select the option that turns on health checks to monitor the health of instances in the target group.

13. Click on **Next**.

14. Configure **Group Size and Scaling**:
    - **Desired Capacity**: Provide the number of instances you want to start with (e.g., 1 instances).
    - **Minimum Capacity**: Set the minimum number of instances allowed in the Auto Scaling Group (e.g., 1 instance).
    - **Maximum Capacity**: Set the maximum number of instances that the Auto Scaling Group can scale up to (e.g., 2 instances).

15. Click on **Next** to create the Auto Scaling Group.

Once these steps are completed, the Auto Scaling Group will be created with the specified load balancer and scaling configuration.

### Confirm Auto Scaling Group Creation

1. **Check for Load Balancer**:
   - Go to the **EC2 Dashboard** → **Load Balancers**.
   - Verify that the **Application Load Balancer** (ALB) you created is listed.

2. **Check for EC2 Instances in the Load Balancer**:
   - Go to the **EC2 Dashboard** → **Instances**.
   - Ensure that the EC2 instances created by the Auto Scaling Group (ASG) are registered with the load balancer.

3. **Check the Auto Scaling Group**:
   - Go to the **EC2 Dashboard** → **Auto Scaling Groups**.
   - Click on the newly created Auto Scaling Group.
   - Under the **Activities** tab, you can view a list of activities that the Auto Scaling Group has performed, such as scaling actions (launching or terminating instances).
![5](https://github.com/user-attachments/assets/a81d1b7a-7f2d-4482-b6b1-32a2cf05edfa)

### Update the Security Group for the Load Balancer

Note: Since the Load Balancer was created along with the Auto Scaling Group (ASG), it is initially using the ASG security group. To update this:

1. Go to the **EC2 Dashboard** → **Load Balancers**.

2. Click on the newly created **Load Balancer**.

3. In the **Description** tab, find the **Security** section.

4. Click on the **Security Group name** to open the Security Group settings.

5. In the Security Group settings, remove the **ASG security group** and add the **ALB security group** (the security group you created specifically for the Application Load Balancer).

This ensures that the Load Balancer uses the correct security group, providing proper security isolation and access control.

### Step 6: Test the Load Balancer Deployment

1. Go to the **EC2 Dashboard** → **Load Balancers**.

2. Select the **Application Load Balancer** that you created.

3. In the **Description** tab, find the **DNS Name** under the **Description** section.

4. Copy the **DNS Name**.

5. Paste the **DNS Name** into your browser's address bar and press Enter.

This should load the website hosted on your EC2 instances, routed through the Load Balancer. If everything is configured correctly, you should see the default page served by the EC2 instances (e.g., the Apache web server default page or the custom page you set up).
![6](https://github.com/user-attachments/assets/e83d7a5c-9b02-4cd9-9a56-19e608b3dd6b)
![7](https://github.com/user-attachments/assets/2c6a9d5a-af32-4552-a22e-1d0f56e3e9af)

This test confirms that the instance is healthy and functioning correctly through the Load Balancer.

- By accessing the website through the **DNS name** of the Load Balancer, we can confirm that the EC2 instance is reachable and serving content properly. Even without knowing the **instance's IP address** or **port**, we are able to access the website directly via the Load Balancer.
  
- Attempting to access the **EC2 instance directly** using its **IP address** and **port** will not return any results. This is because the instance is behind the Load Balancer, and all incoming traffic is routed through it. The Load Balancer ensures that only authorized traffic can reach the instances, based on the security group settings and routing configuration.

### Step 4: Create Auto Scaling Policies

1. Go to the **EC2 Dashboard** → **Auto Scaling Groups**.

2. Select the **Auto Scaling Group** you created earlier.

3. Under the **Auto Scaling Group** details, click on **Dynamic Scaling**.
![8](https://github.com/user-attachments/assets/9d314da4-fe2b-4961-be88-464e69ca105b)

4. From the list of scaling policies (e.g., **Dynamic Scaling**, **Schedule Action Policy**, **Predictive Scaling Policies**), choose the policy that best suits your needs. For example, select **Dynamic Scaling Policy**.

5. **Choose the Policy Type**:
   - Select **Target Tracking Policy**. Choosing this option allows the Auto Scaling Group to automatically adjust the number of instances to maintain a desired metric, like **CPU utilization**, within a specified range. This helps to ensure optimal performance while maintaining cost efficiency.

6. **Set the Metric Type**:
   - Choose **CPU Utilization** as the metric to scale your instances.

7. **Set the Target Value**:
   - Set the target value to the threshold of your choice (e.g., **30**). 

   - **Explanation**: By setting the average CPU utilization to 30%, the Auto Scaling Group will attempt to maintain an average CPU usage of 30% across all instances. If CPU usage exceeds this threshold, the Auto Scaling Group will add new instances to handle the load. If CPU usage falls below the target, instances will be terminated to optimize cost and performance.

8. Save the scaling policy to apply it to the Auto Scaling Group.
![9](https://github.com/user-attachments/assets/93b5e02e-bef1-4652-ac3d-b63b0f114c61)

9. Create

### Step 5: Confirm Configuration via CloudWatch Alarm

1. With the scaling policy you created, a CloudWatch alarm was automatically set up in the background to trigger whenever the threshold is met.

2. To confirm this setup:
   - Go to **CloudWatch** → **Alarms** → **All Alarms**.

3. Here, you will see the alarm associated with the scaling policy. This alarm will be triggered based on the metric you defined (e.g., CPU utilization) and the threshold you set.

4. The alarm will monitor the defined metric and activate the scaling actions, either scaling up or scaling down your Auto Scaling Group based on the configured conditions.
![10](https://github.com/user-attachments/assets/b9177298-efbd-48d4-a503-e866f815a300)

### Step 5: Confirm Configuration via CloudWatch Alarm

1. With the scaling policy you created, two CloudWatch alarms were automatically set up in the background:
   - **Scale-Out Alarm**: Triggers when **CPU utilization** is below **21%**, indicating that more instances are needed to handle the load.
   - **Scale-In Alarm**: Triggers when **CPU utilization** exceeds **30%**, indicating that fewer instances are needed to optimize resource usage and cost.

2. To confirm this setup:
   - Go to **CloudWatch** → **Alarms** → **All Alarms**.

3. You should see both alarms listed here:
   - One for scaling out when CPU utilization is below 21%.
   - One for scaling in when CPU utilization exceeds 30%.

4. These alarms are directly tied to your scaling policies and will trigger the respective actions to either scale in or scale out your Auto Scaling Group.

Here’s the markdown for **Step 6: Test Out the Auto Scaling Group**:

### Step 6: Test Out the Auto Scaling Group

1. **Connect to the EC2 Instance**:
   - Go to the **EC2 Dashboard** → **Instances**.
   - Select the **EC2 instance** created by the Auto Scaling Group.
   - Click on **Connect** → **Connect via EC2 Serial Console**.

2. **Install Stress Utility**:
   To stress the CPU of the EC2 instance, you need to install the **stress** utility:

   ```bash
   $ sudo apt install stress -y
   ```

3. **Add Stress to the EC2 Instance**:
   Use the **stress** utility to add load to the CPU. This will simulate high CPU utilization and should trigger the Auto Scaling Group to scale out:
   ```bash
   $ sudo stress --cpu 12 --timeout 240s
   ```

4. **Confirm the Stress Action**:
   - Go to the **Auto Scaling Group** → **Monitoring** → **EC2**.
   - Observe the **CPU utilization** metric to confirm that the instance is under load.
   
   Once the CPU utilization exceeds the defined threshold (e.g., 30%), the Auto Scaling Group should automatically add new instances to handle the load.

5. **Verify Scaling Action**:
   - Check if the **Auto Scaling Group** has scaled out by looking at the number of instances. You should see the group add additional EC2 instances as the CPU utilization remains high.

This test validates that your Auto Scaling Group can respond to increased CPU load by automatically adding instances to maintain performance.
![11](https://github.com/user-attachments/assets/ba008c74-0f37-4580-9020-6ed2d0972d61)
![12](https://github.com/user-attachments/assets/19b4553e-8454-47cf-a3cc-28e9d2046b4b)

- It surged to over 90%.
- Confirm that the alarm was triggered in **CloudWatch**.
![13](https://github.com/user-attachments/assets/85b7d397-d7b5-4294-87c7-79ace8f0776d)

- The alarm was triggered as a result of the CPU surge.
- Check the **Auto Scaling Group** activity log to confirm the automatic creation of a new instance.
  - Go to the **Newly Created Auto Scaling Group** → **Activity**.
![14](https://github.com/user-attachments/assets/2b442c87-ecc2-44a5-88a6-0efd801d01fc)

- Confirm the creation of a new instance from the **EC2 Instance Dashboard**.
  - Go to the **EC2 Dashboard** → **Instances**.
  - Check for the newly launched instance that was created automatically by the Auto Scaling Group in response to the alarm and surge in CPU utilization.
![Screenshot 2025-04-15 021210](https://github.com/user-attachments/assets/3b6c4b67-16fb-4a85-bc8e-59bebf6d4c1f)

- Confirm the rerouting of traffic from one instance to another by the load balancer.
  - Copy the **DNS name** of the **Load Balancer**.
  - Paste the DNS name in a browser and verify that traffic is routed to the newly created instance.
  - The load balancer will automatically handle the rerouting of traffic to available instances, ensuring seamless access to the application.
![Screenshot 2025-04-15 021443](https://github.com/user-attachments/assets/8b6a6421-bfec-4172-8af6-88ea5c9897ef)
![Screenshot 2025-04-15 021525](https://github.com/user-attachments/assets/cb3bac28-c751-4ba7-99da-fe83d72d3bff)


```




















