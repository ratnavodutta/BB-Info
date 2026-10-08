# IP Schemas — Azure, AWS, On-Prem/Zscaler

Reference map of IP address allocation across Blackbaud's cloud and hybrid network estate, built from the Terraform/docs in `C:\Repos`. This is a snapshot of what's documented in code and docs as of 2026-09-27 — it is not a substitute for Infoblox/IPAM as the system of record, but it cross-checks against it.

Sources: `network_team_azure`, `network_team_firewall`, `network_team_terraform`, `central-networking-core`, `rdo-security-dlp-zscaler`.

---

## 1. Azure

Two subscriptions anchor all Azure address space, each its own /16 supernet:

| Subscription | Purpose | Supernet |
|---|---|---|
| E01 | Education | 10.192.0.0/16 |
| N01 | Network (core/shared) | 10.193.0.0/16 |

Naming convention: `{subscription}-{region-abbrev}-{workload}-{resource-type}` (e.g. `e01-eus2-vwcc-vnet`, `n01-cus-zscc-fw-pip`).

### Regional hub allocation (N01, Virtual WAN backbone)

Regions get consecutive `/23` blocks spaced 8 apart within 10.193.0.0/16 (`network_team_azure\azure-core\docs\ADDING_NEW_REGIONS.md`):

| Region | Hub /23 | Resiliency (RIU) | AZ support |
|---|---|---|---|
| East US 2 | 10.193.0.0/23 | 5 (primary hub) | Yes |
| Central US | 10.193.8.0/23 | 3 | Yes |
| West US | 10.193.16.0/23 | 3 | No |
| Canada Central | 10.193.24.0/23 | 3 | Yes |
| West Europe | 10.193.32.0/23 | 3 | Yes |
| Australia East | 10.193.40.0/23 | 2 | Yes |
| Australia Southeast | 10.193.48.0/23 | 2 | No |
| Canada East | 10.193.56.0/23 | 2 | No |
| North Europe | 10.193.64.0/23 | 3 | Yes |

Next new region: 10.193.72.0/23, then 10.193.80.0/23.

E01 equivalent (from PoC doc, `AzureCloudNetworking_PoC.md`): regional allocation 10.192.0.0/21, VHub 10.192.0.0/23 (eus2), 10.192.8.0/23 (cus), Zscaler space 10.192.34.0/24.

### Zscaler Cloud/App Connector sub-allocations (per region, /24 carved from the hub /23)

From `rdo-security-dlp-zscaler\network_team_azure\zscc-vwan\tfvars`:

| Env-Region | VNet /24 | Public /26 | Cloud Connector /26 |
|---|---|---|---|
| e01-eus2 | 10.192.2.0/24 | 10.192.2.0/26 | 10.192.2.64/26 |
| e01-cus | 10.192.10.0/24 | 10.192.10.0/26 | 10.192.10.64/26 |
| n01-eus2 | 10.193.2.0/24 | 10.193.2.0/26 | 10.193.2.64/26 |
| n01-cus | 10.193.10.0/24 | 10.193.10.0/26 | 10.193.10.64/26 |
| n01-wus | 10.193.18.0/24 | 10.193.18.0/26 | 10.193.18.64/26 |
| n01-cnc | 10.193.26.0/24 | 10.193.26.0/26 | 10.193.26.64/26 |
| n01-we | 10.193.34.0/24 | 10.193.34.0/26 | 10.193.34.64/26 |
| n01-ae | 10.193.42.0/24 | 10.193.42.0/26 | 10.193.42.64/26 |
| n01-as | 10.193.50.0/24 | 10.193.50.0/26 | 10.193.50.64/26 |
| n01-cne | 10.193.58.0/24 | 10.193.58.0/26 | 10.193.58.64/26 |
| n01-ne | 10.193.66.0/24 | 10.193.66.0/26 | 10.193.66.64/26 |

### DNS Resolver allocation

Confirmed with the network team per doc comment: "E01 allocated 10.209.71.0/24; N01 allocated 10.209.68.0/23" (`azure-dns-resolver\docs\README.md`).

| Region | E01 /26 | Status |
|---|---|---|
| eastus2 | 10.209.71.0/26 | Active |
| centralus | 10.209.71.64/26 | Reserved |
| spare | 10.209.71.128/26, 10.209.71.192/26 | — |

N01 (from 10.209.68.0/23): eastus2 10.209.68.0/26 (Active), centralus 10.209.68.64/26, westus 10.209.68.128/26, canadacentral 10.209.68.192/26, westeurope 10.209.69.0/26, australiaeast 10.209.69.64/26, 2 spare /26s (all Reserved except eastus2).

