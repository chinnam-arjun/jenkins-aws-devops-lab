# AWS Remote Jenkins CI/CD Infrastructure

A step-by-step guide to building a distributed CI/CD environment on AWS using Jenkins, remote Docker agents, GitHub Webhooks, Docker Hub, SSH authentication, and EC2 deployment servers.

This guide covers the infrastructure from scratch, including AWS networking, instance preparation, Jenkins installation, SSH agent configuration, Docker installation, pipeline configuration, automated deployment, and troubleshooting.

The objective is to reproduce the architecture described in `README.md`.

---

# Table of Contents

1. Prerequisites
2. Target Architecture
3. AWS Infrastructure Preparation
4. Configure AWS Networking
5. Launch EC2 Instances
6. Configure Jenkins Controller
7. Configure Remote Docker Agent
8. Establish SSH Authentication
9. Connect Docker Agent to Jenkins
10. Configure Deployment Server
11. Configure Deployment SSH Authentication
12. Configure Jenkins Credentials
13. Configure GitHub Webhook
14. Configure Jenkins Pipeline
15. Execute and Verify CI/CD
16. Common Mistakes and Troubleshooting
17. Security Recommendations
18. Infrastructure Verification Checklist
19. Future Improvements

---

# 1. Prerequisites

Before starting, ensure you have the following:

### AWS

- AWS account.
- IAM permissions to create VPCs, subnets, route tables, security groups, EC2 instances, Elastic IPs, and NAT Gateways.
- AWS region: `us-east-1`.

### External Services

- GitHub account and repository.
- Docker Hub account.
- DuckDNS account for Jenkins domain configuration.

### Local Machine

- SSH client.
- Web browser.
- Terminal or PowerShell.

### Basic Knowledge

- Linux commands.
- SSH authentication.
- Docker fundamentals.
- AWS VPC and EC2 networking.
- Jenkins fundamentals.

---

# 2. Target Architecture

We will create three Ubuntu EC2 instances.

| Instance | Network | Responsibility |
|---|---|---|
| Jenkins Controller | Jenkins VPC - Public subnet | Pipeline orchestration |
| Docker Agent | Jenkins VPC - Private subnet | Docker build, testing and publishing |
| Deployment Server | Default VPC | Run the application container |

Additional infrastructure:

- Internet Gateway.
- NAT Gateway.
- Elastic IP for the NAT Gateway.
- Elastic IP for the deployment server.
- DuckDNS domain for Jenkins.
- Docker Hub repository.

### Communication Flow

```text
GitHub
   |
   | Webhook
   v
Jenkins Controller
   |
   | SSH - Private IP
   v
Remote Docker Agent
   |
   | Docker Build
   | Container Testing
   | Docker Push
   v
Docker Hub
   |
   | SSH - Public IP
   v
Deployment EC2
   |
   | Docker Pull
   v
Running Node.js Application
```

Important:

There are two independent SSH connections:

1. Jenkins Controller → Docker Agent.
2. Docker Agent → Deployment Server.

The deployment stage executes on the Docker agent. Therefore, the deployment server sees the Docker agent's NAT Gateway Elastic IP as the SSH source address.

---

# 3. AWS Infrastructure Preparation

## 3.1 Create a dedicated VPC

Navigate to:

AWS Console → VPC → Your VPCs → Create VPC.

Example configuration:

| Configuration | Value |
|---|---|
| Name | jenkins-vpc |
| IPv4 CIDR | 10.0.0.0/16 |
| Region | us-east-1 |

## 3.2 Create subnets

Create two subnets.

**Public subnet**

| Property | Value |
|---|---|
| Name | jenkins-public-subnet |
| CIDR | 10.0.1.0/24 |
| Availability Zone | us-east-1a |
| Auto-assign public IPv4 | Enabled |

**Private subnet**

| Property | Value |
|---|---|
| Name | jenkins-private-subnet |
| CIDR | 10.0.2.0/24 |
| Availability Zone | us-east-1a |
| Auto-assign public IPv4 | Disabled |

These CIDRs are examples. Ensure they do not overlap with your other VPC networks.

## 3.3 Create and attach an Internet Gateway

Navigate to:

VPC → Internet Gateways → Create Internet Gateway.

Name:

```text
jenkins-igw
```

Attach it to:

```text
jenkins-vpc
```

## 3.4 Configure the public route table

Create a route table:

```text
jenkins-public-rt
```

Associate it with:

```text
jenkins-public-subnet
```

Add this route:

| Destination | Target |
|---|---|
| 10.0.0.0/16 | local |
| 0.0.0.0/0 | Internet Gateway |

This provides internet connectivity to the Jenkins controller.

## 3.5 Create a NAT Gateway

Navigate to:

VPC → NAT Gateways → Create NAT Gateway.

Configuration:

```text
Name: jenkins-nat-gateway
Subnet: jenkins-public-subnet
Connectivity type: Public
Elastic IP: Allocate new Elastic IP
```

