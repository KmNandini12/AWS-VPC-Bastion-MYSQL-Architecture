# AWS VPC with Public/Private Subnets and MySQL Database

![AWS VPC Architecture Diagram](https://via.placeholder.com/800x400/0D1117/FFFFFF?text=AWS+VPC+Architecture+Diagram)
[Replace with your Lucidchart diagram link]

## Project Overview
This project demonstrates a secure, production-ready AWS VPC architecture implementing a two-tier application model. It features a public subnet hosting a bastion host and a private subnet containing a MySQL database server, connected via a NAT Gateway for controlled internet access.

## Key Objectives
* Design and implement a secure VPC architecture following AWS best practices
* Isolate sensitive database resources in a private subnet without public internet exposure
* Enable secure administrative access through a bastion host
* Configure network routing, security groups, and NAT Gateway for controlled egress traffic
* Implement MySQL database with remote access capabilities from authorized instances

## Architecture
![Detailed Architecture Diagram](https://via.placeholder.com/1000x500/0D1117/FFFFFF?text=Detailed+Architecture+Diagram)
[Replace with your detailed Lucidchart diagram]

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

![VPC Dashboard](https://screenshots/vpc-dashboard.png)
*VPC dashboard showing created resources*

**2. Subnet Configuration**

| Subnet | Type | AZ | CIDR | Auto-assign IP |
| :--- | :--- | :--- | :--- | :--- |
| Public-Subnet | Public | us-east-1a | 10.0.1.0/24 | Enabled |
| Private-Subnet | Private | us-east-1a | 10.0.2.0/24 | Disabled |

![Subnets View](https://screenshots/subnets-view.png)
*Subnet configuration showing public/private segregation*

**3. Gateway Setup**
* Internet Gateway: Attached to VPC for public subnet internet access
* NAT Gateway: Placed in public subnet with Elastic IP for private subnet egress

![NAT Gateway](https://screenshots/nat-gateway.png)
*NAT Gateway showing "Available" status in public subnet*

**4. Route Tables**
* Public Route Table: Destination: 0.0.0.0/0 -> Target: Internet Gateway (Associated with Public-Subnet)
* Private Route Table: Destination: 0.0.0.0/0 -> Target: NAT Gateway (Associated with Private-Subnet)

![Route Tables](https://screenshots/route-tables.png)
*Route tables showing IGW and NAT Gateway targets*

**5. Security Groups**
* Bastion-SG (Public Instance): Inbound: SSH (22) from trusted IPs | Outbound: All traffic
* MySQL-SG (Private Instance): Inbound: SSH (22) and MySQL (3306) from VPC CIDR | Outbound: All traffic

![Security Groups](https://screenshots/security-groups.png)
*Security group rules ensuring least-privilege access*

### Phase 2: EC2 Instance Deployment

**6. Instance Configuration**

| Instance | Subnet | Public IP | Security Group | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| Bastion-Host | Public-Subnet | Enabled | Bastion-SG | SSH jump host, MySQL client |
| MySQL-Server | Private-Subnet | Disabled | MySQL-SG | Isolated database server |

![EC2 Instances](https://screenshots/ec2-instances.png)
*EC2 instances showing public/private IP configuration*

### Phase 3: Database Configuration

**7. MySQL Installation on Private Instance**
```bash
# Connect to private instance via bastion
ssh -i key.pem ubuntu@<BASTION-PUBLIC-IP>
ssh -i key.pem ubuntu@10.0.2.15

# Install MySQL Server
sudo apt update  # Demonstrates NAT Gateway functionality
sudo apt install mysql-server -y

# Configure MySQL for remote access
sudo sed -i 's/127.0.0.1/0.0.0.0/g' /etc/mysql/mysql.conf.d/mysqld.cnf
sudo systemctl restart mysql
```

# MySQL Installation and Configuration on Private Instance

![MySQL Installation](https://screenshots/mysql-installation.png)

## 8. Database and User Setup

```sql
-- Create database and remote user
CREATE DATABASE company_db;
CREATE USER 'remote_admin'@'%' IDENTIFIED BY 'SecurePass123!';
GRANT ALL PRIVILEGES ON company_db.* TO 'remote_admin'@'%';
GRANT CREATE, ALTER, DROP, INSERT, UPDATE, DELETE, SELECT, REFERENCES, RELOAD ON *.* TO 'remote_admin'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

## Phase 4: Connectivity Testing

### 9. NAT Gateway Verification

```bash
# From private instance (no public IP)
curl ifconfig.me # Returns NAT Gateway's public IP
```

### 10. Database Connectivity Test

```bash
# From bastion host
mysql -h 10.0.2.15 -u remote_admin -p
Enter password: SecurePass123!

-- Inside MySQL
SHOW DATABASES;
USE company_db;
SELECT User, Host FROM mysql.user;
```
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
Cloud Student
[LinkedIn Profile](#) | [GitHub Profile](#)

