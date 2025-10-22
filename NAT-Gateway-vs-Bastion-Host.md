# NAT Gateway vs Bastion Host in AWS

## Overview

These serve completely different purposes in AWS networking:

## NAT Gateway

**Purpose:** Enables private subnet instances to initiate outbound internet connections while blocking inbound traffic from the internet.

**Use case:** Downloading updates, accessing external APIs, or connecting to external services from private instances.

**Traffic flow:** Outbound only (instances → internet)

### Diagram

```text
┌─────────────────────────────────────────────────────┐
│                     VPC                             │
│                                                     │
│  ┌──────────────────┐      ┌──────────────────┐   │
│  │  Public Subnet   │      │  Private Subnet  │   │
│  │                  │      │                  │   │
│  │  ┌────────────┐  │      │  ┌────────────┐ │   │
│  │  │ NAT Gateway│◄─┼──────┼──│  EC2       │ │   │
│  │  └─────┬──────┘  │      │  │ (private)  │ │   │
│  │        │         │      │  └────────────┘ │   │
│  └────────┼─────────┘      └──────────────────┘   │
│           │                                        │
└───────────┼────────────────────────────────────────┘
            │
            ▼
    Internet Gateway
            │
            ▼
        Internet
```

## Bastion Host (Jump Server)

**Purpose:** A secure entry point for administrators to SSH/RDP into private instances from the internet.

**Use case:** Remote management and administration of instances in private subnets.

**Traffic flow:** Inbound (admin → bastion → private instances)

### Diagram

```text
┌─────────────────────────────────────────────────────┐
│                     VPC                             │
│                                                     │
│  ┌──────────────────┐      ┌──────────────────┐   │
│  │  Public Subnet   │      │  Private Subnet  │   │
│  │                  │      │                  │   │
│  │  ┌────────────┐  │      │  ┌────────────┐ │   │
│  │  │  Bastion   │──┼──────┼─►│  EC2       │ │   │
│  │  │   Host     │  │ SSH  │  │ (private)  │ │   │
│  │  └─────▲──────┘  │      │  └────────────┘ │   │
│  │        │         │      │                  │   │
│  └────────┼─────────┘      └──────────────────┘   │
│           │                                        │
└───────────┼────────────────────────────────────────┘
            │
    Internet Gateway
            │
            ▲
    Admin (SSH/RDP)
```

## Key Differences

| Aspect | NAT Gateway | Bastion Host |
|--------|-------------|--------------|
| **Direction** | Outbound from private instances | Inbound to private instances |
| **Managed** | Fully managed AWS service | Self-managed EC2 instance |
| **Purpose** | Internet access for updates/APIs | Remote administration access |
| **Security** | No direct access to instances | Requires hardening & monitoring |
| **Cost** | Hourly + data processing charges | EC2 instance costs |

## Best Practices

### NAT Gateway
- Place in a public subnet
- Associate with an Elastic IP
- Update route tables in private subnets to route `0.0.0.0/0` to NAT Gateway
- Consider using one NAT Gateway per AZ for high availability

### Bastion Host
- Use a hardened AMI (minimal software)
- Restrict Security Group to specific IP addresses
- Enable detailed logging and monitoring
- Consider using AWS Systems Manager Session Manager as an alternative
- Regularly patch and update the OS
- Use MFA for access

## Common Architecture

Many VPCs use **both** components:
- **NAT Gateway** for outbound connectivity (software updates, API calls)
- **Bastion Host** for administrative access (SSH/RDP into private instances)

This combination provides secure internet access for applications while maintaining controlled administrative access.

## Modern Alternatives

- **AWS Systems Manager Session Manager**: Replaces Bastion Hosts with browser-based shell access
- **AWS PrivateLink**: For private connectivity to AWS services without NAT Gateway
- **VPC Endpoints**: For accessing specific AWS services without internet gateway
