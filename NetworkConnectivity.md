# Network Connectivity Map — DNS Resolvers, Private Endpoints, ExpressRoute, Direct Connect

Companion to `IPschemas.md` in this same folder. Built from the same five repos under `C:\Repos` (`network_team_azure`, `network_team_firewall`, `network_team_terraform`, `central-networking-core`, `rdo-security-dlp-zscaler`). Same caveat as `IPschemas.md`: this reflects what's in Terraform/docs, not necessarily live state — cross-check against Azure/AWS directly before relying on it operationally.

---

## 1. DNS Resolvers — Azure Private DNS Resolver

There are exactly two `azurerm_private_dns_resolver` deployments in the canonical module (`network_team_azure\azure-dns-resolver`), one per subscription, both in **eastus2 only** — every other region is planned in docs but not deployed.

### E01 (Education subscription `7253b84c-2f1a-4b10-9046-1fbdcc4334f1`)

Tagged `Environment = "Test"`, `UsedFor = "Test"` in the tfvars — worth flagging since E01 = "Education" conceptually, not "Test" (see Open Items below).

| Item | Value |
|---|---|
| Resource group | `e01-eus2-dnsresolver-rg-1` |
| Region | eastus2 |
| VNet | `e01-eus2-dnsresolver-vnet` — 10.209.71.0/26 |
| Inbound subnet / endpoint | `e01-eus2-dnsresolver-inbound-snet` (10.209.71.0/28) → `e01-eus2-dnsresolver-inbound-ep-1` (IP dynamically assigned by Azure from this /28, not pinned in code) |
| Outbound subnet / endpoint | `e01-eus2-dnsresolver-outbound-snet` (10.209.71.16/28) → `e01-eus2-dnsresolver-outbound-ep-1` |
| Forwarding ruleset | `e01-eus2-dnsresolver-ruleset-1` — 11 rules (blackbaud.global, d00/d01/d05/d06/d07/d40, t00/t01/t07/t40/t45.blackbaud.net) |
| VNet links | Own VNet (required) + `cyber-tool-vnet02` (sub `11a6fb25-e963-4503-9f13-2f400ca2ad2a`, RG `bbsecteus2ark-rg01`, VNet `bbsecteus2ark-vn02`). `cyber-tool-vnet01` link was removed (commented out). |

Source: `network_team_azure\azure-dns-resolver\env\e01\eus2\terraform.tfvars`

### N01 (Network/shared subscription `150ea8a8-4006-49a6-8c11-b7d7c2942e61`)

Tagged `Environment = "Production"`, `UsedFor = "Production"`.

| Item | Value |
|---|---|
| Resource group | `n01-eus2-dnsresolver-rg-1` |
| Region | eastus2 |
| VNet | `n01-eus2-dnsresolver-vnet` — 10.209.68.0/26 |
| Inbound subnet / endpoint | `n01-eus2-dnsresolver-inbound-snet` (10.209.68.0/28) → `n01-eus2-dnsresolver-inbound-ep-1` |
| Outbound subnet / endpoint | `n01-eus2-dnsresolver-outbound-snet` (10.209.68.16/28) → `n01-eus2-dnsresolver-outbound-ep-1` |
| Forwarding ruleset | `n01-eus2-dnsresolver-ruleset-1` — 27 rules (blackbaud.global; s00/s10/s20/s21/s22/s23/s24/s26/s27/s28/s29/s36/s40/s41/s42/s45/s46/s47/s48/s51/s52/s53/s54/s55/s57.blackbaud.net; bbps.com; blackbaud.com; bbpdev.blackbaud.com.au; blackbaud.net; blackbaudhost.com; blackbaudlab.global; corp.dco.net; ctx.cdev.dco.net; dco.net) |
| VNet links | Own VNet only. A CyberArk link to `cyber-tool-vnet01` (sub `c807483d-835a-4e97-8aa1-17a013df9663`, RG `sec02peus2ark-rg01`, VNet `sec02peus2ark-vn01`) was **attempted and failed** — a captured `LinkedAuthorizationFailed` error sits in the tfvars comments. Not "not yet done" — actually blocked. |

Source: `network_team_azure\azure-dns-resolver\env\n01\eus2\terraform.tfvars`

**Both are private resolvers** (dedicated VNet + inbound/outbound endpoints). No public/Azure-managed DNS resolver resources exist in code. Without a VNet link, the docs note inbound queries fall back to Azure's platform DNS `168.63.129.16`, which returns NXDOMAIN for private domains — that address is the implicit "no private resolver reachable" fallback, not a deployed resolver.