Wait until the NAT Gateway status becomes Available.

The NAT Gateway allows the private Docker agent to access external services such as GitHub and Docker Hub without requiring a public IP address.

## 3.6 Configure the private route table

Create:

```text
jenkins-private-rt
```

Associate it with:

```text
jenkins-private-subnet
```

Add:

| Destination | Target |
|---|---|
| 10.0.0.0/16 | local |
| 0.0.0.0/0 | NAT Gateway |

The resulting traffic flow:

```text
Docker Agent
     |
     v
Private Route Table
     |
     v
NAT Gateway
     |
     v
Internet Gateway
     |
     v
Internet
```

The private instance can initiate outbound connections, but the NAT Gateway does not allow unsolicited inbound internet connections to the private agent.

---

# 4. Configure AWS Security Groups

Create separate security groups for each instance.

## 4.1 Jenkins Controller Security Group

Name:

```text
jenkins-controller-sg
```

Inbound rules:

| Type | Port | Source | Purpose |
|---|---:|---|---|
| SSH | 22 | Your public IP/32 | Administration |
| Custom TCP | 8080 | GitHub webhook source ranges or temporary lab access | Jenkins UI and webhook |

For initial testing, you may temporarily allow port 8080 from anywhere. This is not recommended for a publicly exposed production Jenkins server.

Outbound:

Allow the required outbound traffic, including SSH to the private agent.

## 4.2 Docker Agent Security Group

Name:

```text
jenkins-docker-agent-sg
```

Inbound:

| Type | Port | Source | Purpose |
|---|---:|---|---|
| SSH | 22 | jenkins-controller-sg | Jenkins agent connection |

Use the controller security group as the source rather than opening SSH to the internet.

Outbound:

Allow the required outbound traffic.

The agent needs:

- HTTPS 443 for GitHub and Docker Hub.
- HTTP/HTTPS as required for package installation.
- SSH 22 to the deployment EC2.

## 4.3 Deployment Server Security Group

Name:

```text
deployment-server-sg
```

Inbound:

| Type | Port | Source | Purpose |
|---|---:|---|---|
| SSH | 22 | NAT Gateway Elastic IP/32 | Remote deployment |
| HTTP | 80 | 0.0.0.0/0 | Application access |

Do not open Docker daemon port 2375 or 2376 to the public internet.

---

# 5. Launch EC2 Instances

Launch three Ubuntu EC2 instances.

Use an Ubuntu Server LTS AMI and a suitable architecture and instance size for your workload.

### Instance 1: Jenkins Controller

```text
Name: jenkins-controller
VPC: jenkins-vpc
Subnet: jenkins-public-subnet
Public IPv4: Enabled
Security Group: jenkins-controller-sg
```

### Instance 2: Docker Agent

```text
Name: jenkins-docker-agent
VPC: jenkins-vpc
Subnet: jenkins-private-subnet
Public IPv4: Disabled
Security Group: jenkins-docker-agent-sg
```

### Instance 3: Deployment Server

```text
Name: node-app-deployment
VPC: Default VPC
Subnet: Default public subnet
Public IPv4: Enabled
Security Group: deployment-server-sg
```

Allocate and associate an Elastic IP with the deployment server.

This ensures its public IP does not change during normal stop/start operations.

### Record these values

```text
JENKINS_PUBLIC_IP=
JENKINS_DOMAIN=

DOCKER_AGENT_PRIVATE_IP=

NAT_GATEWAY_ELASTIC_IP=

DEPLOYMENT_ELASTIC_IP=
```

Do not publish private keys or credentials alongside these configuration values.

---

# 6. Configure Jenkins Controller

Connect to the Jenkins controller from your local machine.

```bash
ssh -i your-key.pem ubuntu@JENKINS_PUBLIC_IP
```

All commands in this section run on the Jenkins Controller.

## 6.1 Update Ubuntu

```bash
sudo apt update
sudo apt upgrade -y
```

Install essential packages:

```bash
sudo apt install -y \
    curl \
    wget \
    git \
    unzip \
    ca-certificates \
    gnupg \
    openssh-client
```

## 6.2 Install Java 21

Jenkins requires a supported Java runtime.

```bash
sudo apt install -y fontconfig openjdk-21-jre
```

Verify:

```bash
java -version
```

Expected major version:

```text
Java 21
```

## 6.3 Install Jenkins LTS

Add the Jenkins repository signing key:

```bash
sudo mkdir -p /etc/apt/keyrings

sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
```

Add the Jenkins LTS repository:

```bash
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
https://pkg.jenkins.io/debian-stable binary/ | \
sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
```

Install Jenkins:

```bash
sudo apt update

sudo apt install -y jenkins
```

Start and enable the service:

```bash
sudo systemctl enable jenkins

sudo systemctl start jenkins
```

Verify:

```bash
sudo systemctl status jenkins
```

Jenkins listens on port 8080 by default.

## 6.4 Retrieve initial administrator password

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Open:

```text
http://JENKINS_PUBLIC_IP:8080
```

Complete the initial setup.

- Enter the administrator password.
- Install suggested plugins.
- Create the administrator account.
- Complete the Jenkins configuration.

## 6.5 Install required plugins

Navigate to:

Manage Jenkins → Plugins → Available Plugins.

Install the following:

| Plugin | Purpose |
|---|---|
| Git | Git repository integration |
| Pipeline | Declarative pipeline support |
| GitHub | GitHub webhook integration |
| SSH Build Agents | Remote SSH agent connection |
| SSH Agent | SSH credentials inside pipeline |
| Credentials Binding | Access Jenkins credentials in pipeline |

Restart Jenkins if required.

## 6.6 Configure DuckDNS

Create a DuckDNS domain.

Example:

```text
arjun-jenkins.duckdns.org
```

Configure its IPv4 address to point to the Jenkins controller's public IP.

For an instance without an Elastic IP, update DuckDNS whenever the public IPv4 changes after a stop/start operation.

Verify:

```bash
nslookup arjun-jenkins.duckdns.org
```

The resolved IP should match the controller's current public IP.

---

# 7. Configure Remote Docker Agent

Connect to the Docker agent from the controller using its private IP.

You can execute commands through an existing administration path or an authorized SSH connection.

All commands in this section run on the Docker Agent.

## 7.1 Update Ubuntu

```bash
sudo apt update

sudo apt upgrade -y
```

## 7.2 Install Java

The remote agent requires Java to run the Jenkins agent process.

```bash
sudo apt install -y openjdk-21-jre-headless
```

Verify:

```bash
java -version
```

## 7.3 Install SSH Server

```bash
sudo apt install -y openssh-server
```

Enable SSH:

```bash
sudo systemctl enable ssh

sudo systemctl start ssh
```

Verify:

```bash
sudo systemctl status ssh
```

## 7.4 Create a dedicated Jenkins user

Create a dedicated Linux user for Jenkins agent execution.

```bash
sudo adduser --disabled-password --gecos "" jenkins
```

This creates the user without requiring a password for normal SSH-key-based authentication.

Verify:

```bash
id jenkins
```

## 7.5 Create the Jenkins agent working directory

```bash
sudo mkdir -p /home/jenkins/agent

sudo chown -R jenkins:jenkins /home/jenkins/agent
```

Set appropriate permissions:

```bash
sudo chmod 755 /home/jenkins/agent
```

This directory will act as the remote Jenkins agent's working directory.

---

# 8. Install Docker Engine on the Docker Agent

Docker must be installed on the machine that executes the Docker build and testing stages.

All commands in this section run on the Docker Agent.

## 8.1 Add Docker's official repository

```bash
sudo apt update

sudo apt install -y ca-certificates curl

sudo install -m 0755 -d /etc/apt/keyrings
```

Download Docker's GPG key:

```bash
sudo curl -fsSL \
https://download.docker.com/linux/ubuntu/gpg \
-o /etc/apt/keyrings/docker.asc
```

Set permissions:

```bash
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Add the Docker repository:

```bash
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

Update package lists:

```bash
sudo apt update
```

## 8.2 Install Docker Engine

```bash
sudo apt install -y \
    docker-ce \
    docker-ce-cli \
    containerd.io \
    docker-buildx-plugin \
    docker-compose-plugin
```

Enable Docker:

```bash
sudo systemctl enable docker

sudo systemctl start docker
```

Verify:

```bash
sudo docker run hello-world
```

## 8.3 Give Jenkins user Docker permissions

Add Jenkins to the Docker group:

```bash
sudo usermod -aG docker jenkins
```

Verify:

```bash
groups jenkins
```

Expected output includes:

```text
docker
```

Important:

Membership in the Docker group effectively grants root-equivalent privileges because Docker can mount and modify host filesystems.

Only grant this permission to trusted users.

---

# 9. Establish SSH Authentication: Controller to Agent

This is one of the most important parts of the architecture.

The Jenkins controller needs to authenticate with the private Docker agent using SSH public-key authentication.

There are two different SSH files involved:

- `authorized_keys` on the Docker agent.
- `known_hosts` on the Jenkins controller.

They perform different security functions.

## 9.1 Generate an SSH key pair on the controller

Run this on the Jenkins Controller.

The Jenkins service runs as the Linux user `jenkins`, so create its SSH directory with appropriate ownership.

```bash
sudo -u jenkins mkdir -p /var/lib/jenkins/.ssh

sudo chmod 700 /var/lib/jenkins/.ssh
```

Generate an Ed25519 key pair:

```bash
sudo -u jenkins ssh-keygen \
    -t ed25519 \
    -C "jenkins-controller-to-docker-agent" \
    -f /var/lib/jenkins/.ssh/docker-agent-key
```

For this dedicated service key, leave the passphrase empty if you intend to load it directly into Jenkins credentials without an interactive unlock mechanism.

You now have:

