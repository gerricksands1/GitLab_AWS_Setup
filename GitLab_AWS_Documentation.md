# GitLab AWS

## Requirements

- AWS
- GitLab CE Instance (Omnibus)
- Amazon Linux 2
    - t3.large
    - 2 cpu
    - 8 gb of ram
    - 8 gb storage
- VPC
    - IP: 10.0.0.0/16
    - 2 Public Subnets: 10.0.1.0/24 & 10.0.2.0/24, 2 Private Subnets: 10.0.3.0/24 & 10.0.4.0/24
    - 2 Availability Zones
    - 1 Internet Gateway
    - 1 NAT Gateway
- Security Groups
    - Name: alb-sg
        - Inbound Rules 1: IP version - IPv4, Type - HTTPS, Protocol - TCP, Port range - 443, Source - 0.0.0.0/0
        - Inbound Rules 2: IP version - IPv4, Type - HTTP, Protocol - TCP, Port range - 80, Source - 0.0.0.0/0
        - Outbound Rules: IP version - IPv4, Type - HTTP, Protocol - TCP, Port range - 80, Destination - 0.0.0.0/0
    - Name: gitlab-sg
        - Inbound Rules: IP version - IPv4, Type - HTTP, Protocol - TCP, Port range - 80, Source - albsg security group
        - Outbound Rules: IP version - IPv4, Type - All traffic, Protocol - All, Port range - All, Destination - 0.0.0.0/0
- Application Load Balance
- Web Application Firewall
- Session Manager
- AWS IAM
- AWS Certificate Manager
- Domain Registrar

## Steps

### 1. Creating the VPC

- Created a VPC with the above criteria
- The purpose being that the most secure way of building an application that will be open to the internet is to keep the vm inside of a private subnet and use an Application Load Balancer (ALB) for directing the traffic while using a Web Application Firewall (WAF) for protection.

### 2. Creating the Security Groups

- We're creating 2 separate security groups
- The values for both are above
- The way this works is that the security group rules for ALB will listening for HTTP and HTTPS traffic and will then send it out as HTTP traffic to the GitLab VM security group which will be listening for that traffic, but specifically from the ALB.
- This is the reason why the ALB will be in the public subnet while the vm will be in the private subnet.
- Because only HTTPS and HTTP traffic will be listened for there will be no way for any SSH traffic to get through limiting our ability to manage via PuTTY or even PowerShell ssh.

### 3. Managing the GitLab VM

- Since we don't have port 22 open so SSH isn't an option we'll need another method of accessing and managing our Amazon Linux VM.
- We're utilizing AWS Systems Manager Session Manager in order to remote into the vm.
- Got the Identity, Access, and Management console (IAM), go under Roles, and create a new Role.
- Under trusted entity type: AWS service
- Go to "Use case" and select EC2
- Under "Add permissions" search and check "AmazonSSMManagedInstanceCore", then hit next
- Name the role whatever makes sense to you. Example: EC2-SSM-Role
- You'll now have to attach the new role to the GitLab VM.
- Go to EC2 --> Instances --> Click your vm instance id --> Actions --> Securiyt --> Modify IAM Role
- Give it the role you just created.
- If the Linux version you have installed in the vm is Amazon Linux you shouldn't have to do anything else.
- You should now be able to go to the VM if it's running, click on it again, click the "Connect" button and go to the Session Manager tab and click the "Connect" button below.
- You should now see a new tab pop up with a cli. You are now inside your vm to manage it.
- If you added the NAT gateway into your private subnet and your vm is living there as well you should have no problem updating and installing the GitLab instance.

### 4. Installing and Configuring GitLab

- We're going to link (as of this writing): https://packages.gitlab.com/gitlab/gitlab-ce
- Since Amazon Linux is based on Fedora we'll be using the RPM quick install instructions
- Click the "RPM" button and copy (as of this writing): curl -s https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash
- Once the repository is added to your linux instance, run the command: "sudo dnf install gitlab-ce -y". You shouldn't need a password.
- Once the installation process is complete go to "/etc/gitlab" there should be a "gitlab.rb" file, this is the main configuration file. 
- You can use vim or nano to edit the file.
- Within the file you're looking for a line that says "External_URL" there should be no "#" in front of it. 
- Change it to whatever web url you're planning to use. Make sure "https://" is what you are using instead of "http://"
- Save your changes and exit. From there key in command "sudo gitlab-ctl reconfigure".
- Once the system is done configuring the site should be ready.
- Within the "/etc/gitlab" folder there should now be a new file called "initial_root_password". You'll have to open the file as sudo to access it.
- Write down the initial password because you'll need it for the initial login.

### 5. Prepare Certificate for HTTPS

- For this portion I'm not using Route 53 because my registrar is through www.namecheap.com
- The service we're going to use is the AWS Certificate Manager (ACM) to get a free certificate similar to "Let's Encrypt" with Nginx.
- Go to ACM and request a public certificate
- Fill in your "Fully Qualified Domain Name" (FQDN). An example could be: www.example.com or project.example.com
- Once the certificate is requested it will be pending, but not yet official.
- In this example we're using NameCheap as the registrar so we're creating the CNAME records there by managing the domain, going to "Advanced DNS", and from there creating the CNAME Record type.
- We're getting the CNAME record from the certificate itself.
- The CNAME name should be something like: _123abce3.project.example.com. We're only using the "_123abce3.project" portion for the name in NameCheap's CNAME record.
- Then copy the CNAME value and paste that into NameCheap's value.
- Give it time to connect, it'll take a few minutes to an hour before the certificate's status changes from pending to success.

### 6. Create Application Load Balance and Web Application Firewall

- Go to EC2 and got to load balancers on the left side
- Create the Application Load Balancer specifically.
- Pick internet facing, select the vpc, and select both of the availability zones.
- For the "Listeners and Rules" portion set Protocol:Port to "HTTPS:443", set "Default Action" to forward to a group target.
- The group target should be the GitLab VM
- Within the port-target page go to tab "health check" hit the edit tag and change it to "/users/sign_in"

### 7. Add the CNAME to Registrar to Connect to Application Load Balancer

- Go to the load balancer, port-alb, and copy the DNS name.
- Go to namecheap and add under the AdvancedDNS and create another CNAME entry
- The host will be either www or project (example for: www.example.com or project.example.com)
- Then the DNS name will be the CNAME value
- You'll want to wait a few hours for all computers to be able to recognize it, but you can check dns checker sites (www.dnschecker.org) after 10 minutes to verify if it works.

### 8. Configure Gitlab to Listen to HTTPS

- We're going to use Sessions Manager to access our VM
- Key command "sudo vim /etc/gitlab/gitlab.rb"
- This is what we need to edit:
```
gitlab_rails['trusted_proxies'] = ["10.0.0.0/16"]
nginx['listen_port'] = 80
nginx['listen_https'] = false
nginx['real_ip_trusted_addresses'] = ["10.0.0.0/16"]
nginx['real_ip_header'] = 'X-Forwarded-For'
nginx['real_ip_recursive'] = "on"
```
- Make sure that "External_URL" is set to what your address should be: https://project.example.com or what you're setting it to.
- This should allow the ALB to access the GitLab app and trusted to redirect with ssl traffic.