**A second, currently-inactive DNS resolver code path exists**: `rdo-security-dlp-zscaler\network_team_azure\modules-zscc\terraform-zscc-private-dns-azure\main.tf` defines its own `azurerm_private_dns_resolver` for Zscaler ZPA app-segment DNS steering, invoked from `zscaler-cc-terraform-azure\examples\cc_lb\main.tf` gated by `zpa_enabled`. The only example wiring this up (`terraform_d40.tfvars`, sub `38dc1973-b3ec-4db8-8f42-f76fc59e65b0`, eastus2) has `zpa_enabled` commented out — so this module is **not currently deployed** anywhere, it just exists in code.

**Summary: 2 live private DNS resolvers (E01 + N01, both eastus2), 1 dormant Zscaler ZPA private resolver module (not deployed).**

---

## 2. Private Endpoints & Private DNS Zones — Azure

**Zero `azurerm_private_endpoint` resources exist in any of the five repos.** Grepped `azurerm_private_endpoint`, `private_endpoint`, `privatelink` across everything — the only hit was a commented-out `private_endpoint_network_policies_enabled` setting in `network_team_terraform\modules\azurerm_subnet\main.tf` (the subnet module supports the flag, nothing actually uses it).

**Zero `azurerm_private_dns_zone` resources exist anywhere** either — no `privatelink.*.azure.com` zone linking pattern is present in this codebase.

**Count: 0 Private Endpoints, 0 Private DNS Zones.** State storage accounts referenced in these repos (e.g. `e01eus2vwanst`, `n01eus2vwanst`) appear to be reached via standard endpoints, not Private Endpoints, as far as this code shows.

---

## 3. AWS Equivalents — Route 53 Resolver & VPC Endpoints / PrivateLink

### Route 53 Resolver

One module: `rdo-security-dlp-zscaler\zscaler-cc-terraform-aws\modules\terraform-zscc-route53-aws\main.tf` — `aws_route53_resolver_endpoint.zpa_r53_ep`, **outbound only**, one per Zscaler Cloud Connector VPC deployment. IPs are dynamically allocated (not pinned). Paired with a FORWARD rule (routes ZPA app-segment domains to the Cloud Connector) and a SYSTEM rule set (lets AWS resolve Zscaler's own domains directly).

**No inbound Route 53 Resolver endpoint exists anywhere in these repos.**

Exact count of live outbound endpoints wasn't fully confirmable from static code alone (module is referenced by name in multiple examples; not every example's enable-flag was traced) — treat as "up to one per Cloud Connector VPC" rather than a hard number.

### VPC Endpoints / AWS PrivateLink for managed services (S3, DynamoDB, etc.)

**Zero found.** Every `aws_vpc_endpoint*` resource in these repos is `vpc_endpoint_type = "GatewayLoadBalancer"` — used exclusively for Zscaler traffic interception, not AWS service PrivateLink.

### GatewayLoadBalancer VPC Endpoints (traffic-inspection, not PrivateLink-to-AWS-services)

Two independent systems creating/consuming GWLB endpoints:

**(a) `central-networking-core\core\gateway_load_balancers.tf`** — consumes pre-existing Zscaler-side endpoint services by ARN, one per AZ, across 16 tfvars files (8 regions × prod/non-prod): ap-southeast-2, ca-central-1, eu-west-1, eu-west-2, us-east-1, us-east-2, us-west-1, us-west-2. **~32 active endpoints** (2 per file × 16 files); additional endpoint-service entries exist commented out (e.g. `ath-prd` in `us-east-1-prd.tfvars`), implying a prior or parallel Zscaler tenant/migration.

**(b) `rdo-security-dlp-zscaler\zscaler-cc-terraform-aws\modules\terraform-zscc-gwlbendpoint-aws\main.tf`** — creates the GWLB endpoint service + endpoint per Cloud Connector subnet/AZ. Standard matrix: 32 deployments (16 non-prod + 16 prod) across the same 8 regions. Plus 6 custom/named deployments (kdto, t03=test, s30=prod, IDM, kpe1, kpw2 — mostly us-east-1, kpw2 in us-west-2) adding **15 more** endpoints. Plus one more top-level example (`cc_gwlb_asg\terraform.tfvars`, env "KDTO", az_count=2) that may duplicate the `terraform-kdto.tfvars` config rather than being a separate live environment (unconfirmed from static analysis).

**Combined tally: roughly 79–81 GWLB-type VPC endpoints** across 8+ AWS regions, split prod/non-prod. This is a tally, not a de-duplicated count — (a) and (b) may be client/service pairs of the same logical connections rather than fully independent endpoints.

