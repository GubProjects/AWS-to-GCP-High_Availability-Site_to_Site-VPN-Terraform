# AWS ↔ GCP High-Availability Site-to-Site VPN (Terraform)

This project provisions a high-availability, BGP-based site-to-site VPN between AWS and Google Cloud using Terraform.

It creates a resilient, multi-tunnel IPSec connection that enables private, encrypted traffic between a VPC in AWS and a VPC in GCP.

> **This is not a single-tunnel demo. It is a production-style hybrid network design.**

## 🎯 What This Builds

| Layer | AWS | GCP |
|-------|-----|-----|
| Network | VPC (10.230.0.0/16) | Custom VPC |
| Gateway | Virtual Private Gateway | HA VPN Gateway |
| Routing | BGP over IPSec | Cloud Router (BGP) |
| Redundancy | 2 VPN connections | 4 tunnel endpoints |
| Encryption | IPSec + IKEv2 | IPSec + IKEv2 |

**Traffic flows securely between clouds using dynamic routing (BGP) so routes fail over automatically.**

## 🏗️ Architecture Overview

```
AWS VPC
  │
  ├── VPN Gateway
  │     ├─ Tunnel 1
  │     ├─ Tunnel 2
  │     ├─ Tunnel 3
  │     └─ Tunnel 4
  │
  ▼
Encrypted IPSec Tunnels
  ▲
  │
GCP HA VPN Gateway
  │
  └── Cloud Router (BGP)
```

Each tunnel runs BGP so routes are learned dynamically. If one tunnel fails, traffic shifts automatically.

## 🔁 Traffic Flow

1. AWS advertises VPC CIDR via BGP
2. GCP Cloud Router learns routes
3. GCP advertises its subnet ranges
4. AWS updates routes automatically
5. Encrypted traffic flows over the best tunnel

**No static routes. No manual failover.**

## 🔐 Security Model

- IPSec encryption
- IKEv2 key exchange
- BGP authentication
- Firewall rules scoped to tunnel IPs
- Segmented cloud networks

**All traffic is private and encrypted.**

## 📦 Resources Created

### AWS

- VPC
- Virtual Private Gateway
- Two Customer Gateways
- Two VPN connections (4 tunnels)

### GCP

- Custom VPC
- HA VPN Gateway
- External VPN Gateway (AWS)
- Cloud Router (BGP)
- Four VPN tunnels
- Router interfaces + peers
- Firewall rules for IPSec

## 🧩 Design Decisions

| Decision | Why |
|----------|-----|
| HA VPN on GCP | Native multi-interface redundancy |
| 4 tunnels | Tolerates gateway or AZ failures |
| BGP routing | Automatic failover |
| Terraform | Reproducible, auditable infrastructure |

## 🚀 Usage

### Prerequisites

- AWS & GCP accounts
- Terraform ≥ 1.5
- GCP service account key
- AWS credentials configured

### Deploy

```bash
terraform init
terraform plan
terraform apply
```

### Verify

After deployment, verify tunnel status:

```bash
# AWS: Check tunnel status
aws ec2 describe-vpn-connections --region us-east-1

# GCP: Check tunnel status
gcloud compute vpn-tunnels list

# Verify BGP routes learned
gcloud compute routers get-status <router-name> --region us-central1
```

## ⚠️ Security Notes

- Replace all shared secrets before deployment
- Do not commit credentials to version control
- Restrict firewall rules in production (principle of least privilege)
- Enable logging on both sides for troubleshooting
- Rotate pre-shared keys periodically
- Review BGP authentication settings

## 🎯 Use Cases

- Hybrid cloud workloads
- Cross-cloud disaster recovery
- Multi-cloud Kubernetes cluster connectivity
- Secure service connectivity across providers
- Cloud migration with live traffic

## 📊 Observability

Monitor tunnel and BGP health:

- AWS CloudWatch metrics for VPN connections
- GCP Cloud Monitoring for VPN tunnels
- BGP session status via Cloud Router
- Firewall rule logging

## 🧠 Philosophy

> Cloud networks should fail safely, route dynamically, and recover automatically.

This project shows how to build real hybrid connectivity, not a lab demo.

---

## Project Structure

```
.
├── main.tf              # Root module & provider config
├── aws.tf               # AWS VPC, VPN resources
├── gcp.tf               # GCP VPC, HA VPN resources
├── bgp.tf               # BGP routing configuration
├── variables.tf         # Input variables
├── outputs.tf           # Output values
├── terraform.tfvars     # Variable assignments (example)
└── README.md            # This file
```

## Prerequisites Setup

### AWS

```bash
# Configure AWS credentials
aws configure

# Or use environment variables
export AWS_ACCESS_KEY_ID="..."
export AWS_SECRET_ACCESS_KEY="..."
```

### GCP

```bash
# Create service account
gcloud iam service-accounts create terraform-sa

# Grant required permissions
gcloud projects add-iam-policy-binding <PROJECT_ID> \
  --member="serviceAccount:terraform-sa@<PROJECT_ID>.iam.gserviceaccount.com" \
  --role="roles/compute.admin"

# Create and download key
gcloud iam service-accounts keys create key.json \
  --iam-account=terraform-sa@<PROJECT_ID>.iam.gserviceaccount.com

export GOOGLE_APPLICATION_CREDENTIALS="$(pwd)/key.json"
```

## Troubleshooting

### Tunnel Down

1. Verify firewall rules allow IPSec traffic (UDP 500, 4500)
2. Check pre-shared key matches on both sides
3. Review logs in CloudWatch (AWS) and Cloud Logging (GCP)

### BGP Not Exchanging Routes

1. Verify BGP session is established: `gcloud compute routers get-status <router>`
2. Check IP address configuration on both sides
3. Verify firewall allows BGP traffic (TCP 179)

### High Latency

1. Confirm tunnel is active and not falling back to secondary
2. Monitor cloud region latency
3. Consider regional VPN gateway placement

## Contributing

Issues and pull requests welcome. Please include:

- Terraform version
- Cloud provider versions
- Error logs
- Steps to reproduce

## License

[Add appropriate license]

## Support

For issues with this Terraform configuration, open a GitHub issue.  
For cloud provider-specific issues, consult AWS and GCP documentation.