### ExpressRoute peering

- N01: primary 10.214.0.92/30, secondary (APIPA) 169.254.0.92/30; destinations 10.226.0.64/26.
- E01: primary 10.214.0.88/30, secondary (APIPA) 169.254.0.88/30; destinations 10.213.7.0/26, 10.213.13.0/26.

### Azure Firewall public IP / NAT egress (per subscription + region)

| Sub-Region | Load Balancer (internal) | Public IP | VWCC NAT | ZSAC NAT |
|---|---|---|---|---|
| E01-CUS | 10.192.10.68 | 20.221.30.48/28 | 20.221.30.49 | 20.221.30.48 |
| E01-EUS2 | 10.192.2.70 | 20.1.213.16/28 | 20.1.213.17 | 20.1.213.16 |
| N01-AE | 10.193.42.68 | 20.211.63.136/29 | 20.211.63.137 | 20.211.63.136 |
| N01-CNC | 10.193.26.68 | 20.151.42.64/29 | 20.151.42.65 | 20.151.42.64 |
| N01-CUS | 10.193.10.68 | 20.109.197.176/28 | 20.109.197.177 | 20.109.197.176 |
| N01-EUS2 | 10.193.2.68 | 4.152.108.128/28 | 4.152.108.129 | 4.152.108.128 |
| N01-WE | 10.193.34.68 | 4.175.3.240/29 | 4.175.3.241 | 4.175.3.240 |
| N01-WUS | 10.193.18.68 | 52.190.143.48/28 | 52.190.143.49 | 52.190.143.48 |
| N01-AS | — | 52.189.194.184/29 | — | — |
| N01-CNE | — | 20.104.182.48/29 | — | — |
| N01-NE | — | 4.208.8.16/29 | — | — |

**New since 2026-09-27:** `network_team_azure` added zscc-vwan NAT Gateway public IPs for E01 — `e01-eus2-zscc-vwan-ngw-pip` and `e01-cus-zscc-vwan-ngw-pip` (PR 572420). Only the names are in code; the assigned addresses must be read from Azure. Not yet in the table above.

**E01 t60anet01 VNet:** `network_team_firewall` (PR 570391, merged 2026-09-23) uses 10.213.12.0/22 as the t60anet01 VNet (contains 10.213.13.0/26). Related ranges: loc00nfs02 subnets 10.213.12.224/27 and 10.213.14.0/27; ATX LO SQL subnets 10.18.111.0/24, 10.64.20.0/24, 10.64.30.0/24, 10.18.1.0/24; C8 Isilon subnet 10.18.240.0/23 (Dell SyncIQ ports 2098, 3148, 3149, 5667, 5668, 8470).

### Reservation / exclusion notes

`terraform-zscc-private-dns-azure` README: "Do not use these class C networks for DNS resolver subnets: 10.0.1.0/24 through 10.0.16.0/24."

### Test/sandbox only (not production allocation)

`network_team_terraform\azure_vnet_test` and `azure_vnet_peering_test`: 10.168.0.0/16, 10.186.0.0/16 — fixtures only, do not treat as real address space.

---

## 2. AWS

`central-networking-core` is the closest thing to an AWS IPAM system of record in the repos. Base supernet: **10.194.0.0/16**, one region per `/23` pair (prod / non-prod).

### Master region table

Allocation convention (`docs\adding-regions.md`): "Production: even-numbered third octet. Non-Production: prod + 2." Next region after the table: prd=10.194.60.0/23, np=10.194.62.0/23.

| Region | Prod CIDR | Non-Prod CIDR | Status |
|---|---|---|---|
| ca-central-1 | 10.194.0.0/23 | 10.194.2.0/23 | Active |
| eu-west-1 | 10.194.4.0/23 | 10.194.6.0/23 | Active (see discrepancy note) |
| eu-west-2 | 10.194.8.0/23 | 10.194.10.0/23 | Active (see discrepancy note) |
| us-east-1 | 10.194.12.0/23 | 10.194.14.0/23 | Active |
| us-east-2 | 10.194.16.0/23 | 10.194.18.0/23 | Active |
| us-west-1 | 10.194.20.0/23 | 10.194.22.0/23 | Active |
| us-west-2 | 10.194.24.0/23 | 10.194.26.0/23 | Active |
| ap-southeast-2 | 10.194.28.0/23 | 10.194.30.0/23 | Going live (GWLB routes enabled 2026-09-28; was Pending) |