```text
docker-agent-key
docker-agent-key.pub
```

The private key must remain protected.

## 9.2 Copy the public key to the Docker agent

Display the public key on the controller:

```bash
sudo cat /var/lib/jenkins/.ssh/docker-agent-key.pub
```

Copy its entire contents.

Connect to the Docker Agent through your authorized administration method.

Create the SSH directory:

```bash
sudo -u jenkins mkdir -p /home/jenkins/.ssh

sudo chmod 700 /home/jenkins/.ssh
```

Add the controller public key:

```bash
sudo nano /home/jenkins/.ssh/authorized_keys
```

Paste the public key.

Set ownership and permissions:

```bash
sudo chown -R jenkins:jenkins /home/jenkins/.ssh

sudo chmod 700 /home/jenkins/.ssh

sudo chmod 600 /home/jenkins/.ssh/authorized_keys
```

The agent now trusts the corresponding private key holder for SSH authentication as the `jenkins` user.

## 9.3 Configure known_hosts on the controller

Return to the Jenkins Controller.

Obtain the Docker agent's host key:

```bash
sudo -u jenkins ssh-keyscan -H DOCKER_AGENT_PRIVATE_IP
```

Before trusting a host key, independently verify its fingerprint using the agent's console:

```bash
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

Once verified, add the host key to the controller:

```bash
sudo -u jenkins ssh-keyscan -H DOCKER_AGENT_PRIVATE_IP \
>> /var/lib/jenkins/.ssh/known_hosts
```

Set permissions:

```bash
sudo chown jenkins:jenkins /var/lib/jenkins/.ssh/known_hosts

sudo chmod 600 /var/lib/jenkins/.ssh/known_hosts
```

## 9.4 Test SSH authentication

From the Jenkins Controller:

```bash
sudo -u jenkins ssh \
-i /var/lib/jenkins/.ssh/docker-agent-key \
jenkins@DOCKER_AGENT_PRIVATE_IP
```

If authentication works, you should enter the Docker agent shell as:

```text
jenkins
```

Exit:

```bash
exit
```

At this point, SSH authentication between the controller and agent is working.

---

# 10. Configure Docker Agent in Jenkins

Open:

Jenkins → Manage Jenkins → Nodes → New Node.

Create a permanent agent.

Configuration:

| Setting | Value |
|---|---|
| Node name | docker-agent |
| Type | Permanent Agent |
| Number of executors | 1 |
| Remote root directory | /home/jenkins/agent |
| Labels | docker |
| Usage | Only build jobs with label expressions |
| Launch method | Launch agents via SSH |
| Host | Docker Agent private IP |
| Credentials | SSH private-key credential |
| Host key verification | Known hosts file verification strategy |

### Create Jenkins SSH credentials

Navigate to:

Manage Jenkins → Credentials → System → Global credentials → Add Credentials.

Configuration:

```text
Kind: SSH Username with private key
Username: jenkins
Private Key: Enter directly
ID: docker-agent-ssh
Description: SSH access to private Docker agent
```

Paste the private key generated earlier:

```bash
sudo cat /var/lib/jenkins/.ssh/docker-agent-key
```

Do not paste the `.pub` file.

Select the credential while configuring the agent.

### Host key verification

Configure the SSH Build Agents plugin to use the known hosts file strategy.

The Jenkins service account must be able to read:

```text
/var/lib/jenkins/.ssh/known_hosts
```

Do not disable host-key verification simply to make the connection succeed.

Save the node configuration.

Jenkins should establish an SSH connection and launch the agent.

### Verify

Navigate to:

```text
Jenkins → Nodes → docker-agent
```

Expected status:

```text
Agent is online
```

If the connection fails, inspect:

```bash
sudo journalctl -u jenkins -f
```

Also inspect the node's launch logs in Jenkins.

---

# 11. Configure the Deployment Server

The deployment server is hosted in the AWS Default VPC and has an Elastic IP.

Connect from your local machine:

```bash
ssh -i deployment-key.pem ubuntu@DEPLOYMENT_ELASTIC_IP
```

All commands in this section run on the Deployment Server.

## 11.1 Update Ubuntu

```bash
sudo apt update

sudo apt upgrade -y
```

## 11.2 Install Docker Engine

Follow the same official Docker repository installation procedure used for the Docker agent.

Install:

```bash
sudo apt install -y \
    docker-ce \
    docker-ce-cli \
    containerd.io \
    docker-buildx-plugin \
    docker-compose-plugin
```

Enable Docker:

```bash
sudo systemctl enable docker

sudo systemctl start docker
```

Verify:

```bash
sudo docker run hello-world
```

## 11.3 Create a dedicated deployment user

Your pipeline uses:

```groovy
DEPLOY_USER = 'jenkins'
```

Create that user on the deployment server:

```bash
sudo adduser --disabled-password --gecos "" jenkins
```

Add the deployment user to the Docker group:

```bash
sudo usermod -aG docker jenkins
```

Verify:

```bash
groups jenkins
```

The output should include `docker`.

## 11.4 Create the application directory

```bash
sudo mkdir -p /opt/node-app

