# Graded Lab Activity #6

**Private access to an Azure Storage account with Azure Private Link and a Private Endpoint**

| | |
|---|---|
| **Name** | Hye Ran Yoo |
| **Student number** | 041145212 |
| **Course** | CST8912 – Cloud Solution Architecture |
| **Section** | 013 |
| **Date** | October 21, 2026 |

## What this lab does

I make a storage account that is closed to the internet and open only inside my virtual network.

1. Create a VNet with a Bastion host.
2. Create a storage account and turn off public network access.
3. Create a private endpoint for the blob service. It gets a private IP in my subnet.
4. Create a Windows VM with no public IP in the same subnet.
5. Connect to the VM with Bastion and test: `nslookup` returns the private IP, and Storage Explorer opens the container.
6. Delete everything.

```mermaid
%%{init: {"flowchart": {"nodeSpacing": 25, "rankSpacing": 40, "padding": 6}, "themeVariables": {"fontSize": "15px"}}}%%
flowchart TB
    LAPTOP["<b>My laptop</b><br/>Azure portal"]
    BAS["<b>Bastion8912</b> (A)<br/>Basic · public IP: public-ip<br/>AzureBastionSubnet 10.0.1.0/26"]
    subgraph SUB1["Subnet-1-8912 · 10.0.0.0/24"]
        VM["<b>8912-VM1</b> (E)<br/>Windows Server 2022<br/>10.0.0.5 · no public IP · nsg-1"]
        PE["<b>Privateendpoint-8912</b> (D)<br/>NIC 10.0.0.4 · target: blob"]
        DNS["<b>Private DNS zone</b> (D)<br/>privatelink.blob.core.windows.net<br/>storage18912hry = 10.0.0.4"]
    end
    ST["<b>storage18912hry</b> (B, C, G)<br/>Standard · LRS<br/>public network access: Disabled<br/>from the internet: 403<br/>container: container"]
    LAPTOP ==>|"HTTPS 443"| BAS
    BAS ==>|"RDP, private IP"| SUB1
    VM ==>|"Storage Explorer"| PE
    VM -.->|"nslookup"| DNS
    PE ==>|"Private Link"| ST
    classDef edge fill:#E3F0FC,stroke:#0078D4,stroke-width:2px,color:#0B2A4A
    classDef vm fill:#FFF1E0,stroke:#D9480F,stroke-width:2px,color:#4A1F00
    classDef store fill:#E5F6EC,stroke:#2B8A3E,stroke-width:2px,color:#0E3B1B
    classDef dns fill:#F3E8FF,stroke:#6F42C1,stroke-width:2px,color:#2E1A47
    classDef client fill:#FFFFFF,stroke:#555555,stroke-width:2px,color:#222222
    class BAS,PE edge
    class VM vm
    class ST store
    class DNS dns
    class LAPTOP client
    style SUB1 fill:#F7FBFF,stroke:#0078D4,stroke-dasharray:4 3,color:#0B2A4A
```

The subnet `Subnet-1-8912` and `AzureBastionSubnet` are in the VNet `8912-vnet1` (10.0.0.0/16). All resources are in the resource group `CST8912demo` in Canada Central. The letters (A) to (G) are the sections of the instructions.

## Changes from the instructions

| Instruction | What I did | Why |
|---|---|---|
| Storage account name `Storage1-8912` | `storage18912hry` | Storage names allow only lowercase letters and numbers. `storage18912` was already taken, so I added my initials. |
| VNet name `8912-vnet1` (step A-3) and `8912-vnet` (steps D-7, E-5) | `8912-vnet1` everywhere | The instructions use two names. I used the name I created. |
| VM size `B1` | `Standard_B2ls_v2` (2 vCPU, 4 GiB) | B1 sizes are not available for my subscription (`NotAvailableForSubscription`). The lab instructor said on September 18 that we can use another available size. My first try with `B2als_v2` failed with `AllocationFailed`, so I used `B2ls_v2`. |
| Subnet option "Enable private subnet" (not in the instructions) | Turned off | The portal now creates new subnets as private subnets with no outbound access. I unchecked the option when I created the VNet (step A-9), but the subnet was still created as a private subnet. The VM could not download Storage Explorer (step H-8), so I turned the option off on the subnet and restarted the VM. |

