# Virtual Private Cloud (VPC)

## 1. Core Concept (Deep Explanation)

VPC is **Google Cloud's networking layer** that provides isolated network environments. Unlike AWS VPCs:

- **Global resource**: Single VPC spans all regions automatically
- **Subnets regional**: Subnets exist in specific regions (not AZs)
- **Routes global**: Define routing rules once, apply everywhere
- **VPC-native**: GKE clusters are VPC-native with pod networking
- **Private/Public IPs**: Resources can have private and/or public IPs

**Internal Architecture:**
- Subnets: Regional network ranges (10.0.0.0/20 in us-central1)
- Routes: Define traffic forwarding (default route → internet gateway)
- Firewall rules: Stateful filtering at network interface level
- Cloud NAT: Outbound internet access from private resources
- Cloud VPN / Interconnect: Hybrid connectivity

## 2. Why This Exists

**Problems it solves:**
- Network isolation (security blast radius)
- Multi-tenancy (teams in separate VPCs)
- Hybrid cloud connectivity (VPN to on-prem)
- Private database access (Cloud SQL via private IP)

## 3. When To Use

**Production systems require:**
- Private IP addressing for databases
- VPC-native GKE clusters
- Network segmentation between environments
- VPN/Interconnect to on-premises infrastructure

## 4. When NOT To Use

**Avoid:**
- Complex network topology (AWS has better DC networking)
- Multiple VPCs across many regions (VPC is global)

## 5. Real-World Example: Multi-Tier Production Architecture

```yaml
# VPC Structure
VPC: prod-platform (Global, auto-creates in all regions)

# Subnets per region
Subnet (us-central1):
  name: prod-api-subnet
  range: 10.1.0.0/20    # 4096 IPs
  secondary_ranges:
    pods: 10.4.0.0/14   # For GKE pods (255K IPs)
    services: 10.8.0.0/20  # For Kubernetes services

Subnet (us-east1):
  name: prod-data-subnet
  range: 10.2.0.0/20
  secondary_ranges:
    pods: 10.12.0.0/14
    services: 10.16.0.0/20

# Firewall Rules
Rules:
  - ingress-internet-https:
      priority: 1000
      direction: INGRESS
      allow: tcp:443
      sourceRanges: 0.0.0.0/0
      
  - ingress-internal:
      priority: 1001
      direction: INGRESS
      allow: tcp:0-65535,udp:0-65535
      sourceRanges: 10.0.0.0/8  (internal only)
      
  - egress-allow-all:
      priority: 2000
      direction: EGRESS
      allow: tcp:0-65535,udp:0-65535
      destinationRanges: 0.0.0.0/0

# Routes
Routes:
  - default-internet:
      destination: 0.0.0.0/0
      nextHopGateway: default-internet-gateway
      
  - internal-routing:
      destination: 10.0.0.0/8
      nextHopGateway: (stays local)

# Cloud NAT (for private instances)
NatGateway:
  name: prod-nat
  region: us-central1
  sourceSubnetworks: [prod-api-subnet]
```

## 6. Architecture Thinking

**VPC Global Span:**

```
┌─────────────────────────────────────────────────────────┐
│              VPC: prod-platform (Global)               │
│                                                         │
│  ┌──────────────────┐         ┌──────────────────┐    │
│  │ us-central1      │         │ us-east1          │    │
│  │ Subnet:          │         │ Subnet:           │    │
│  │ 10.1.0.0/20      │         │ 10.2.0.0/20       │    │
│  │                  │         │                   │    │
│  │ ┌──────┐        │         │ ┌──────┐         │    │
│  │ │GKE   │        │         │ │Cloud │         │    │
│  │ │Pod   │        │         │ │SQL   │         │    │
│  │ │10.1.1.5        │         │ │(private       │    │
│  │ └──────┘        │         │ │10.2.1.5)       │    │
│  │                  │         │ └──────┘         │    │
│  └────────┬─────────┘         └────────┬─────────┘    │
│           │   (routes transparently)   │               │
│  ┌────────▼─────────────────────────────▼──────┐      │
│  │ Global routing table                        │      │
│  │ - Internal: stay local                      │      │
│  │ - External: → Internet Gateway              │      │
│  └─────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────┘
```

**Traffic Flow - Internal:**

```
GKE Pod (us-central1)
    ↓
Wants to connect to Cloud SQL (us-central1)
    ↓
Checks routing table (destination 10.2.1.5)
    ↓
Route: 10.0.0.0/8 → stays local (next-hop local)
    ↓
Checks firewall (ingress-internal allows 10.0.0.0/8)
    ↓
Connection allowed
    ✓ Cloud SQL responds
```

**Traffic Flow - External Internet:**