sudo chown jenkins:jenkins /opt/node-app

sudo chmod 750 /opt/node-app
```

This directory can later hold application-specific deployment configuration.

The current pipeline does not use an environment file. If one is introduced later, ensure its permissions are restricted.

---

# 12. Establish SSH Authentication: Docker Agent to Deployment Server

The Docker agent must authenticate with the deployment server.

The Jenkins pipeline uses:

```groovy
sshagent(credentials: ['deployement-instance-ssh'])
```

Therefore, we need a deployment SSH key stored in Jenkins Credentials.

## 12.1 Generate a separate deployment key

Generate this key on the Jenkins Controller as the Jenkins service user:

```bash
sudo -u jenkins ssh-keygen \
    -t ed25519 \
    -C "docker-agent-to-deployment-server" \
    -f /var/lib/jenkins/.ssh/deployment-key
```

Use a dedicated key rather than reusing the controller-to-agent key.

Display the public key:

```bash
sudo cat /var/lib/jenkins/.ssh/deployment-key.pub
```

Copy it.

## 12.2 Configure authorized_keys on the deployment server

On the Deployment Server:

```bash
sudo -u jenkins mkdir -p /home/jenkins/.ssh

sudo chmod 700 /home/jenkins/.ssh
```

Edit:

```bash
sudo nano /home/jenkins/.ssh/authorized_keys
```

Paste the public key.

Set permissions:

```bash
sudo chown -R jenkins:jenkins /home/jenkins/.ssh

sudo chmod 700 /home/jenkins/.ssh

sudo chmod 600 /home/jenkins/.ssh/authorized_keys
```

## 12.3 Configure known_hosts on the Docker agent

This step is important.

The deployment SSH command executes from the Docker agent, not from the Jenkins controller.

Therefore, the deployment server's host key must be trusted by the Docker agent.

On the Deployment Server, inspect its host key fingerprint:

```bash
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

On the Docker Agent, create the SSH directory:

```bash
sudo -u jenkins mkdir -p /home/jenkins/.ssh
```

Retrieve the host key:

```bash
sudo -u jenkins ssh-keyscan -H DEPLOYMENT_ELASTIC_IP \
>> /home/jenkins/.ssh/known_hosts
```

Verify the fingerprint before accepting the key.

Set permissions:

```bash
sudo chown -R jenkins:jenkins /home/jenkins/.ssh

sudo chmod 700 /home/jenkins/.ssh

sudo chmod 600 /home/jenkins/.ssh/known_hosts
```

## 12.4 Configure Jenkins deployment credentials

Navigate to:

Jenkins → Manage Jenkins → Credentials → System → Global credentials → Add Credentials.

Configuration:

```text
Kind: SSH Username with private key
Username: jenkins
Private Key: Enter directly
ID: deployement-instance-ssh
Description: Deployment EC2 SSH access
```

Paste:

```bash
sudo cat /var/lib/jenkins/.ssh/deployment-key
```

The credential ID must match the Jenkinsfile exactly.

## 12.5 Test the deployment connection

From the Docker Agent, test using the deployment private key through an authorized test path, or use the Jenkins credential during a pipeline test.

The expected SSH target is:

```text
jenkins@DEPLOYMENT_ELASTIC_IP
```

Verify that the deployment user can run Docker commands without sudo.

---

# 13. Configure Docker Hub

Create a Docker Hub repository.

Example:

```text
Repository: jenkins-aws-docker-lab
Visibility: Public
Username: YOUR_DOCKERHUB_USERNAME
```

For publishing, generate a Docker Hub access token.

Do not use your normal account password.

Navigate to:

Docker Hub → Account Settings → Personal Access Tokens.

Create a token with the permissions needed to push images.

## Add credentials to Jenkins

Navigate to:

Jenkins → Manage Jenkins → Credentials → System → Global credentials → Add Credentials.

Configuration:

```text
Kind: Username with password
Username: YOUR_DOCKERHUB_USERNAME
Password: DOCKER_HUB_ACCESS_TOKEN
ID: dockerhub
Description: Docker Hub image publishing
```

Your pipeline references:

```groovy
credentialsId: 'dockerhub'
```

The deployment server does not need Docker Hub authentication for the current public image repository.

For a private repository, configure pull authentication on the deployment server securely.

---

# 14. Configure GitHub Webhook

The GitHub webhook triggers the Jenkins pipeline whenever code is pushed.

## 14.1 Configure the Jenkins job

Create a new Jenkins job:

```text
New Item → Pipeline
```

Configure:

```text
Name: node-app-cicd
```

Under Pipeline:

```text
Definition: Pipeline script from SCM
SCM: Git
Repository URL: YOUR_GITHUB_REPOSITORY_URL
Branch: */main
Script Path: Jenkinsfile
```

