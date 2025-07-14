# Red Team Infrastructure (Terraform + Ansible + AWS + Cloudflare)

This repo contains a fully Terraform-managed Ansible-configured AWS deployment for red team operations. It supports standing up infrastructure for Cloudflare redirectors, C2 servers, bastion hosts, and more — all reproducible, configurable, and automated. Full walkthrough on my blog: https://www.cyberforks.com/automating-red-team-infrastructure

---

## 🧩 Components

- **Redirector**: For traffic forwarding and domain fronting
- **C2 Server**: Beacon receiver and staging infrastructure (e.g., Mythic, Cobalt Strike, etc.)
- **Bastion Host**: Secure SSH gateway for operational access
- **Security Groups**: Locked down with configurable ingress rules
- **VPC + Subnets + Route Tables**: Isolated networking for red team infrastructure
- **SSH Key**: Private keys created and deployed into EC2 for use

---

## Pre-requisites
```bash
- sudo apt update
- sudo apt install terraform, ansible, awscli, cloudflared
```
- An Attack Box (operator box e.g., Kali Linux)
- An AWS account where you can create Access Keys to configure AWS CLI
- A Cloudflare account where you can create API keys


## 🚀 Deployment Instructions

```bash
# Step 0
# Clone the repo.
git clone git@github.com:rbfp/redteam-infra.git
mv redteam-infra <project_name> && cd <project_name>

# Configure your AWS CLI profile.
## [in web browser] Log into AWS console w/ permissions to create Access Keys and create a user to have PowerUser permissions.
aws configure

# Configure your Cloudflare setup.
## [in web browser] Purchase a hostname from Cloudflare to be used in Cloudflare.
## [in web browser] Create an API Token with these settings (save this API token):
## - Permission: Account, Cloudflare One Networks, Edit
## - Permission: Account, Cloudflare One Connector: cloudflared, Edit
## - Permission: Account, Load Blancing: Monitors and Pools, Edit
## - Permission: Zone, DNS, Edit
## - Zone Resources: Include, Specific zone, <pruchased_domain>
cloudflared login

# Step 1
# Run through the setup script to deploy to a Region you select
# Note the project name you pick
# Creates: (Resource) - (Username) - (Key to use)
# Bastion Host - bastion - {project_name}-bastion.pem
# Redirector - redirector - internal.pem
# C2 Server - c2server - internal.pem
# The [Redirector] and [C2 Server] can only be ssh-ed from w/n the [Bastion Host]
./1_infra_setup.sh

# Step 2
# Verify your AWS resources are running with manage_aws.sh > option 2 > [type your region]
## Cannot ssh via IPv6 from within my enterprise.
./0_manage_aws.sh

# Step 3
# Set up Cloudflare redirector
## [in web browser] Download cloudflared.deb from Cloudflare website
## [in web browser] Create S3 Bucket to upload Cloudflare files
## - Permission: Unlock public access
## - Object Policy: {"Version": "2012-10-17", "Statement":[{"Sid": "AllowPublicRead","Effect": "Allow","Principal": "*","Action": "s3:GetObject","Resource": "arn:aws:s3:::<bucket_name>/*"}]}
## [in local device] install cloudflared and login
## [in local device] after successfuly login, you're computer will have a ~/.cloudflared/cert.pem file
## [in web browser] Upload cloudflared.deb and ~/.cloudflared/cert.pem
## [in bastion host] Create cloudflare_setup.sh
vim cloudflare_setup.sh
## [in bastion host] Copy and paste the script below into vim and save
## [+] ----- START SCRIPT ----- [+]
#!/bin/bash
set -euo pipefail
CF_API_TOKEN="<CF_API_TOKEN>"
TUNNEL_NAME="<project_name>-c2-tunnel"
TUNNEL_DOMAIN="<purchased_domain>"
CLOUDFLARED_DIR="/home/redirector/.cloudflared"
mkdir /home/redirector/.cloudflared
mv /home/redirector/cert.pem /home/redirector/.cloudflared/
cloudflared tunnel create $TUNNEL_NAME
TUNNEL_ID=$(basename "$CLOUDFLARED_DIR"/*.json)
TUNNEL_ID="${TUNNEL_ID%.json}"
echo "Tunnel ID: $TUNNEL_ID"
cat << EOF2 > $CLOUDFLARED_DIR/config.yml
tunnel: $TUNNEL_ID
credentials-file: $CLOUDFLARED_DIR/$TUNNEL_ID.json
protocol: http2
no-autoupdate: true
edge-ip-version: 6
ingress:
  - hostname: $TUNNEL_DOMAIN
    service: https://localhost:443
    originRequest:
      noTLSVerify: true
  - service: http_status:404
EOF2
## [+] ----- END SCRIPT ----- [+]
## [in web browser] Copy AWS S3 URLs (e.g. https://<bucketname>.s3.<region>.amazonaws.com/cert.pem)
## [in bastion host] Edit URLs to put "dualstack" between "s3" and "us-east-1" (or w/e region) (e.g. https://bucketname.s3.dualstack.us-east-1.amazonaws.com/cert.pem)
## [in bastion host] Download the files on Bastion host
## [in bastion host] Change the permission of the script and pem
chmod +x cloudflare_setup.sh
chmod +x cert.pem
## [in bastion host] Copy files to redirector (your cloudflare.deb filename may be different depending on arch & ver)
scp -i .ssh/internal.pem cloudflared.deb cloudflare_setup.sh cert.pem redirector@<redirector_ip>:~
ssh -I .ssh/internal.pem redirector@<redirector_ip> "sudo dpkg -i cloudflared.deb && ./cloudflare_setup.sh"
## [in bastion host] run cloudflared to activate your Cloudflare tunnel
ssh -I .ssh/internal.pem redirector@<redirector_ip>
ssh -i .ssh/internal.pem redirector@10.0.15.60 "cloudflared --no-autoupdate --protocol http2 --edge-ip-version 6 tunnel run 'soc-east-1-c2-tunnel'" &
## [in web browser] Change all AWS S3 buckets to deny public access
## - Permission: Deny public access

# Step 4 (Cobalt Strike not currently in use, will be using Metasploit)
# Set up Cobalt Strike or C2 of choice, this will also configure your redirector for CS
# PreRequisites:
## 1) Host your own CobaltStrike file and replace the link in playbook.yml with your link
## 2) Should be a 7z file
## 3) Password protect your public Cobaltstrike 7z file
./3_cobalt_setup.sh
mkdir openjdk-11-jdk openjdk-11-jre c2-dependas re-dependas
cd /home/bastion/openjdk-11-jdk && apt-get download $(apt-cache depends --recurse --no-recommends --no-suggests --no-conflicts --no-breaks --no-replaces --no-enhances --no-pre-depends openjdk-11-jdk | grep '^\w')

# Step 5
# Run Cobalt Strike Server 
# ssh to Bastion then ssh to C2 Server
# Obtain/create Malleable C2 Profile
# You pick the {{ cobalt_server_pass }}
# Leave terminal open or CS server will shutdown
# In the C2 Server run:
chmod +x ~/CS491/Server/teamserver ~/CS491/Server/TeamServerImage
cd ~/CS491/Server/
sudo ./teamserver {{ c2_ipv4 }} {{ cobalt_server_pass }} {{ c2_profile }}

# Set up ssh proxy to connect your attack box to your c2 server
# Leave open or your CS client won't talk to your CS server
# On your attack box, open a new terminal and run:
ssh -i {{ project_name }}-bastion.pem -L 50050:{{ c2_ipv4 }}:50050 bastion@{{bastion-ipv6}}

# Redirect IPv6 traffic hitting Redirector to C2 Server
# Leave terminal open or your beacons won't talk to your CS server
# On your attack box, open a new terminal
# ssh to bastion then ssh to redirector
# In your Redirector run:
sudo socat TCP6-LISTEN:443,reuseaddr,fork TCP4:{{ c2_ipv4}}:443

# Run Cloudflare Tunneling to mask your IPv6 address
# Leave terminal open or your Cloudflare tunnel will close
# On your attack box, open a new terminal
# ssh into your Bastion then ssh into your Redirector
# In your Redirector run:
cloudflared --no-autoupdate --protocol http2 --edge-ip-version 6 tunnel run "{{ project_name }}-c2-tunnel"

# Download & extract the CobaltStrike 7zip onto your Attack Box then run:
chmod +x ~/CS491/Client/cobaltstrike-client.sh

# Open a new terminal and run:
./CS491/Client/cobaltstrike-client.sh
## Fill in:
## Alias: rbfp@{{ project_name }}
## Host: 127.0.0.1
## Port: 50050
## User: rbfp
## Password: {{ cobalt_server_pass }}
```
## Troubleshooting
### Cloudflare error 1033 when navigating to your domain
```bash
# Check the domain's DNS record, update the tunnel in the DNS record to an Active tunnel. 
# The target should be something like <tunnel_id>.cfargotunnel.com
```
### Cloudflare error 1016 when navigating to your domain
```bash
# Add cname record as indicated on DNS page. Should go to <TUNNEL_ID>.cfargotunnel.com
```
