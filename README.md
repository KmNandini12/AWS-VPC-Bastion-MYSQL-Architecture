# AWS VPC with Public/Private Subnets and MySQL Database


## Project Overview
This project demonstrates a secure, production-ready AWS VPC architecture implementing a two-tier application model. It features a public subnet hosting a bastion host and a private subnet containing a MySQL database server, connected via a NAT Gateway for controlled internet access.

## Key Objectives
* Design and implement a secure VPC architecture following AWS best practices
* Isolate sensitive database resources in a private subnet without public internet exposure
* Enable secure administrative access through a bastion host
* Configure network routing, security groups, and NAT Gateway for controlled egress traffic
* Implement MySQL database with remote access capabilities from authorized instances

## Architecture
![Detailed Architecture Diagram]( https://github.com/KmNandini12/AWS-VPC-Bastion-MYSQL-Architecture/blob/d1ecde8690ad1e5ef493c52d03d76b93249e4642/Architecture.png)


### Components

| Component | Purpose | CIDR Block | Key Features |
| :--- | :--- | :--- | :--- |
| VPC | Main network container | 10.0.0.0/16 | Isolated network environment |
| Public Subnet | Bastion host deployment | 10.0.1.0/24 | Internet-facing, hosts jump server |
| Private Subnet | Database isolation | 10.0.2.0/24 | No internet ingress, NAT egress only |
| Internet Gateway | Internet connectivity | N/A | Enables public subnet internet access |
| NAT Gateway | Private subnet egress | N/A | Allows outbound traffic from private instances |
| Bastion Host | Secure access point | EC2 in Public Subnet | SSH access control, MySQL client |
| MySQL Server | Database server | EC2 in Private Subnet | Isolated database instance |

## Implementation Steps

### Phase 1: Infrastructure Setup

**1. VPC Creation**
* VPC CIDR: 10.0.0.0/16
* Region: us-east-1
* Tenancy: Default

![VPC Dashboard]( https://github.com/KmNandini12/AWS-VPC-Bastion-MYSQL-Architecture/blob/d1ecde8690ad1e5ef493c52d03d76b93249e4642/VPC%20Dashboard.png)
*VPC dashboard showing created resources*

**2. Subnet Configuration**

| Subnet | Type | AZ | CIDR | Auto-assign IP |
| :--- | :--- | :--- | :--- | :--- |
| Public-Subnet | Public | us-east-1a | 10.0.1.0/24 | Enabled |
| Private-Subnet | Private | us-east-1a | 10.0.2.0/24 | Disabled |

![Subnets View]( https://github.com/KmNandini12/AWS-VPC-Bastion-MYSQL-Architecture/blob/5fc53d726e255ed77cd204d8f844bf341501e1a6/Private%20Subnet.png)
*Subnet configuration showing private segregation*

**3. Gateway Setup**
* Internet Gateway: Attached to VPC for public subnet internet access
* NAT Gateway: Placed in public subnet with Elastic IP for private subnet egress

![NAT Gateway]( https://github.com/KmNandini12/AWS-VPC-Bastion-MYSQL-Architecture/blob/5fc53d726e255ed77cd204d8f844bf341501e1a6/NAT%20Gateway.png
)
*NAT Gateway showing "Available" status in public subnet*

**4. Route Tables**
* Public Route Table: Destination: 0.0.0.0/0 -> Target: Internet Gateway (Associated with Public-Subnet)
* Private Route Table: Destination: 0.0.0.0/0 -> Target: NAT Gateway (Associated with Private-Subnet)

![Route Tables]( https://github.com/KmNandini12/AWS-VPC-Bastion-MYSQL-Architecture/blob/5fc53d726e255ed77cd204d8f844bf341501e1a6/Public%20Route%20Table.png)
![Route Tables]( https://github.com/KmNandini12/AWS-VPC-Bastion-MYSQL-Architecture/blob/5fc53d726e255ed77cd204d8f844bf341501e1a6/Private%20Route%20Table.png)
*Route tables showing IGW and NAT Gateway targets*

**5. Security Groups**
* Public Instance: Inbound: SSH (22) from trusted IPs | Outbound: All traffic
* Private Instance (MYSQL Server): Inbound: SSH (22) and MySQL (3306) from VPC CIDR | Outbound: All traffic

![Security Groups]( https://github.com/KmNandini12/AWS-VPC-Bastion-MYSQL-Architecture/blob/5fc53d726e255ed77cd204d8f844bf341501e1a6/Private%20Security%20Group.png)
*Security group rules ensuring least-privilege access*

### Phase 2: EC2 Instance Deployment

**6. Instance Configuration**

| Instance | Subnet | Public IP | Security Group | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| Bastion-Host | Public-Subnet | Enabled | Public-SG | SSH jump host, MySQL client |
| MySQL-Server | Private-Subnet | Disabled | Private-SG | Isolated database server |

![EC2 Instances]( https://github.com/KmNandini12/AWS-VPC-Bastion-MYSQL-Architecture/blob/5fc53d726e255ed77cd204d8f844bf341501e1a6/Public%20Server.png)
![EC2 Instances]( https://github.com/KmNandini12/AWS-VPC-Bastion-MYSQL-Architecture/blob/5fc53d726e255ed77cd204d8f844bf341501e1a6/MYSQL%20Server(private).png)
*EC2 instances showing public/private IP configuration*

### Phase 3: Database Configuration

**7. MySQL Installation on Private Instance**
```bash
# Connect to private instance via bastion
ssh -i key.pem ubuntu@<BASTION-PUBLIC-IP>
ssh -i key.pem ubuntu@<PRIVATE-IP-ADDRESS>

# Install MySQL Server
sudo apt update  # Demonstrates NAT Gateway functionality
sudo apt install mysql-server -y

# Configure MySQL for remote access, chnaging bind addr 127.0.0.1 --> 10.0.2.139(Private server IP Addr)
sudo sed -i 's/127.0.0.1/10.0.2..139/g' /etc/mysql/mysql.conf.d/mysqld.cnf
sudo systemctl restart mysql
```
![Enter private Server](https://github.com/KmNandini12/AWS-VPC-Bastion-MYSQL-Architecture/blob/5fc53d726e255ed77cd204d8f844bf341501e1a6/Accessing%20SQL%20Server%20via%20Bastion%20Server.png)
*ssh into Private server via Bastion server*

# MySQL Installation and Configuration on Private Instance

## 8. Database and User Setup

```sql

CREATE DATABASE company_db;
CREATE USER 'remote_admin'@'%' IDENTIFIED BY 'your_password';
GRANT ALL PRIVILEGES ON company_db.* TO 'remote_admin'@'%';
GRANT CREATE, ALTER, DROP, INSERT, UPDATE, DELETE, SELECT, REFERENCES, RELOAD ON *.* TO 'remote_admin'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```
![MySQL Installation]( https://github.com/KmNandini12/AWS-VPC-Bastion-MYSQL-Architecture/blob/79e39fa46a223f213133fbcf2ec864002c64cb74/Creation%20of%20DB%20and%20USER%20via%20Private%20Server%20.png)
*Create database and remote user*

## Phase 4: Connectivity Testing

### 9. NAT Gateway Verification

```bash
# From private instance (no public IP)
curl ifconfig.me # Returns NAT Gateway's public IP
```
![NAT Gateway verification](https://github.com/KmNandini12/AWS-VPC-Bastion-MYSQL-Architecture/blob/7813a05af98876371481f86f6781885b5c9fe431/Success%20NAT%20Connection.jpeg)
*Showing success NAT connection and*
*curl command showing ip of public server*
### 10. Database Connectivity Test

```bash
# From bastion host
mysql -h 10.0.2.139 -u remote_admin -p
Enter password.

-- Inside MySQL
SHOW DATABASES;
USE company_db;
SELECT User, Host FROM mysql.user;
```
![SQL Access](https://github.com/KmNandini12/AWS-VPC-Bastion-MYSQL-Architecture/blob/79e39fa46a223f213133fbcf2ec864002c64cb74/Accessing%20MYSQL%20via%20Public%20Server(Bastion%20Host).png)
## Key Features Demonstrated

### Security
* Network Isolation: Database in private subnet with no public IP
* Least Privilege: Security groups restrict access to necessary ports only
* Secure Access: SSH access only through bastion host from trusted IPs
* Encrypted Communication: MySQL connections within VPC

### Networking
* NAT Gateway: Enables private instances to initiate outbound connections
* Route Table Management: Controlled traffic flow between subnets
* CIDR Planning: Proper IP address allocation and subnet sizing
* AZ Awareness: All resources deployed in same Availability Zone

### Database Management
* Remote Configuration: MySQL configured for VPC-wide access
* User Management: Principle of least privilege for database users
* Service Management: Systemd control of MySQL service
* Configuration Management: Bind address modification for network access

## Technical Challenges and Solutions

**Challenge 1: Private Instance Internet Access**
* **Problem:** MySQL server needs apt update but has no public IP
* **Solution:** NAT Gateway in public subnet provides outbound internet access

**Challenge 2: Secure Database Access**
* **Problem:** Need remote MySQL access without exposing to public internet
* **Solution:** Security group rules allowing MySQL port 3306 only from VPC CIDR

**Challenge 3: Bastion Key Management**
* **Problem:** SSH key needs to be on bastion to access private instance
* **Solution:** SCP transfer with proper permissions (chmod 400)

## Performance and Cost Optimization

**Cost Considerations**
* NAT Gateway: ~$0.045/hour + data processing charges
* EIP: Free when attached to running instance
* t2.micro: Free tier eligible for both instances
* Cleanup: Important to release resources to avoid unnecessary charges

**Performance Notes**
* NAT Gateway provides up to 45 Gbps bandwidth
* Same AZ deployment minimizes latency
* Security group rules processed at instance level (low latency)

## Testing Methodology

**Connectivity Tests:**
* Bastion -> Internet (via IGW)
* Private Instance -> Internet (via NAT)
* Bastion -> Private Instance (SSH)
* Bastion -> MySQL (Port 3306)

**Security Tests:**
* Attempt direct SSH to private instance (should fail)
* Attempt MySQL from unauthorized IP (should fail)
* Verify no public IP on private instance

**Functionality Tests:**
* Database creation and access
* User privilege verification
* Service restart and persistence

## Cleanup Procedure

```bash
# Order of operations to avoid dependency errors
1. Terminate EC2 instances
2. Delete NAT Gateway
3. Release Elastic IP (if not auto-released)
4. Detach and delete Internet Gateway
5. Delete custom Security Groups
6. Delete Subnets
7. Delete custom Route Tables
8. Delete VPC
```
## Learning Outcomes

### AWS Services Mastered
* VPC: Virtual Private Cloud creation and management
* Subnets: Public/private subnet design and implementation
* Route Tables: Traffic routing and gateway associations
* Security Groups: Instance-level firewall configuration
* NAT Gateway: Outbound internet access for private resources
* EC2: Instance deployment and management

### DevOps Skills Demonstrated
* Infrastructure as Code principles
* Network security best practices
* Database server configuration
* SSH tunneling and secure access patterns
* Troubleshooting network connectivity

### Security Best Practices
* Defense in depth through subnet isolation
* Principle of least privilege in security groups
* Secure credential management
* Controlled internet egress via NAT

## Useful Resources
* [AWS VPC Documentation](https://docs.aws.amazon.com/vpc/)
* [MySQL Remote Access Configuration](https://dev.mysql.com/doc/refman/8.0/en/remote-administrative-connection.html)
* [SSH Agent Forwarding Guide](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/using-ssh-agent-forwarding)
* [NAT Gateway Pricing](https://aws.amazon.com/vpc/pricing/)

## Author
**Kumari Nandini**
[LinkedIn Profile](https://www.linkedin.com/in/kumari-nandini/) | [GitHub Profile](https://github.com/KmNandini12)