Under Build Triggers, enable:

```text
GitHub hook trigger for GITScm polling
```

Save the job.

## 14.2 Configure GitHub Webhook

Open:

GitHub Repository → Settings → Webhooks → Add webhook.

Configuration:

```text
Payload URL:
http://YOUR_JENKINS_DOMAIN:8080/github-webhook/

Content type:
application/json

Events:
Just the push event
```

Save the webhook.

The trailing slash in `/github-webhook/` is important.

## 14.3 Verify webhook delivery

Open:

GitHub → Repository Settings → Webhooks.

Select your webhook.

Navigate to Recent Deliveries.

Push a commit:

```bash
git add .

git commit -m "Trigger Jenkins pipeline"

git push origin main
```

Check the delivery status.

A successful webhook delivery should return an HTTP success response.

Then verify that Jenkins automatically starts the job.

Note: This HTTP-based webhook configuration is for the current lab. Use HTTPS and a webhook secret when hardening the controller.

---

# 15. Configure the Jenkins Pipeline

Create a `Jenkinsfile` in the application repository.

The following pipeline implements the complete workflow.

Look at the `Jenkinsfile` in the repository for an code.

**Important:** This version uses verified SSH host keys, unlike the initial lab pipeline that used `StrictHostKeyChecking=no`. Make sure the deployment host key has been installed and verified on the Docker agent before running it.

The example retains the current deployment strategy; it does not implement automatic rollback or zero-downtime deployment.

## Application prerequisites

Your Node.js application must:

- Listen on port 3000.
- Listen on `0.0.0.0` inside the container.
- Include a working Dockerfile.
- Respond successfully to the health-check URL.

Example:

```javascript
app.listen(3000, '0.0.0.0', () => {
    console.log('Server running on port 3000');
});
```

If your application listens only on `localhost` inside the container, Docker port mapping will not make it accessible externally.

---

# 16. Execute and Verify CI/CD

Once the infrastructure and Jenkins job are configured, perform a complete test.

## 16.1 Verify the Docker agent

In Jenkins:

```text
Manage Jenkins → Nodes → docker-agent
```

Confirm:

```text
Agent Status: Online
Label: docker
```

## 16.2 Push application changes

On your local machine:

```bash
git add .

git commit -m "Test remote CI/CD deployment"

git push origin main
```

## 16.3 Observe Jenkins execution

Navigate to:

```text
Jenkins → node-app-cicd → Build History
```

The pipeline should execute:

1. Checkout.
2. Verify Agent.
3. Build Docker Image.
4. Run Container Test.
5. Push to Docker Hub.
6. Deploy to AWS EC2.

## 16.4 Verify Docker Hub

Open your Docker Hub repository.

Verify that both image tags exist:

```text
YOUR_USERNAME/jenkins-aws-docker-lab:BUILD_NUMBER

YOUR_USERNAME/jenkins-aws-docker-lab:latest
```

## 16.5 Verify the deployment server

Connect to the deployment server:

```bash
ssh ubuntu@DEPLOYMENT_ELASTIC_IP
```

Inspect running containers:

```bash
sudo docker ps
```

Inspect logs:

```bash
sudo docker logs node-cicd
```

Inspect the application:

```bash
curl -I http://localhost/
```

## 16.6 Verify externally

Open:

```text
http://DEPLOYMENT_ELASTIC_IP/
```

You should see your deployed Node.js application.

At this point, the complete remote CI/CD pipeline is operational.

---

# 17. Common Mistakes and Troubleshooting

This section covers common problems encountered while configuring remote Jenkins and Docker environments.

## 17.1 Jenkins service fails to start

**Symptoms:**

```text
jenkins.service failed
```

Check:

```bash
sudo systemctl status jenkins

sudo journalctl -u jenkins -n 100 --no-pager
```

Verify Java:

```bash
java -version
```

Common causes:

- Unsupported Java version.
- Port 8080 already in use.
- Incorrect Jenkins installation.
- Insufficient memory.

Check whether port 8080 is occupied:

```bash
sudo ss -lntp | grep 8080
```

## 17.2 Jenkins cannot connect to the private Docker agent

**Symptoms:**

```text
Connection timed out
```

Check:

1. Controller and agent are in the same VPC or have private routing.
2. Docker agent has a private IP.
3. Agent security group permits inbound TCP 22 from the controller security group.
4. SSH service is running.
5. Agent instance is running.

Test from the controller:

```bash
nc -zv DOCKER_AGENT_PRIVATE_IP 22
```

If the connection times out, investigate security groups and routing before changing SSH configuration.

## 17.3 SSH permission denied

**Symptoms:**

```text
Permission denied (publickey)
```

Check the agent's authorized keys:

```bash
sudo cat /home/jenkins/.ssh/authorized_keys
```

Verify permissions:

```bash
sudo chmod 700 /home/jenkins/.ssh

sudo chmod 600 /home/jenkins/.ssh/authorized_keys

sudo chown -R jenkins:jenkins /home/jenkins/.ssh
```