```
GKE Pod (private IP: 10.1.1.5)
    ↓
Wants to reach external API (1.2.3.4)
    ↓
No explicit route for 1.2.3.4
    ↓
Default route: 0.0.0.0/0 → Cloud NAT
    ↓
Cloud NAT translates 10.1.1.5 → cloud-nat-ip (35.x.x.x)
    ↓
External API responds to cloud-nat-ip
    ↓
Cloud NAT reverse-translates back to 10.1.1.5
    ✓ Pod receives response
```

## 7. Common Mistakes

| Mistake | Problem | Solution |
|---------|---------|----------|
| Using overlapping IP ranges | Can't route correctly | Plan IP addressing (10.0.0.0/8 for dev, 10.1.0.0/8 for prod) |
| No firewall rules | Everything open (security risk) | Create explicit allow rules, default deny |
| Public IPs on sensitive databases | Database directly exposed | Use Private IP, disable public IP |
| All traffic through internet gateway | No encryption, potential inspection | Use VPN/Interconnect for hybrid traffic |
| Forgetting to enable VPC flow logs | Can't debug connectivity issues | Enable VPC flow logs for troubleshooting |
| Database in public subnet | Unintended exposure | Always use private subnet for databases |
| No Cloud NAT for outbound | Private instances can't reach internet | Enable Cloud NAT for outbound traffic |
| Single region disaster | Region failure = entire app down | Use regional subnets in multiple regions |
| Not sizing subnets large enough | Can't add new resources later | Plan for 2-3x growth (/16 or larger) |
| Overly permissive firewall | Unnecessary traffic flows | Restrict to specific protocols/IPs |

## 8. Interview Questions (with Answers)

### Q1: Design VPC architecture for multi-environment setup (dev, staging, prod).

**Answer:**
```
Option 1: Single VPC, separate subnets per environment
├─ prod-api-subnet: 10.1.0.0/20 (us-central1)
├─ staging-api-subnet: 10.2.0.0/20 (us-central1)
└─ dev-api-subnet: 10.3.0.0/20 (us-central1)
   ✓ Simpler management (one VPC)
   ✗ Less isolation (one VPC breach affects all)

Option 2: Separate VPC per environment (Recommended)
├─ prod-vpc: 10.0.0.0/8 (multiple regions)
├─ staging-vpc: 172.16.0.0/12
└─ dev-vpc: 192.168.0.0/16
   ✓ Blast radius contained (prod breach doesn't affect dev)
   ✓ Can have different security policies
   ✗ More complex to manage (VPC peering needed for cross-env)
```

**Implementation:**

```bash
# Create prod VPC
gcloud compute networks create prod-vpc \
  --subnet-mode=custom \
  --bgp-routing-mode=regional

# Create prod subnet
gcloud compute networks subnets create prod-api-subnet \
  --network=prod-vpc \
  --region=us-central1 \
  --range=10.1.0.0/20 \
  --secondary-ranges pods=10.4.0.0/14,services=10.8.0.0/20

# Create firewall rules for prod
gcloud compute firewall-rules create prod-allow-internal \
  --network=prod-vpc \
  --allow=tcp,udp,icmp \
  --source-ranges=10.0.0.0/8

# Create staging VPC with different IP range
gcloud compute networks create staging-vpc \
  --subnet-mode=custom

gcloud compute networks subnets create staging-api-subnet \
  --network=staging-vpc \
  --region=us-central1 \
  --range=172.16.0.0/20
```

### Q2: Peer two VPCs together. How does traffic flow?

**Answer:**
```bash
# Create VPC peering between prod and staging
gcloud compute networks peerings create prod-to-staging \
  --network=prod-vpc \
  --auto-create-routes \
  --peer-project=PROJECT_ID \
  --peer-network=staging-vpc

# Reverse peering
gcloud compute networks peerings create staging-to-prod \
  --network=staging-vpc \
  --auto-create-routes \
  --peer-project=PROJECT_ID \
  --peer-network=prod-vpc

# Verify peering
gcloud compute networks peerings list --network=prod-vpc
```

**Traffic Flow:**
```
Prod GKE Pod (10.1.1.5)
    ↓
Wants to reach Staging GKE Pod (172.16.1.5)
    ↓
Routing: 172.16.0.0/12 → peer network (staging-vpc)
    ↓
Traffic flows through VPC peering tunnel
    ✓ Networks connected
```

### Q3: Cloud SQL private IP is configured. How do GKE pods connect?