Step numbers below follow the sections of the instructions: for example, A-3 is section A ("Create a Virtual network and Bastion Host"), step 3.

## A. Create a virtual network and Bastion host (steps A-1 to A-11)

**Steps A-1 to A-3. Virtual networks → Create → Basics tab.**

| Setting | Value |
|---|---|
| Subscription | Azure for Students |
| Resource group | `CST8912demo` (new) |
| Name | `8912-vnet1` |
| Region | Canada Central |

![VNet Basics](screenshots/01a-vnet-basics.png)

**Steps A-4 to A-6. Security tab → Enable Bastion.**

| Setting | Value |
|---|---|
| Enable Azure Bastion | checked |
| Azure Bastion host name | `Bastion8912` |
| Azure Bastion public IP address | Create a public IP address → `public-ip` |

![VNet Security tab with Bastion](screenshots/01b-vnet-security-bastion.png)

**Steps A-7 to A-10. IP Addresses tab → edit the default subnet.**

| Setting | Value |
|---|---|
| Subnet template | Default |
| Name | `Subnet-1-8912` |
| IPv4 address range | 10.0.0.0/16 (not changed) |
| Starting address | 10.0.0.0 (not changed) |
| Subnet size | /24, 256 addresses (not changed) |
| Enable private subnet | unchecked here, but the subnet was still created as private (see step H-8) |

- The portal added `AzureBastionSubnet` (10.0.1.0/26) by itself when I enabled Bastion.

![Edit subnet](screenshots/01c-vnet-edit-subnet.png)

![Subnet list](screenshots/01d-vnet-address-space.png)

**Step A-11. Review + create → Create.**

![VNet review](screenshots/01e-vnet-review-create.png)

**Result.**
- `8912-vnet1` has 2 subnets. DNS servers: Azure provided DNS service.
- `Bastion8912` is in `AzureBastionSubnet`, tier Basic, with the public IP `public-ip`.

![VNet overview](screenshots/01f-vnet-overview-created.png)

![Bastion overview](screenshots/01g-bastion-overview.png)

- The Bastion configuration page shows SKU Basic with 2 instances. The free Developer SKU is also listed, but it does not use a public IP, so it does not match the instructions.

![Bastion configuration](screenshots/01h-bastion-configuration.png)

## B. Create a storage account (steps B-1 to B-5)

**Steps B-1 to B-3. Storage accounts → Create → Basics tab.**

| Setting | Value |
|---|---|
| Resource group | `CST8912demo` |
| Storage account name | `storage18912hry` |
| Region | Canada Central |
| Primary service | Azure Blob Storage or Azure Data Lake Storage |
| Performance | Standard |
| Redundancy | Locally-redundant storage (LRS) |

- `storage18912` was already taken, so I used `storage18912hry`.

![Storage name already taken](screenshots/02a-storage-name-taken.png)

![Storage Basics](screenshots/02a-storage-basics.png)

**Steps B-4 to B-5. Review → Create.**

![Storage review](screenshots/02b-storage-review.png)

**Result.**
- `storage18912hry` is a StorageV2 account, Standard, LRS, in canadacentral.
- At this point Public network access is Enabled and Private endpoint connections is 0.

![Storage overview after creation](screenshots/02c-storage-overview-created.png)

## C. Disable public access to the storage account (steps C-1 to C-5)

**Steps C-1 to C-5. Storage account → Security + networking → Networking → Public network access → Disabled → Save.**

- In my portal this setting is on the Public access tab. I clicked Manage and chose Disable.