**Discrepancy flagged:** `infrastructure-management.md` lists a different pair for eu-west-1/eu-west-2 — prod=10.194.36.0/23 / np=10.194.38.0/23 for eu-west-1, and prod=10.194.48.0/23 / np=10.194.50.0/23 for eu-west-2. This conflicts with the README table above and hasn't been reconciled. Worth raising with the network team before treating either version as final.

### Subnet-level pattern (per AZ, /27s inside a region's /23)

Example for us-east-1 non-prod (10.194.14.0/23):

| Tier | AZ-1a | AZ-1b |
|---|---|---|
| Core network | 10.194.14.0/27 | 10.194.14.32/27 |
| Firewall | 10.194.14.64/27 | 10.194.14.96/27 |
| GWLB | 10.194.14.128/27 | 10.194.14.160/27 |

**ap-southeast-2 note (updated 2026-10-08):** `enable_gwlb_routes = false` is now commented out in both np and prd tfvars, so the route to the new cloud connectors is provisioned. `gwlb_endpoint_service_names` keys must be the region's real AWS AZ IDs (`apse2-az1`, `apse2-az3`), not `region_short` (`ape2`) — the wrong prefix caused an "Invalid index" plan error. `ape2` is still correct for backend state file naming.

### Business-unit / application VPC address space (cross-confirmed between `network_team_firewall` and `central-networking-core`)

| Business unit | Environment | CIDR(s) | Region |
|---|---|---|---|
| Award Mgmt | Non-Prod | 10.210.12.0/24, 10.210.13.0/24 | us-east-1 |
| Award Mgmt | Prod | 10.210.14.0/24 | us-east-1 |
| Award Mgmt | Prod | 10.210.15.0/25 | eu-west-2 |
| Award Mgmt | Prod | 10.210.15.128/25 | ca-central-1 |
| Award Mgmt | Prod | 10.232.128.0/17 | ap-southeast-2 |
| BBEM (K-12) | Non-Prod (KDTO) | 10.230.160.0/19 | us-east-1 |
| BBEM (K-12) | Prod (KPE1) | 10.230.0.0/18 | us-east-1 |
| BBEM (K-12) | Prod (KME1) | 10.230.140.0/22 | us-east-1 |
| BBEM (K-12) | Prod (KPW2) | 10.230.252.0/22 | us-west-2 |
| BBTM (Tuition Mgmt) | Non-Prod (d04-ops01) | 10.230.208.0/20, 10.0.16.0/20 | us-east-1 |
| BBTM (Tuition Mgmt) | Prod (s32-ops01/prx01) | 10.202.16.0/20, 10.202.0.0/20, 192.168.12.0/24 | us-east-1 |
| Payments | Prod | 10.180.96.0/22 – 10.180.240.0/20 (7 blocks, see below), 10.245.128.0/20, 10.245.144.0/20, 10.246.128.0/20, 10.246.144.0/20, 10.246.176.0/22 | us-east-1 |
| OmniPoint | Prod | 10.210.32.0/24 | us-east-1 |
| LuminateOnline | Mixed | 10.0.2.0/24, 10.211.0.128/26, 10.209.70.0/24 (vpn-spoke, dev), 10.209.127.0/24 (VpnHub-0 / LO2), 10.232.1.0/24, 10.210.16.0/20 (Engineering), 10.233.32.0/20 (Cloudfoundry) | us-east-1 |
| GoodMove | Non-Prod | 10.100.0.0/16 | us-east-2 |
| GoodMove | Prod | 10.200.0.0/16 | us-east-2 |
| JustGiving | Non-Prod | 10.20.0.0/16 (STG-VPC), 10.22.0.0/16 (STG-LAMBDA) | eu-west-1 |
| JustGiving | Prod | 10.34.0.0/24 (PROD-DATA), 10.216.0.0/24 (PROD-BACKOFFICE), 10.50.0.0/16 (PRD-VPC), 10.52.0.0/16 (PRD-LAMBDA) | eu-west-1 |

Payments' 7 Prod blocks (all us-east-1): 10.180.96.0/22, 10.180.128.0/20, 10.180.144.0/20, 10.180.176.0/20, 10.180.192.0/20, 10.180.208.0/20, 10.180.240.0/20.

**Flag:** `10.230.208.0/20` and `192.168.12.0/24` are called out in one comment as "[secondary CIDR (overlapping)]" — worth reconciling against IPAM directly given the overlap analysis already run this session found real AWS-Azure conflicts in adjacent ranges.

### Zscaler App/Cloud Connector VPCs (AWS)