**Answer:**
```
Strategy 1: Shared VPC (recommended for same organization)
├─ Both GKE and Cloud SQL in same VPC
└─ Direct private IP connection (10.x.x.x)

Strategy 2: VPC Peering
├─ GKE in prod-vpc, Cloud SQL in data-vpc
├─ Peer prod-vpc ↔ data-vpc
├─ Pod connects to Cloud SQL private IP across peering

Strategy 3: VPC Connector (legacy)
├─ Connector in shared subnet
├─ Routes traffic through proxy
└─ Extra latency (~50ms)
```

**Implementation - Same VPC (best):**

```bash
# Create Cloud SQL in prod-vpc
gcloud sql instances create prod-db \
  --network=prod-vpc \
  --no-assign-ip  # Private IP only

# Deploy GKE in same VPC
gcloud container clusters create prod-gke \
  --network=prod-vpc \
  --subnetwork=prod-api-subnet \
  --enable-ip-alias \
  --cluster-secondary-range-name=pods \
  --services-secondary-range-name=services

# From pod, connect to Cloud SQL private IP
# Example: psql -h 10.1.2.5 -U postgres dbname
```

### Q4: Troubleshoot: Pod can't reach external API (connection timeout).

**Answer:**
```bash
# Step 1: Check if Cloud NAT is enabled
gcloud compute routers list --filter="region:us-central1"
gcloud compute routers nats list \
  --router=ROUTER_NAME \
  --router-region=us-central1

# Step 2: Check firewall rules (outbound)
gcloud compute firewall-rules list \
  --filter="network=prod-vpc AND direction=EGRESS" \
  --format='table(name, direction, allowed)'

# Verify egress rule allows external traffic
gcloud compute firewall-rules create prod-allow-external \
  --network=prod-vpc \
  --direction=EGRESS \
  --allow=tcp:443,tcp:80 \
  --destination-ranges=0.0.0.0/0

# Step 3: Check VPC Flow Logs
gcloud compute networks enable-flow-logs prod-vpc

# Step 4: Verify Cloud NAT has public IP
gcloud compute addresses list --filter="name:nat-ip"

# Step 5: Test from pod
kubectl exec -it POD_NAME -- curl https://api.example.com

# Step 6: Debug with ping if ICMP allowed
kubectl exec -it POD_NAME -- ping 8.8.8.8
```

### Q5: Design network segmentation with security zones.

**Answer:**
```
VPC Structure with Security Zones:

DMZ (Demilitarized Zone)
├─ Public subnet: 10.100.0.0/24
├─ Firewall rules: Allow ingress from internet (port 443)
└─ Resources: Cloud Load Balancer

Application Tier
├─ Private subnet: 10.110.0.0/24
├─ Firewall rules: Allow only from DMZ, permit to DB
└─ Resources: GKE cluster

Database Tier
├─ Private subnet: 10.120.0.0/24
├─ Firewall rules: Allow only from App tier
└─ Resources: Cloud SQL

Management Tier
├─ Private subnet: 10.130.0.0/24
└─ Firewall rules: Allow from bastion host only
```

**Firewall implementation:**

```bash
# DMZ → Public (open to internet)
gcloud compute firewall-rules create dmz-ingress-https \
  --network=prod-vpc \
  --allow=tcp:443 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=dmz

# DMZ → App
gcloud compute firewall-rules create dmz-to-app \
  --network=prod-vpc \
  --allow=tcp:8080 \
  --source-tags=dmz \
  --target-tags=app

# App → Database
gcloud compute firewall-rules create app-to-db \
  --network=prod-vpc \
  --allow=tcp:5432 \
  --source-tags=app \
  --target-tags=database

# Deny all by default
gcloud compute firewall-rules create default-deny-all \
  --network=prod-vpc \
  --direction=INGRESS \
  --priority=65534 \
  --deny=all
```

### Q6: Enable VPC Flow Logs to debug connectivity.

**Answer:**
```bash
# Enable flow logs on subnet
gcloud compute networks subnets update prod-api-subnet \
  --region=us-central1 \
  --enable-flow-logs \
  --logging-aggregation-interval=interval-5-sec \
  --logging-flow-sampling=0.5  # Sample 50% of flows

# Query flow logs
gcloud logging read \
  'resource.type="vpc_flow" AND jsonPayload.dest_ip="10.2.1.5"' \
  --format='table(timestamp, jsonPayload.src_ip, jsonPayload.dest_ip, jsonPayload.action)' \
  --limit 20

# Analyze flow logs in BigQuery
bq query --use_legacy_sql=false '
SELECT
  timestamp,
  src_ip,
  dest_ip,
  dest_port,
  action,
  bytes_sent,
  COUNT(*) as flow_count
FROM
  `project-id.vpc_flows.prod_api_subnet_*`
WHERE
  action = "ACCEPT"
  AND timestamp >= TIMESTAMP_SUB(NOW(), INTERVAL 1 HOUR)
GROUP BY
  timestamp, src_ip, dest_ip, dest_port, action, bytes_sent
ORDER BY
  bytes_sent DESC
'
```