![Choose Disable](screenshots/03a-storage-public-access-disable.png)

![Public network access is Disabled](screenshots/03b-storage-public-access-disabled.png)

## D. Create the private endpoint (steps D-1 to D-10)

**Steps D-1 to D-4. Private endpoints → Create → Basics tab.**

| Setting | Value |
|---|---|
| Resource group | `CST8912demo` |
| Name | `Privateendpoint-8912` |
| Network Interface Name | `Privateendpoint-8912-nic` (default) |
| Region | Canada Central |

![Private endpoint Basics](screenshots/04a-pe-basics.png)

**Steps D-5 to D-6. Resource tab.**

| Setting | Value |
|---|---|
| Connection method | Connect to an Azure resource in my directory |
| Resource type | Microsoft.Storage/storageAccounts |
| Resource | `storage18912hry` |
| Target sub-resource | blob |

![Private endpoint Resource tab](screenshots/04b-pe-resource.png)

**Step D-7. Virtual Network tab.**

| Setting | Value |
|---|---|
| Virtual network | `8912-vnet1` |
| Subnet | `Subnet-1-8912` |
| Network policy for private endpoints | edit → Network security groups and Route Tables checked → Save |
| Private IP configuration | Dynamically allocate IP address |

![Edit subnet network policy](screenshots/04c-pe-network-policy.png)

![Private endpoint Virtual Network tab](screenshots/04d-pe-virtual-network.png)

**Steps D-8 to D-9. DNS tab, defaults.**
- Integrate with private DNS zone: Yes. Private DNS zone: `privatelink.blob.core.windows.net` (new).

![Private endpoint DNS tab](screenshots/04e-pe-dns.png)

**Step D-10. Review + create → Create.**

![Private endpoint review](screenshots/04f-pe-review-create.png)

![Private endpoint deployment complete](screenshots/04g-pe-deployment-complete.png)

**Result.**
- Connection status: Approved (auto-approved). Network interface: `Privateendpoint-8912-nic`. Private IP: **10.0.0.4**.
- DNS configuration: `storage18912hry.privatelink.blob.core.windows.net` → 10.0.0.4 in the zone `privatelink.blob.core.windows.net`.
- The storage account now shows Public network access: Disabled and Private endpoint connections: 1.

![Private endpoint overview](screenshots/04h-pe-overview.png)

![Private endpoint DNS configuration](screenshots/04i-pe-dns-configuration.png)

![Storage overview after the private endpoint](screenshots/02d-storage-overview-after-pe.png)

## E. Create the test virtual machine (steps E-1 to E-7)

**Steps E-1 to E-3. Virtual machines → Create → Basics tab.**

| Setting | Value |
|---|---|
| Resource group | `CST8912demo` |
| Virtual machine name | `8912-VM1` |
| Region | Canada Central |
| Availability options | No infrastructure redundancy required |
| Security type | Standard |
| Image | Windows Server 2022 Datacenter - x64 Gen2 |
| VM architecture | x64 |
| Size | `Standard_B2ls_v2` (see below) |
| Authentication | Username and password |
| Public inbound ports | None |

- VM size: I checked the sizes in Cloud Shell first. `Standard_B1s` is `NotAvailableForSubscription`. The B2 v2 sizes are available.

![VM sizes in Cloud Shell](screenshots/05a-vm-size-check-cloudshell.png)

![Size B2ls_v2 selected](screenshots/05a3-vm-size-select-b2ls.png)

![VM Basics](screenshots/05b-vm-basics.png)

![Inbound ports None and licensing](screenshots/05c-vm-inbound-ports-licensing.png)

**Steps E-4 to E-5. Networking tab.**

| Setting | Value |
|---|---|
| Virtual network | `8912-vnet1` |
| Subnet | `Subnet-1-8912 (10.0.0.0/24)` |
| Public IP | None |
| NIC network security group | Advanced |
| Configure network security group | Create new → `nsg-1`, defaults |