| Env | VPC CIDR | Source |
|---|---|---|
| kdto | 10.230.137.0/24 | `terraform-kdto.tfvars` |
| kpe1 | 10.230.139.0/24 | `terraform-kpe1.tfvars` |
| s30 | 10.246.18.0/23 | `terraform-s30.tfvars` |
| t03 | 10.245.18.0/23 | `terraform-t03.tfvars` |
| aws-np-cac1a (ZSCC) | 10.195.17.0/25 (cc 10.195.17.0/26, public 10.195.17.64/26) | `IMPLEMENTATION_PLAN.md` |
| Zscaler Cloud Connector (broad) | 10.195.0.0/19 | `central-networking-core\cloudwatch-query.tf` comment |

### Global RFC1918 allow-lists (repeated across firewall policies)

10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 — plus 100.64.0.0/10 (CGNAT) in one variant. Note: `network_team_azure` module defaults also include 198.18.0.0/16 in the RFC1918 allow-list for the AWS ZSAC module.

### AWS Zscaler NAT egress IPs (partial — non-prod fully captured, prod table was truncated when read)

| Region | ZSCC-A (non-prod) | ZSCC-B (non-prod) |
|---|---|---|
| us-east-1 | 100.49.169.105 | 44.196.115.65 |
| us-east-2 | 3.143.60.153 | 3.137.28.225 |
| us-west-1 | 52.53.116.186 (B) | 54.241.159.105 (C) |
| us-west-2 | 54.244.88.173 | 52.34.109.197 |
| ca-central-1 | 15.157.250.133 | 15.223.109.39 |
| eu-west-1 | 54.75.179.226 | 52.51.31.105 |
| eu-west-2 | 18.171.255.227 | 35.178.21.66 |

Prod table starts with us-east-1 ZSCC-A 52.2.59.75 / ZSCC-B 3.225.187.97; the rest of the prod table wasn't fully read this pass — see `rdo-security-dlp-zscaler\docs\operations.md` (lines ~159-260) for the complete set.

---

## 3. On-Prem / Other

No dedicated on-prem IP plan document was found in these repos (on-prem/nios_x address space lives in Infoblox per the dashboard pull done earlier this session — see the IPAM overlap report for the 482 on-prem networks captured there).

Reservations and cross-cloud markers found in these repos that touch on-prem or hybrid connectivity:
- ExpressRoute peering ranges above (10.214.0.88/30, 10.214.0.92/30, plus APIPA 169.254.0.88/30 and 169.254.0.92/30) connect Azure back to Blackbaud's WAN/on-prem edge.
- `network_team_terraform` test fixtures (10.168.0.0/16, 10.186.0.0/16) are sandbox-only.

---

## 4. Naming / Tagging Conventions (useful for future automated parsing)

- **Azure:** `{subscription}-{region-abbrev}-{workload}-{resource-type}` (subscriptions: E01=Education, N01=Network). Region abbreviations: eus2, cus, wus, cnc, we, ae, as, cne, ne.
- **AWS (central-networking-core):** backend state files named `{region}-{np|prd}[-backend].hcl`; region abbreviations: cac1, euw1, euw2, use1, use2, usw1, usw2, ape2.
- **AWS (network_team_firewall):** rule-group files prefixed with a numeric SID range + region + app, e.g. `1800-rule_groups_us-east-1-award_mgmt.tf`, `25200-rule_groups_eu-west-1-justgiving.tf` — the app name in the filename doubles as the "purpose" tag for the CIDRs inside it.

---

## 5. Open Items / Discrepancies to Resolve

1. **eu-west-1 / eu-west-2 AWS prod-np pair mismatch** between `central-networking-core\README.md` and `central-networking-core\docs\infrastructure-management.md` (see section 2). Needs network team confirmation on which is current.
2. **10.230.208.0/20 and 192.168.12.0/24** flagged inline as overlapping — cross-reference against the IPAM overlap findings from the dashboard/SharePoint/Azure Resource Graph comparison done earlier this session.
3. **AWS Zscaler prod NAT IP table** only partially captured from `rdo-security-dlp-zscaler\docs\operations.md` — re-read lines ~159-260 for the full set if needed.
4. No single on-prem IP plan document exists in these repos; on-prem allocation should be sourced from Infoblox/the CDE dashboard directly, not from `C:\Repos`.

---

*Built by reading `network_team_azure`, `network_team_firewall`, `network_team_terraform`, `central-networking-core`, and `rdo-security-dlp-zscaler` under `C:\Repos` on 2026-09-27. Cross-check against Infoblox/IPAM before treating any range here as authoritative for provisioning — this reflects what's in Terraform/docs, not necessarily what's actually deployed.*