### Q7: Set up hybrid connectivity with VPN to on-premises.

**Answer:**
```bash
# Create Cloud Router (BGP routing)
gcloud compute routers create prod-router \
  --network=prod-vpc \
  --region=us-central1 \
  --asn=64514

# Create VPN gateway
gcloud compute vpn-gateways create prod-vpn-gateway \
  --network=prod-vpc \
  --region=us-central1

# Create peer VPN gateway (on-premises side)
gcloud compute external-vpn-gateways create onprem-vpn-gateway \
  --interface address=203.0.113.1  # On-premises WAN IP

# Create VPN tunnel
gcloud compute vpn-tunnels create prod-to-onprem \
  --vpn-gateway=prod-vpn-gateway \
  --peer-external-gateway=onprem-vpn-gateway \
  --region=us-central1 \
  --shared-secret="YOUR_SHARED_SECRET" \
  --peer-external-gateway-interface=0

# Configure BGP for dynamic routing
gcloud compute routers update prod-router \
  --interface name=interface-0 \
  --interface-ip-address=169.254.0.1/30 \
  --peer-name=onprem-bgp \
  --peer-asn=65000 \
  --peer-ip-address=169.254.0.2

# Advertise GCP routes (pods, services, etc)
gcloud compute router bgp-peers create onprem-bgp \
  --router=prod-router \
  --router-asn=64514 \
  --interface=interface-0 \
  --peer-asn=65000
```

### Q8: Compare VPC peering vs. Shared VPC.

**Answer:**

| VPC Peering | Shared VPC |
|------------|-----------|
| Connect two separate VPCs | Single VPC, multiple projects |
| Bidirectional peering required | Admin project → member projects |
| Each project manages own resources | Admin project controls resources |
| Use case: Multi-org setup | Use case: Single org, many teams |
| Transitive peering: No direct flow across 3 VPCs | Transitive: All projects can communicate |
| Each VPC separate IAM | Shared IAM management |

### Q9: Design disaster recovery with multi-region VPC.

**Answer:**
```
VPC Architecture:

Primary Region (us-central1)
├─ GKE cluster: prod-gke-primary
├─ Cloud SQL: prod-db-primary
├─ Cloud Storage: regional snapshot

Secondary Region (us-east1)
├─ GKE cluster: prod-gke-secondary (standby)
├─ Cloud SQL: prod-db-replica
├─ Cloud Storage: same bucket (cross-region)

VPC Subnets
├─ Subnet us-central1: 10.1.0.0/20
└─ Subnet us-east1: 10.2.0.0/20
    (connected via global VPC routing)

Failover Strategy:
1. Cloud SQL automatic failover (if primary fails)
2. Load Balancer switches to secondary GKE
3. DNS update (if needed)
4. RTO: <1 minute, RPO: <5 minutes
```

### Q10: Network policy vs. firewall rules - what's the difference?

**Answer:**

| Firewall Rules | Network Policy (Kubernetes) |
|---|---|
| GCP-level (VM interface) | Kubernetes-level (pod interface) |
| Apply to all VMs/GKE nodes | Apply to pods only |
| Stateful (track connections) | Stateful |
| Source/destination by CIDR/tag | Source/destination by pod label |
| Faster (hardware enforced) | Slower (software enforced in pod) |

**Use both:**
```yaml
# Firewall: Allow app nodes to receive traffic on port 8080
gcloud compute firewall-rules create allow-app-traffic \
  --network=prod-vpc \
  --allow=tcp:8080 \
  --source-ranges=35.224.0.0/13  # LB range
  --target-tags=app-node

# Network Policy: Only app pods with label "frontend" can reach backend pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

## 9. Advanced Insights (Senior-level)

### Network Performance Optimization

**Subnet sizing:**
- /24: 254 usable IPs (small apps)
- /20: 4094 usable IPs (medium apps)
- /16: 65534 usable IPs (large apps)

**Secondary IP ranges:**
- Required for GKE pod networking (pods use secondary range)
- Plan: 1 pod per IP address (reserve 255K IPs for growth)

### Cost Optimization

**Minimize egress costs:**
- Private services (Cloud SQL) don't incur egress
- Use Cloud CDN for external content
- Egress to internet: $0.12/GB (expensive)

### Security Hardening

**Defense in depth:**
1. VPC isolation (separate VPCs per environment)
2. Firewall rules (stateful, default deny)
3. Network policies (pod-level segmentation)
4. VPC Service Controls (data exfiltration prevention)