Verify the username.

The Jenkins agent connection must use:

```text
jenkins
```

Not:

```text
ubuntu
```

Also verify that the private key in Jenkins corresponds to the public key in `authorized_keys`.

## 17.4 Host key verification failed

**Symptoms:**

```text
Host key verification failed
```

Possible causes:

- Missing host key in `known_hosts`.
- Host key changed after rebuilding the EC2 instance.
- Wrong host key entry.
- Connecting through an unexpected hostname or IP.

Inspect:

```bash
ssh-keygen -F DOCKER_AGENT_PRIVATE_IP
```

For the deployment server, execute the same check on the Docker agent:

```bash
ssh-keygen -F DEPLOYMENT_ELASTIC_IP
```

Verify the new host fingerprint independently before updating the known_hosts entry.

Do not blindly delete or trust an unexpected host key.

## 17.5 Docker permission denied

**Symptoms:**

```text
permission denied while trying to connect to
the Docker daemon socket
```

Check:

```bash
groups jenkins

ls -l /var/run/docker.sock
```

Add the Jenkins user to the Docker group:

```bash
sudo usermod -aG docker jenkins
```

Restart the Jenkins agent session so that the new group membership takes effect.

Verify by running:

```bash
sudo -u jenkins docker ps
```

If the Jenkins agent is already running, disconnect and reconnect it.

## 17.6 Docker agent cannot access the internet

**Symptoms:**

```text
Could not resolve host
```

or:

```text
Connection timed out
```

Check the private route table:

```text
0.0.0.0/0 → NAT Gateway
```

Check:

- NAT Gateway status is Available.
- NAT Gateway is in a public subnet.
- Public subnet has a route to the Internet Gateway.
- NAT Gateway has an Elastic IP.
- Private agent has the correct route table.
- DNS resolution is enabled for the VPC.

Test from the Docker agent:

```bash
curl -I https://github.com
```

Test Docker Hub connectivity:

```bash
curl -I https://registry-1.docker.io/v2/
```

A `401 Unauthorized` response from the registry endpoint is acceptable for this connectivity test; it indicates the endpoint is reachable.

## 17.7 Deployment SSH connection times out

**Symptoms:**

```text
ssh: connect to host ... port 22: Connection timed out
```

Remember:

The SSH connection originates from the Docker agent, not the Jenkins controller.

Check the deployment server security group.

It should permit:

```text
Inbound TCP 22
Source: NAT_GATEWAY_ELASTIC_IP/32
```

Check:

- Deployment EC2 has an Elastic IP.
- Deployment EC2 is in a public subnet.
- Its route table points to the Internet Gateway.
- SSH service is running.
- Docker agent can reach the deployment public IP.

Test from the Docker agent:

```bash
nc -zv DEPLOYMENT_ELASTIC_IP 22
```

## 17.8 GitHub webhook does not trigger Jenkins

Check:

- Jenkins controller public IP.
- DuckDNS DNS resolution.
- Security group inbound port 8080.
- Webhook URL.
- GitHub plugin installation.
- Job build trigger configuration.

Correct webhook path:

```text
/github-webhook/
```

Check GitHub:

```text
Repository → Settings → Webhooks → Recent Deliveries
```

Check Jenkins:

```text
Job → Build History
```

Also verify that the webhook endpoint is reachable from outside the AWS VPC.

## 17.9 Docker image push fails

**Symptoms:**

```text
unauthorized: authentication required
```

Check:

- Docker Hub username.
- Access token.
- Jenkins credential ID.
- Username/password credential type.
- Repository name.
- Repository permissions.

The credential ID must match:

```groovy
credentialsId: 'dockerhub'
```

## 17.10 Application container starts but application is inaccessible

Inspect:

```bash
docker ps -a

docker logs node-cicd
```

Check application port mapping:

```text
Host port: 80
Container port: 3000
```

Verify the application listens on:

```text
0.0.0.0:3000
```

Check the deployment server security group permits inbound HTTP port 80.

## 17.11 Container replacement succeeds but the new application fails

The current deployment removes the existing container before starting the replacement.

If the new container exits, the application becomes unavailable.

Inspect:

```bash
docker ps -a

docker logs node-cicd
```

For production, implement:

- Image pull before stopping the existing application.
- Health-check validation after startup.
- Rollback to the previous image on failure.
- A deployment strategy that avoids unnecessary downtime.

## 17.12 Jenkins disk space warnings

Check disk usage:

```bash
df -h
```

Check Docker storage:

```bash
docker system df
```

Check memory:

```bash
free -h
```

Inspect Jenkins logs:

```bash
sudo journalctl -u jenkins -n 100 --no-pager
```

Remove unused Docker resources carefully:

```bash
docker image prune -f

docker container prune -f
```

Avoid running aggressive volume or system-wide prune commands without checking whether important data is stored there.

---

# 18. Security Recommendations

