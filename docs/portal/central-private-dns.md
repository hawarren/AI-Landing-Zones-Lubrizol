# Portal accelerator: reuse central Private DNS zones

This guide explains how the portal-based AI Landing Zone accelerator integrates
with Private DNS zones owned by a platform or hub team. It covers the failure
mode before the integration, the current deployment flow, and the network
responsibilities that remain with the platform team.

## Why central DNS needs explicit integration

In a hub-and-spoke environment, the platform team typically owns the Private DNS
zones and the spoke uses private endpoints to reach Azure services. A private
endpoint creates a private IP for a service, but clients also need DNS records
that resolve the service hostname to that private IP.

Before this change, selecting **Platform Landing Zone Integration** told the
accelerator not to create local DNS zones. The portal collected selected zone
resource IDs, but the deployment did not pass those IDs through to the AI
Foundry deployment or use them in its private endpoint DNS zone groups. AI
Foundry's nested deployment consequently had DNS configuration disabled and
could create private endpoints without associating them with the central zones.
No zone group was managing the endpoint records in the hub zones.

The endpoint could exist while its service hostname had no corresponding
private record in the central zone. The Azure admin then had to add the records
manually. If the hostname did not resolve to the private endpoint address from
the Foundry network path, Microsoft Foundry's model playground could not reach
the model endpoint and its query failed.

## What changed

The update connects the portal's selected zone resources to deployment-time DNS
configuration:

1. **Portal selection:** In Platform Landing Zone mode, the Networking step
   requires the existing zones for AI Services, OpenAI, Cognitive Services,
   Azure AI Search, Cosmos DB, Blob Storage, and Key Vault. It also collects
   zones for optional services when those services are enabled. Azure AI Search
   now has an explicit zone selector as well.
2. **Pass the resource IDs:** The form outputs each selected zone's full Azure
   resource ID. The parent ARM template maps the AI Foundry-related IDs by DNS
   suffix and passes them to the AI Foundry wrapper as
   `existingDnsZoneResourceIds`. The IDs are kept separate from the older
   resource-group-name map; a full ID can correctly identify a zone in another
   resource group or subscription.
3. **Configure Foundry private endpoints:** The wrapper forwards the IDs to
   its private endpoint deployment. When an ID is supplied, the wrapper uses it
   as the `privateDnsZoneId` for the endpoint's DNS zone group. Azure manages
   the endpoint DNS records through that zone group instead of requiring an
   administrator to create those records by hand.
4. **Avoid taking ownership of hub DNS:** The wrapper skips local zone creation
   and automatic VNet-link creation for zones supplied by resource ID. The
   parent template likewise uses the selected zone IDs in the zone groups for
   the Key Vault, Cosmos DB, Blob Storage, App Configuration, and Container
   Registry private endpoints when those services are deployed. In Platform
   Landing Zone mode, these endpoint deployments are enabled when the
   corresponding existing zone ID is provided.

## Deployment flow

```text
Portal zone selectors
        │ full zone resource IDs
        ▼
Portal form outputs → parent ARM template
        ├─ AI Services / OpenAI / Cognitive Services / Search / Blob / Cosmos / Key Vault
        │        └─ AI Foundry wrapper → private endpoint DNS zone groups
        └─ Key Vault / Cosmos / Blob / App Configuration / Container Registry
                 └─ parent-template private endpoint DNS zone groups
                                  │
                                  ▼
                     platform-owned hub DNS zones
                     (zone groups manage endpoint records)
```

The deployment associates private endpoints with the selected zones; it does
not make the hub's DNS zones or network path available to the spoke by itself.
The platform/network owners remain responsible for:

- Linking each shared zone to the spoke VNet, or providing an approved resolver
  and forwarding path.
- Maintaining routing, peering, and network access between the caller and the
  private endpoint.
- Granting the deployment identity the cross-scope permissions needed to
  associate the endpoint with each zone. Confirm the required permissions with
  the zone owner before deploying.
- Ensuring selected IDs point to the correct Azure Private DNS zone for each
  service and cloud environment.

VNet peering alone does not propagate Private DNS zone links. A successful
deployment also does not prove that the Foundry model playground's network path
can resolve or reach the model endpoint; validate resolution and connectivity
from the intended client path.

## Operational result

With the relevant zone IDs selected and permissions in place, deployment
associates the private endpoint with its central DNS zone and lets Azure manage
the endpoint record through the zone group. This removes the per-deployment
manual record-entry step that previously blocked model playground queries.
The platform team continues to own the shared zones and their VNet links; the
accelerator does not create duplicate zones or add links to centrally managed
zones.

For the broader hub/spoke ownership and DNS resolution contract, see the
[hub-and-spoke network reference](../bicep/hub-and-spoke.md#private-dns-ownership).
