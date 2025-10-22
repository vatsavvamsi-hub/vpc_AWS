# AWS VPC Components - Technical Overview

## VPC Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            AWS Region (us-east-1)                              │
│                                                                                 │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │                    VPC (10.0.0.0/16)                                     │  │
│  │                                                                           │  │
│  │  ┌─────────────────────┐                ┌─────────────────────┐           │  │
│  │  │  Availability Zone A │                │  Availability Zone B │           │  │
│  │  │                     │                │                     │           │  │
│  │  │  ┌───────────────┐  │                │  ┌───────────────┐  │           │  │
│  │  │  │ Public Subnet │  │                │  │ Public Subnet │  │           │  │
│  │  │  │ 10.0.1.0/24   │  │                │  │ 10.0.2.0/24   │  │           │  │
│  │  │  │               │  │                │  │               │  │           │  │
│  │  │  │ [EC2-Web]     │  │                │  │ [EC2-Web]     │  │           │  │
│  │  │  │ [Load Balancer│  │                │  │ [NAT Gateway] │  │           │  │
│  │  │  └───────────────┘  │                │  └───────────────┘  │           │  │
│  │  │                     │                │                     │           │  │
│  │  │  ┌───────────────┐  │                │  ┌───────────────┐  │           │  │
│  │  │  │Private Subnet │  │                │  │Private Subnet │  │           │  │
│  │  │  │ 10.0.10.0/24  │  │                │  │ 10.0.20.0/24  │  │           │  │
│  │  │  │               │  │                │  │               │  │           │  │
│  │  │  │ [RDS Primary] │  │                │  │ [RDS Standby] │  │           │  │
│  │  │  │ [App Servers] │  │                │  │ [App Servers] │  │           │  │
│  │  │  └───────────────┘  │                │  └───────────────┘  │           │  │
│  │  └─────────────────────┘                └─────────────────────┘           │  │
│  │                                                                           │  │
│  │  ┌─────────────────────────────────────────────────────────────────────┐  │  │
│  │  │                      Route Tables                                  │  │  │
│  │  │  Public RT: 0.0.0.0/0 → Internet Gateway                           │  │  │
│  │  │  Private RT: 0.0.0.0/0 → NAT Gateway                               │  │  │
│  │  │              10.0.0.0/16 → Local                                   │  │  │
│  │  └─────────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                           │  │
│  │  ┌─────────────────────────────────────────────────────────────────────┐  │  │
│  │  │                    Security Groups                                  │  │  │
│  │  │  Web-SG: HTTP(80), HTTPS(443) from 0.0.0.0/0                       │  │  │
│  │  │  App-SG: Port 8080 from Web-SG only                                │  │  │
│  │  │  DB-SG: Port 3306 from App-SG only                                 │  │  │
│  │  └─────────────────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
│                                    │                                           │
│              ┌─────────────────────┴─────────────────────┐                     │
│              │         Internet Gateway                 │                     │
│              │         (igw-12345678)                   │                     │
│              └─────────────────────┬─────────────────────┘                     │
└─────────────────────────────────────┼─────────────────────────────────────────┘
                                      │
                              ┌───────┴───────┐
                              │   Internet    │
                              │               │
                              └───────────────┘

VPC Endpoints:
┌─────────────────┐    ┌─────────────────┐
│   S3 Endpoint   │    │ DynamoDB        │
│   (Gateway)     │    │ Endpoint        │
└─────────────────┘    └─────────────────┘
```

## Technical Component Breakdown

### 1. VPC (Virtual Private Cloud)
- **CIDR Block**: 10.0.0.0/16 (65,536 IP addresses)
- **Isolation**: Logically isolated network segment in AWS cloud
- **Tenancy**: Default (shared hardware) or Dedicated (isolated hardware)
- **DNS**: Provides DNS resolution (enableDnsSupport, enableDnsHostnames)

### 2. Subnets
**Public Subnets** (10.0.1.0/24, 10.0.2.0/24)
- Routes traffic to Internet Gateway
- Instances get public IP addresses
- Used for: Load balancers, bastion hosts, NAT gateways

**Private Subnets** (10.0.10.0/24, 10.0.20.0/24)
- No direct internet access
- Routes traffic through NAT Gateway for outbound
- Used for: Application servers, databases, backend services

### 3. Internet Gateway (IGW)
- **Function**: Enables internet connectivity for VPC
- **Scalability**: Horizontally scaled, redundant, highly available
- **One-to-One**: One IGW per VPC
- **Stateless**: Performs network address translation (NAT) for instances with public IPs

### 4. NAT Gateway
- **Purpose**: Allows private subnet instances to access internet
- **Placement**: Must be in public subnet
- **Bandwidth**: Scales up to 45 Gbps
- **High Availability**: Deploy in multiple AZs for redundancy
- **Managed Service**: AWS handles patching and maintenance

### 5. Route Tables
**Public Route Table**:
```
Destination     Target
10.0.0.0/16    Local
0.0.0.0/0      igw-12345678
```

**Private Route Table**:
```
Destination     Target
10.0.0.0/16    Local
0.0.0.0/0      nat-67890123
```

### 6. Security Groups (Instance-level Firewall)
- **Stateful**: Return traffic automatically allowed
- **Allow Rules Only**: Cannot create deny rules (implicit deny)
- **Instance Level**: Applied to ENI (Elastic Network Interface)
- **Multiple Assignment**: Instance can have multiple security groups

### 7. Network ACLs (Subnet-level Firewall)
- **Stateless**: Must explicitly allow return traffic
- **Allow/Deny Rules**: Support both allow and deny rules
- **Subnet Level**: Applied to entire subnet
- **Rule Numbers**: Processed in numerical order (100, 200, etc.)

### 8. VPC Endpoints
**Gateway Endpoints** (S3, DynamoDB):
- Route table entries
- No additional charges for data transfer
- Prefix lists for security group rules

**Interface Endpoints** (Other AWS services):
- ENI with private IP in subnet
- Powered by AWS PrivateLink
- Charged per hour + data processing

### 9. Additional Components

**DHCP Options Set**:
- Domain name servers
- Domain name
- NTP servers
- NetBIOS name servers

**VPC Peering**:
- Connect VPCs across accounts/regions
- Non-transitive routing
- CIDR blocks cannot overlap

**Transit Gateway**:
- Regional network hub
- Connects VPCs and on-premises networks
- Supports multicast and routing tables