The current setup is a functional learning environment. Before using it for production workloads, implement the following improvements.

| Component | Recommended improvement |
|---|---|
| Jenkins access | Private access using VPN or SSM |
| Webhook | HTTPS and webhook secret validation |
| SSH | Host-key verification and restricted access |
| Jenkins credentials | AWS Secrets Manager |
| IAM | Dedicated least-privilege EC2 roles |
| Docker Hub | Access tokens or private ECR |
| Deployment | Health checks and automatic rollback |
| Application | HTTPS and TLS termination |
| Logging | CloudWatch integration |
| Monitoring | Instance, container and pipeline monitoring |

### Important Docker security consideration

Adding a user to the Docker group grants substantial privileges over the host.

Do not treat Docker group membership as an ordinary low-privilege permission.

### Important Jenkins security consideration

Jenkins administrators and users able to execute arbitrary pipeline code may effectively gain significant access to the infrastructure.

Use controlled repository access, credential scoping, and appropriate job permissions.

---

# 19. Infrastructure Verification Checklist

Use this checklist before considering the infrastructure operational.

### AWS Networking

- [ ] Jenkins VPC created.
- [ ] Public subnet configured.
- [ ] Private subnet configured.
- [ ] Internet Gateway attached.
- [ ] NAT Gateway available.
- [ ] Private route table points to NAT Gateway.
- [ ] Deployment EC2 hosted in Default VPC.
- [ ] Deployment EC2 has Elastic IP.
- [ ] Security groups configured correctly.

### Jenkins Controller

- [ ] Ubuntu updated.
- [ ] Java 21 installed.
- [ ] Jenkins LTS installed.
- [ ] Jenkins service running.
- [ ] Required plugins installed.
- [ ] DuckDNS configured.
- [ ] Administrator access working.

### Docker Agent

- [ ] Ubuntu updated.
- [ ] Java installed.
- [ ] SSH server running.
- [ ] Jenkins user created.
- [ ] Agent working directory created.
- [ ] Docker Engine installed.
- [ ] Jenkins user added to Docker group.
- [ ] SSH authentication configured.
- [ ] Agent registered in Jenkins.
- [ ] Agent status Online.
- [ ] NAT connectivity verified.

### Deployment Server

- [ ] Ubuntu updated.
- [ ] Docker Engine installed.
- [ ] Deployment user created.
- [ ] Docker permissions configured.
- [ ] SSH authentication configured.
- [ ] Deployment server reachable from Docker agent.
- [ ] Port 80 accessible externally.

### CI/CD Pipeline

- [ ] GitHub repository connected.
- [ ] GitHub webhook configured.
- [ ] Jenkins credentials configured.
- [ ] Docker Hub repository created.
- [ ] Docker image builds successfully.
- [ ] Container health check passes.
- [ ] Image pushed to Docker Hub.
- [ ] Deployment SSH connection succeeds.
- [ ] Application container starts successfully.
- [ ] Application accessible externally.

---

# 20. Future Improvements

After completing the initial implementation, the following improvements can evolve the environment into a production-oriented CI/CD platform.

### Phase 1: Secure Infrastructure

- Restrict direct public access to Jenkins.
- Implement VPN or Systems Manager access.
- Configure HTTPS for Jenkins.
- Enforce SSH host-key verification.
- Restrict security group rules.
- Implement webhook secret validation.

### Phase 2: Centralized Secrets Management

- Integrate AWS Secrets Manager.
- Introduce dedicated IAM instance roles.
- Retrieve application secrets at deployment time.
- Remove sensitive configuration from source code.
- Implement secret rotation.

### Phase 3: Registry and Docker Security

- Evaluate Amazon ECR.
- Implement image vulnerability scanning.
- Use immutable image tags.
- Configure secure Docker communication where remote daemon APIs are required.
- Restrict Docker daemon access.

### Phase 4: Reliable Deployment

- Implement application health checks.
- Introduce automatic rollback.
- Configure blue-green deployment.
- Add deployment validation.
- Separate deployment configuration from the application image.

### Phase 5: Monitoring

- Integrate CloudWatch.
- Centralize application logs.
- Monitor EC2 and Docker resources.
- Configure deployment notifications.
- Track Jenkins build failures.

---

# Conclusion

Following this guide, you should be able to build a distributed AWS CI/CD infrastructure consisting of:

- A Jenkins controller in a public subnet.
- A dedicated Docker build agent in a private subnet.
- NAT Gateway-based outbound internet connectivity.
- A separate EC2 deployment server in the AWS Default VPC.
- SSH-based authentication between infrastructure components.
- GitHub Webhook-triggered pipeline execution.
- Docker Hub image publishing.
- Automated Docker container deployment.

The resulting architecture provides a foundation for understanding and implementing remote CI/CD workflows using AWS, Jenkins, Docker, and Linux.

**The objective is not just to get a Jenkins pipeline working, but to understand how the infrastructure, networking, authentication, build environment, registry, and deployment environment work together.**