![Create nsg-1](screenshots/05d1-vm-nsg-create.png)

![VM Networking tab](screenshots/05d-vm-networking.png)

**Steps E-6 to E-7. Review + create → Create.**

![VM review, part 1](screenshots/05e-vm-review-create-1.png)

![VM review, part 2](screenshots/05e-vm-review-create-2.png)

![VM review, part 3](screenshots/05e-vm-review-create-3.png)

**First try failed.**
- My first VM with size `B2als_v2` failed: `AllocationFailed` – no capacity for this size in the region at that time. `nsg-1` was created in this try.
- I deleted the failed VM and its network interface, and created the VM again with `B2ls_v2` and the existing `nsg-1`.

![First VM deployment failed](screenshots/05f-try1-vm-deploy-failed.png)

![Error details: AllocationFailed](screenshots/05f-try1-allocation-failed-error.png)

**Result.**
- `8912-VM1` is Running, size `Standard_B2ls_v2`, private IP **10.0.0.5**, no public IP, in `8912-vnet1/Subnet-1-8912`.

![VM overview](screenshots/05f-vm-overview-created.png)

## F. Storage access key (steps F-1 to F-4)

**Steps F-1 to F-4. Storage account → Security + networking → Access keys → key1 → Show → copy the Connection string.**

- I copied the connection string of key1. I use it in step H-13. It is not in this report.

![Access keys](screenshots/06-storage-access-keys.png)

## G. Add a blob container (steps G-1 to G-6)

**Steps G-1 to G-6. Storage account → Data storage → Containers → + Container → name `container`, Private (no anonymous access) → Create.**

- The container `container` is in the list with Anonymous access level Private. `$logs` is created by Azure.

![Container list](screenshots/07b-container-list.png)

## H. Test connectivity to the private endpoint (steps H-1 to H-20)

**Steps H-1 to H-5. Virtual machine → Bastion → username and password → Connect.**

- The Bastion page shows `Bastion8912`, SKU Basic. Authentication type VM Password, username `hyeran8912`.
- The VM desktop opened in a new browser tab at `bastion.azure.com`. The VM has no public IP.

![Bastion connect page](screenshots/08a0-bastion-connect-form.png)

![Connected to the VM through Bastion](screenshots/08a-bastion-connected.png)

**Steps H-6 to H-7. Windows PowerShell → nslookup.**

```text
PS C:\Users\hyeran8912> nslookup storage18912hry.blob.core.windows.net
Server:  UnKnown
Address:  168.63.129.16

Non-authoritative answer:
Name:    storage18912hry.privatelink.blob.core.windows.net
Address:  10.0.0.4
Aliases:  storage18912hry.blob.core.windows.net
```

- The storage account name resolves to the private IP **10.0.0.4**. This is the private endpoint in `Subnet-1-8912`.
- `168.63.129.16` is the Azure provided DNS. It uses the private DNS zone from step D-9.

![nslookup returns the private IP](screenshots/08b-nslookup-private-ip.png)

**Step H-8. Install Microsoft Azure Storage Explorer on the virtual machine.**

*Problem: the VM had no internet.*
- The download page did not open. `Test-NetConnection www.microsoft.com -Port 443` returned `TcpTestSucceeded : False`.
- Cause: `Subnet-1-8912` was a private subnet (no default outbound access). The portal sets this by default for new VNets.
- Fix: I unchecked "Enable private subnet" on the subnet, saved, and did Stop and Start on the VM. The same test then returned `True`.
- This does not change the private endpoint test. The storage account is still closed to the internet.

![No internet from the VM](screenshots/08x-vm-no-internet-test.png)

![Subnet was private](screenshots/08y-subnet-private-enabled.png)

![Private subnet turned off](screenshots/08z-subnet-private-disabled.png)

![Internet works after the restart](screenshots/08z2-vm-internet-ok.png)

