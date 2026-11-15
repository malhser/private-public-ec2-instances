# AWS NAT Gateway – Private Subnet Internet Access (RCA)

## 1\. Project Overview

### Scenario

A junior cloud engineer launched an EC2 instance inside a private subnet for security purposes. However, when attempting to update packages or download software using commands such as:

``` bash
sudo yum update -y
```

the command timed out because the private instance did not have a route to the internet.

As a Senior Cloud Engineer, the task was to investigate the VPC architecture, identify the missing routing component, implement the solution, and verify internet connectivity.

### Project Objective

- Create a custom VPC.
- Create public and private subnets.
- Configure an Internet Gateway.
- Launch a bastion host in the public subnet.
- Launch an EC2 instance in the private subnet without a public IP.
- Reproduce the internet connectivity failure.
- Deploy a NAT Gateway in the public subnet.
- Update the private subnet route table.
- Verify outbound internet connectivity.
- Document the Root Cause Analysis.

## 2\. AWS Architecture

### Before Fix – Broken Architecture

``` text
                  Internet
                     |
                     |
              Internet Gateway
                     |
                Public Subnet
                     |
                Bastion Host
                     |
                     | SSH
                     |
                Private Subnet
                     |
                Private EC2
                     |
               No NAT Gateway
                     |
              Internet Access
                   FAILED
```

### After Fix – Correct Architecture

``` text
                       Internet
                          |
                          |
                  Internet Gateway
                          |
                    Public Subnet
                          |
                 -------------------
                 |                 |
             Bastion Host      NAT Gateway
                                   |
                              Elastic IP
                                   |
                                   |
                              Private Subnet
                                   |
                               Private EC2
                                   |
                            Outbound Internet
                                 WORKING
```

## 3\. AWS Services Used

| AWS Service | Purpose |
| --- | --- |
| Amazon VPC | Isolated network environment |
| Public Subnet | Hosts internet-facing resources |
| Private Subnet | Hosts private EC2 resources |
| Internet Gateway | Provides internet connectivity to public subnet resources |
| NAT Gateway | Allows private instances to initiate outbound internet connections |
| Elastic IP | Public IP assigned to the NAT Gateway |
| Amazon EC2 | Hosts the bastion and private instances |
| Route Tables | Controls network traffic routing |
| Security Groups | Controls instance-level network access |

## 4\. Infrastructure Configuration

| Component | Configuration |
| --- | --- |
| AWS Region | ap-south-1 (Mumbai) |
| VPC | Custom VPC |
| Public Subnet | Public subnet in selected AZ |
| Private Subnet | Private subnet in selected AZ |
| Internet Gateway | Attached to custom VPC |
| NAT Gateway | Deployed in public subnet |
| Elastic IP | Associated with NAT Gateway |
| Bastion Host | EC2 in public subnet |
| Private EC2 | EC2 in private subnet |
| Private EC2 Public IP | Disabled |

## 5\. Step 1: Create VPC and Subnets

Created a custom VPC with an IPv4 CIDR block.

Example:

``` text
VPC CIDR: 10.0.0.0/16

Public Subnet:  10.0.1.0/24
Private Subnet: 10.0.2.0/24
```

Created two subnets inside the VPC:

1. Public subnet – for the bastion host and NAT Gateway.
2. Private subnet – for the private EC2 instance.

## 6\. Step 2: Configure Internet Gateway

Created an Internet Gateway and attached it to the custom VPC.

### Public Route Table

Configured the public subnet route table:

| Destination | Target |
| --- | --- |
| 10.0.0.0/16 | local |
| 0.0.0.0/0 | Internet Gateway |

Associated the public route table with the public subnet.

The Internet Gateway provides internet connectivity to resources with appropriate public IPv4 addressing and security rules.

## 7\. Step 3: Launch EC2 Instances

### Bastion Host

Launched an EC2 instance in the public subnet.

Configuration:

- Instance name: Bastion-Host
- Subnet: Public Subnet
- Auto-assign Public IP: Enabled
- Security Group: Allow SSH from my trusted public IP address.

### Private EC2 Instance

Launched another EC2 instance in the private subnet.

Configuration:

- Instance name: Private-EC2
- Subnet: Private Subnet
- Auto-assign Public IP: Disabled
- Security Group: Allow SSH from the bastion host security group.

Connected to the private instance through the bastion host using SSH.

## 8\. Step 4: Reproduce the Connectivity Issue

Before deploying the NAT Gateway, checked internet connectivity from the private EC2 instance.

Commands:

``` bash
ping google.com
```

``` bash
curl -I https://google.com
```

Attempted package updates:

``` bash
sudo yum update -y
```

### Observed Result

The private instance could not access the internet because its route table contained only the local route.

### Private Route Table Before Fix

| Destination | Target |
| --- | --- |
| 10.0.0.0/16 | local |

There was no default route for internet-bound traffic.

## 9\. Step 5: Root Cause Analysis (RCA)

### Root Cause

The private EC2 instance could not access the internet because the private subnet route table did not contain a default route to a NAT Gateway.

The private instance had no public IP address and could not directly use the Internet Gateway for outbound internet access.