**AWS account IDs found in plaintext**: only `070978211605` and `798924148385` (tagged `Workload = "TransitHub"` in `base_cc_gwlb_asg\main.tf`; everything else is tagged `"Decentralized"`). Other identifiers are AWS CLI profile names (`aws-prd`, `bbk12`, `dto`, `IDM`), not raw account numbers.

---

## 4. ExpressRoute (Azure)

Module: `network_team_azure\azure-core\modules\expressroute_circuit\main.tf`. **Exactly 2 circuits total, one per subscription — confirmed no others exist anywhere.**

### E01 — `e01-eus2-to-dc3-er`

| Item | Value |
|---|---|
| Subscription | E01 `7253b84c-2f1a-4b10-9046-1fbdcc4334f1` — tagged Environment="Test", PCIAuditLevel="Not PCI Level 1a or Level 2" |
| Resource group / region | `e01-eus2-vwan-rg-1` / eastus2 |
| Provider / peering location | Equinix, Washington DC |
| Bandwidth / SKU | 1000 Mbps, Standard tier, MeteredData family |
| Peering | AzurePrivatePeering, VLAN 1110, peer ASN 65000 |
| Primary peer prefix | 10.214.0.88/30 |
| Secondary (APIPA) | 169.254.0.88/30 |
| Destinations routed via this circuit | 10.213.7.0/26 (d60anet01, Oracle Delegated Subnet, via `AzureFirewall_e01-eus2-vhub-1`); 10.213.13.0/26 (t60anet01, Oracle Delegated Subnet, via `AzureFirewall_e01-cus-vhub-1`) |
| Gateway | `e01-eus2-vhub-ergw-1`, 1 scale unit, on vhub `e01-eus2-vhub-1` (10.192.0.0/23) |

A second vhub `e01-cus-vhub-1` (centralus, 10.192.8.0/23) exists but its ER gateway is commented out — not deployed.

### N01 — `n01-eus2-to-dc3-er`

| Item | Value |
|---|---|
| Subscription | N01 `150ea8a8-4006-49a6-8c11-b7d7c2942e61` — tagged Environment="Production", PCIAuditLevel="Level2" |
| Resource group / region | `n01-eus2-vwan-rg-1` / eastus2 |
| Provider / peering location | Equinix, Washington DC (same colo as E01, separate circuit) |
| Bandwidth / SKU | 5000 Mbps, Standard tier, MeteredData family |
| Peering | VLAN 1111, peer ASN 65000 |
| Primary peer prefix | 10.214.0.92/30 |
| Secondary (APIPA) | 169.254.0.92/30 |
| Destination routed via this circuit | 10.226.0.64/26 (s20ablkbvn, Oracle Delegated Subnet, via `AzureFirewall_n01-eus2-vhub-1`) |
| Gateway | `n01-eus2-vhub-ergw-1`, **3** scale units |

N01 has 8 additional vhubs (cus, wus, cnc, we, ae, as, cne, ne). ER gateways are actually provisioned in eastus2, westus, and westeurope; centralus/canadacentral/australiaeast/australiasoutheast/canadaeast/northeurope have a vhub but no ER gateway deployed. Only the one circuit above feeds traffic in — other regional gateways rely on VWAN hub-to-hub routing rather than their own circuit.