*Install.*
- In the VM I downloaded `StorageExplorer-windows-x64.exe` from the Microsoft page and installed it. The setup also installed .NET.

![Storage Explorer downloaded in the VM](screenshots/08c1-storage-explorer-downloaded.png)

**Step H-9. Finish and open Storage Explorer.**

![Storage Explorer opened](screenshots/08c-storage-explorer-launched.png)

**Steps H-10 to H-11. Power plug → Select Resource → Storage account or service.**

![Select Resource](screenshots/08d-se-select-resource.png)

**Step H-12. Connection string → Next.**

![Connection method](screenshots/08e-se-connection-method.png)

**Steps H-13 to H-14. Paste the connection string → Next.**
- The display name was filled in from the account name.

![Connection string entered](screenshots/08f-se-connection-string.png)

**Steps H-15 to H-16. Summary → Connect.**

![Summary](screenshots/08g-se-summary.png)

**Steps H-17 to H-19. Storage Accounts → expand the account → Blob Containers.**
- `container` is there. It opens with no error. It is empty because I did not upload files.
- Activities shows "Successfully added new connection".

![Blob container in Storage Explorer](screenshots/08h-se-blob-container.png)

- For comparison, from my laptop the same container gives error 403 "This request is not authorized", because public network access is disabled.

![Same container from the laptop: 403](screenshots/08i-laptop-container-403.png)

**Step H-20. Close the connection to 8912-VM1.**

![Disconnected from the VM](screenshots/08j-bastion-disconnected.png)

## I. Clean up and document (step H-21)

**Step H-21. Clean all the resources created during the lab and document all the steps using screenshots.**

- Before deleting, the resource group had 11 resources (12 with the hidden virtual network link of the DNS zone).

![Resources before deletion](screenshots/09-0-rg-resources-before-delete.png)

- I deleted the resource group `CST8912demo`.

![Delete resource group confirmation](screenshots/09a-rg-delete-confirm.png)

- The notification shows "Deleted resource group CST8912demo". The deletion took about 15 minutes because of the Bastion host.

![Deleted resource group notification](screenshots/09b-rg-deleted-notification.png)

- After the deletion, only `NetworkWatcherRG` is left. It was there before the lab, so I did not touch it.

![Resource groups after deletion](screenshots/09c-rg-list-after-delete.png)

## Findings and analysis

- **Public access off first.** After step C, the storage account did not accept requests from the internet. From my laptop, the container page showed error 403.
- **Same name, private address.** Inside the VNet, `storage18912hry.blob.core.windows.net` resolved to 10.0.0.4 through `storage18912hry.privatelink.blob.core.windows.net`. The private DNS zone from the DNS tab did this. The application does not need a new name.
- **A private endpoint is a network interface.** It took the first free IP in `Subnet-1-8912` (10.0.0.4). The VM got the next one (10.0.0.5). Storage Explorer on the VM used the normal account name, the name resolves to 10.0.0.4, and public access is off. So the container was opened through the private endpoint.
- **The VM has no public IP.** I connected with Azure Bastion in the browser. Only Bastion has a public IP. Bastion Basic is charged by the hour, so I did the lab in one session.
- **Network policy.** I turned on network security groups and route tables for private endpoints in the subnet, as the instructions say. The portal explains that this is needed if NSG or route table rules should also apply to the private endpoint. This subnet has no NSG or route table, so I did not see a change in this lab.
- **Private subnet.** The portal now creates new subnets as private subnets by default. A private subnet blocks default outbound access, so the VM could not reach the internet to download Storage Explorer. The `nslookup` test already returned the private IP before this change, so the private endpoint did not depend on it. I turned the option off only for the download.
- **Clean up.** One resource group held all the resources (11 in the list, 12 with the hidden virtual network link). Deleting the group removed the VM, the Bastion host, the public IP, the VNet, the private endpoint, the DNS zone and the storage account in one step.