### Why Does a Private Subnet Require a NAT Gateway?

An Internet Gateway and a NAT Gateway serve different purposes.

**Internet Gateway (IGW):**

- Provides internet connectivity for resources with public IP addresses and appropriate routing and security configuration.
- Supports inbound and outbound internet communication.
- Commonly used by resources in public subnets.

**NAT Gateway:**

- Allows instances in private subnets to initiate outbound connections to the internet.
- Translates private source IP addresses to the NAT Gateway's public Elastic IP.
- Allows return traffic for connections initiated by private instances.
- Does not allow unsolicited inbound internet connections to private instances.

Therefore, a NAT Gateway provides outbound internet access while keeping private EC2 instances without public IP addresses.

## 10\. Step 6: Implement the Fix – Create NAT Gateway

Created a NAT Gateway with the following configuration:

- Name: NAT-Gateway
- Subnet: Public Subnet
- Connectivity Type: Public
- Elastic IP: Allocated and associated

**Important:** The NAT Gateway must be deployed in a public subnet with a route to the Internet Gateway.

Waited until the NAT Gateway status changed to Available.

## 11\. Step 7: Update Private Route Table

Navigated to:

AWS Console → VPC → Route Tables → Private Route Table → Routes → Edit Routes.

Added the following route:

| Destination | Target |
| --- | --- |
| 10.0.0.0/16 | local |
| 0.0.0.0/0 | NAT Gateway |

Associated the private route table with the private subnet.

### Traffic Flow After Fix

1. Private EC2 sends an internet-bound request.
2. Private route table forwards the request to the NAT Gateway.
3. NAT Gateway translates the private source IP to its Elastic IP.
4. Internet Gateway forwards the request to the internet.
5. Response traffic returns through the NAT Gateway.
6. NAT Gateway forwards the response to the private EC2 instance.

## 12\. Step 8: Verify Internet Connectivity

Connected to the private EC2 instance through the bastion host.

Executed the following commands:

### Test 1: DNS Resolution

``` bash
nslookup google.com
```

### Test 2: Ping

``` bash
ping -c 4 google.com
```

### Test 3: HTTPS Connectivity

``` bash
curl -I https://google.com
```

### Test 4: Package Update

For Amazon Linux 2023:

``` bash
sudo dnf update -y
```

For older Amazon Linux or compatible systems:

``` bash
sudo yum update -y
```

### Expected Result

- DNS resolution succeeds.
- HTTPS requests receive a response.
- Package repositories become reachable.
- The private EC2 instance can download software without a public IP address.

Note: Ping may fail if ICMP traffic is blocked. Use `curl` and package manager tests as the primary proof of internet connectivity.

## 13\. RCA Summary

| Category | Details |
| --- | --- |
| Issue | Private EC2 could not download software or update packages |
| Impact | Package installation and software updates failed |
| Root Cause | Missing default route to NAT Gateway |
| Investigation | Reviewed the private subnet route table |
| Resolution | Created a public NAT Gateway with an Elastic IP |
| Routing Fix | Added 0.0.0.0/0 route to NAT Gateway |
| Validation | Verified DNS, HTTPS and package manager connectivity |
| Final Status | Private instance outbound internet access restored |

## 14\. Screenshots for GitHub Submission

Create a folder called `screenshots/` and add your actual AWS Console and terminal screenshots.

| Screenshot | Description |
| --- | --- |
| 01-vpc.png | Custom VPC configuration |
| 02-subnets.png | Public and private subnet details |
| 03-internet-gateway.png | IGW attached to VPC |
| 04-public-route-table.png | Public route table with IGW |
| 05-private-ec2.png | Private EC2 instance without public IP |
| 06-broken-connectivity.png | Failed internet connectivity before NAT |
| 07-nat-gateway.png | NAT Gateway in public subnet |
| 08-private-route-table-fixed.png | Default route pointing to NAT Gateway |
| 09-connectivity-success.png | Successful curl or package update |
| 10-architecture.png | Final AWS architecture |

## 15\. Final Outcome

Successfully identified and resolved the private subnet internet connectivity issue.

The final architecture includes:

- Custom VPC with public and private subnets.
- Internet Gateway attached to the VPC.
- Bastion host in the public subnet.
- Private EC2 instance without a public IP.
- NAT Gateway deployed in the public subnet.
- Elastic IP associated with NAT Gateway.
- Private route table configured with a default route to NAT Gateway.
- Successful outbound internet connectivity from the private EC2 instance.

## 16\. Cost Optimization and Cleanup

NAT Gateways incur hourly charges and data processing charges. Elastic IP and other AWS resources may also incur costs depending on usage and current AWS pricing.

After completing the assignment:

1. Terminate the EC2 instances if no longer required.
2. Delete the NAT Gateway.
3. Release the Elastic IP after the NAT Gateway is deleted and the address is no longer needed.
4. Detach and delete the Internet Gateway.
5. Delete the custom subnets.
6. Delete the custom VPC after all dependent resources are removed.

Screen shots

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/52821600-5e66-4e78-8726-e949c58a89a1" />

```