All values above (both circuits' primary/secondary/APIPA IPs, both destination CIDR sets) were cross-checked against what was already known from `IPschemas.md` and match exactly — no discrepancies.

---

## 5. AWS Direct Connect

**None. Zero Direct Connect resources of any kind** (`aws_dx_connection`, `aws_dx_gateway`, virtual interfaces, LAGs) exist anywhere in these five repos. There is no AWS equivalent to Azure's ExpressRoute here — hybrid/inter-region AWS connectivity in this codebase runs through `central-networking-core`'s AWS Cloud WAN (`cloudwan` root, `cloudwan-{np,prd}` backend state) and Network Firewall inspection VPCs instead.

---

## 5b. Updates since 2026-09-27 (checked 2026-10-08)

- **E01 CUS VNet peering added:** Corp IT D01 Central US (`vnet-corp-d01-cus`, RG `rg-network-d01-cus`, sub `cc393352-2043-4481-a89d-8a18154b539d`) in `azure-vnet-peering\env\e01\cus` (PR 571165).
- **E01 Zscaler NAT Gateway PIPs:** `e01-eus2-zscc-vwan-ngw-pip`, `e01-cus-zscc-vwan-ngw-pip` added (PR 572420); addresses not in code.
- **AWS ap-southeast-2:** GWLB routes now enabled and endpoint-service keys corrected to real AZ IDs (`apse2-az1`, `apse2-az3`). Endpoint count is unchanged; the region is moving from pending to live.
- **E01 firewall:** new t60anet01 (10.213.12.0/22) rules to ATX SQL and C8 Isilon subnets; see `IPschemas.md`.
- **Unchanged:** ExpressRoute circuits, DNS resolvers, Private Endpoints/DNS Zones (still 0), Direct Connect (still none).

---

## 6. Environment / Subscription / Account Map (consolidated)

| Item | Subscription / Account | Env tag | Region(s) |
|---|---|---|---|
| DNS Resolver (azure-dns-resolver) | E01 `7253b84c-2f1a-4b10-9046-1fbdcc4334f1` | **Test** | eastus2 only |
| DNS Resolver (azure-dns-resolver) | N01 `150ea8a8-4006-49a6-8c11-b7d7c2942e61` | **Production** | eastus2 only |
| ExpressRoute circuit `e01-eus2-to-dc3-er` | E01 | Test | eastus2 (peering @ Washington DC) |
| ExpressRoute circuit `n01-eus2-to-dc3-er` | N01 | Production | eastus2 (peering @ Washington DC) |
| ExpressRoute Gateways (vhub) | N01 | Production | eastus2, westus, westeurope deployed; centralus/canadacentral/australiaeast/australiasoutheast/canadaeast/northeurope vhub-only |
| ExpressRoute Gateway (vhub) | E01 | Test | eastus2 deployed; centralus vhub-only |
| Zscaler CC GWLB endpoints | AWS profiles `aws-prd` (+ implicit non-prod), `bbk12`, `dto`, `IDM` | prd / np / custom (KDTO, t03=test, s30=prod, IDM) | ap-southeast-2, ca-central-1, eu-west-1, eu-west-2, us-east-1, us-east-2, us-west-1, us-west-2 |
| central-networking-core GWLB endpoints | AWS accounts `070978211605` / `798924148385` = "TransitHub"; others "Decentralized" | np / prd per region (16 combos) | same 8 regions |
| Zscaler Azure Private DNS Resolver (ZPA) | Azure sub `38dc1973-b3ec-4db8-8f42-f76fc59e65b0` (d40zscc example) | inactive — `zpa_enabled` not set | eastus2 |
| Azure Private Endpoints | none found | n/a | n/a |
| Azure Private DNS Zones | none found | n/a | n/a |
| AWS Route53 Resolver (inbound) | none found | n/a | n/a |
| AWS VPC Endpoints for S3/DynamoDB/etc. | none found | n/a | n/a |
| AWS Direct Connect (any resource) | none found | n/a | n/a |

---

## 7. Open Items / Discrepancies to Resolve

1. **E01 tagged "Test," not "Education"** — both the DNS resolver and ExpressRoute circuit tfvars for E01 tag `Environment = "Test"` / `UsedFor = "Test"`, with a weaker PCI level than N01. Worth confirming with the network team whether E01 is genuinely a test-tier subscription despite being named for Education, since that changes how its config should be read operationally.
2. **Possible duplicate KDTO Zscaler-AWS GWLB config** — `zscaler-cc-terraform-aws\examples\base_cc_gwlb_asg\tfvars\terraform-kdto.tfvars` and `examples\cc_gwlb_asg\terraform.tfvars` both tag `environment_tag = "KDTO"` with `az_count = 2`. Could be the same live environment described twice across two example scaffolds, or two genuinely separate deployments — static analysis can't tell which.
3. **N01 CyberArk VNet link failed, not pending** — the commented-out `cyber-tool-vnet01` link in N01's DNS resolver tfvars carries a captured `LinkedAuthorizationFailed` error against subscription `c807483d-835a-4e97-8aa1-17a013df9663`. This was attempted and blocked, worth following up on rather than assuming it's simply unscheduled work.
4. **`ath-prd` GWLB endpoint-service entries commented out** in `central-networking-core\core\tfvars\us-east-1-prd.tfvars` alongside the active `aws-prd` ones — suggests a prior or parallel Zscaler tenant/migration that hasn't been cleaned up in code.
5. **GWLB endpoint tally (~79–81) is not de-duplicated** — the `central-networking-core` side and the `zscaler-cc-terraform-aws` side may represent the two ends (service + consumer) of the same logical connections rather than independent endpoints; confirm with the network/security team before quoting an exact total.

---

*Built by surveying the same five repos as `IPschemas.md` (`network_team_azure`, `network_team_firewall`, `network_team_terraform`, `central-networking-core`, `rdo-security-dlp-zscaler`) on 2026-09-27. Reflects Terraform/docs state, not necessarily deployed/live state — confirm against Azure Portal / AWS Console before treating any figure here as current.*
