# SOP: Azure Linux VM access over Point-to-Site VPN

| Field | Value |
| --- | --- |
| Document | SOP — Create Linux VM and reach it with Azure P2S VPN |
| VNet | `leetamVNET` |
| Auth | Azure certificate |
| Client | Windows, Azure VPN Client |
| Result | VPN up; SSH to `10.0.0.4:22` succeeded |
| Source | Grok thread 2026-09-30 11:53–15:16 UTC, ChatGPT thread 2026-09-30 15:22–16:02 UTC |

This procedure is the path that was completed. It assumes the resource group, virtual network, and a VM subnet already exist. It does not use a VM public IP.

## 1. As-built values

Use these values. Do not substitute names that were only examples in the early guide.

| Item | Value | Status |
| --- | --- | --- |
| Virtual network | `leetamVNET` | Confirmed |
| VM private IP | `10.0.0.4` | Confirmed |
| P2S client address | `172.16.201.2` | Confirmed on interface `leetamVNET` |
| P2S address pool | `172.16.201.0/24` | Used; client address falls in this pool |
| Root certificate | `AzureP2SRootCert` | Confirmed |
| Client certificate | `AzureP2SClientCert` | Confirmed, exported as `.pfx` |
| VPN profile | `AzureVPN/azurevpnconfig.xml` or `azurevpnconfig_cert.xml` | Confirmed |
| Connection name in Azure VPN Client | `leetamVNET` | Confirmed |
| SSH check | `Test-NetConnection 10.0.0.4 -Port 22` → `TcpTestSucceeded : True` | Confirmed |
| Existing gateway public IPs | `VPNgtwyIP1`, `VPNgtwyIP2` | Seen in portal; already attached |
| Third public IP for User VPN | `leetamVPNgtwy-P2S-IP` | Name used in the working portal steps |
| SSH user | Chosen at VM create | Not recorded in the transcript. Do not guess. |

The gateway in this environment was an availability-zone SKU in active-active mode. That is why a third public IP was required for User VPN. A new single-instance gateway does not need that third IP.

## 2. Prerequisites

- Azure subscription access to the existing resource group and `leetamVNET`.
- An existing VM subnet in that VNet. Do not place the VM in `GatewaySubnet`.
- A free address range inside the VNet for `GatewaySubnet` if that subnet does not already exist. Minimum `/27`. No NSG on `GatewaySubnet`.
- A Windows machine for certificate creation and the Azure VPN Client.
- PowerShell run as administrator for certificate commands.
- No overlap between the VNet, `GatewaySubnet`, and the P2S pool `172.16.201.0/24`.

## 3. Create GatewaySubnet

Skip this step if `GatewaySubnet` already exists on `leetamVNET`.

1. Open the VNet → Subnets → + Subnet.
2. Set subnet purpose to Virtual network gateway. The name becomes `GatewaySubnet`.
3. Choose a free range inside the VNet, `/27` or larger. Do not overlap the VM subnet.
4. Save. Do not attach an NSG or put a VM on this subnet.

CLI form, after replacing the resource group, VNet, and prefix:

```bash
az network vnet subnet create \
  --resource-group <resource-group> \
  --vnet-name leetamVNET \
  --name GatewaySubnet \
  --address-prefix <free-range>/27
```

## 4. Create or confirm the VPN gateway

If the gateway already exists, skip creation and go to section 5.

1. Create a virtual network gateway in the same region as `leetamVNET`.
2. Gateway type: VPN. VPN type: Route-based.
3. SKU: VpnGw1 or higher. Do not use Basic.
4. Virtual network: `leetamVNET`. `GatewaySubnet` must already exist.
5. Public IP: new Standard, Static.
6. Wait until provisioning succeeds. This often takes 30–60 minutes.

This environment already had an active-active zone-redundant gateway using `VPNgtwyIP1` and `VPNgtwyIP2`. Do not reuse those two IPs for P2S.

## 5. Create the Linux VM

1. Create an Azure VM in the existing resource group and region.
2. Image: Ubuntu Server 22.04 LTS, or the image you actually need.
3. Authentication: SSH public key. Record the admin username. The transcript does not record it.
4. Networking:
   - Virtual network: `leetamVNET`
   - Subnet: the existing VM subnet, not `GatewaySubnet`
   - Public IP: None
   - NSG: none, or a rule that allows TCP 22 from the VNet / P2S pool `172.16.201.0/24`
5. After creation, record the private IP. In this build it was `10.0.0.4`.

## 6. Generate certificates on Windows

Run PowerShell as administrator. One root certificate is uploaded to Azure. Each client machine gets its own client certificate signed by that root. The same client certificate can be copied, but a unique certificate per machine is the preferred pattern.

VpnGw1 supports 250 P2S connections. Higher SKUs support more. That limit is on the gateway, not on the VM.

### 6.1 Root certificate

```powershell
$root = New-SelfSignedCertificate -Type Custom -KeySpec Signature `
  -Subject "CN=AzureP2SRootCert" -KeyExportPolicy Exportable `
  -HashAlgorithm sha256 -KeyLength 2048 `
  -CertStoreLocation "Cert:\CurrentUser\My" `
  -KeyUsageProperty Sign -KeyUsage CertSign
```

Export the public part only. Azure never receives the root private key.

```powershell
$cerPath = "$env:USERPROFILE\Desktop\AzureP2SRootCert.cer"
Export-Certificate -Cert $root -FilePath $cerPath -Type CERT
```

Open the `.cer` in a text editor only if you exported Base-64. If the file is binary, export again as Base-64 from `certmgr.msc`:

1. `certmgr.msc` → Personal → Certificates.
2. `AzureP2SRootCert` → All Tasks → Export.
3. No, do not export the private key.
4. Base-64 encoded X.509 (`.cer`).
5. Copy the text between `BEGIN CERTIFICATE` and `END CERTIFICATE`. That is the value uploaded to Azure.

### 6.2 Client certificate

```powershell
$client = New-SelfSignedCertificate -Type Custom -KeySpec Signature `
  -Subject "CN=AzureP2SClientCert" -KeyExportPolicy Exportable `
  -HashAlgorithm sha256 -KeyLength 2048 `
  -CertStoreLocation "Cert:\CurrentUser\My" `
  -Signer $root `
  -TextExtension @("2.5.29.37={text}1.3.6.1.5.5.7.3.2")
```

Export the client certificate as a `.pfx` if another machine must use it. The password is chosen at export time and is not stored in this SOP.

```powershell
$pwd = Read-Host "PFX password" -AsSecureString
Export-PfxCertificate -Cert $client `
  -FilePath "$env:USERPROFILE\Desktop\AzureP2SClientCert.pfx" `
  -Password $pwd
```

Certificates created this way are already in Current User → Personal. `certmgr.msc` is the viewer, not a required upload step for the machine that created them. On another machine, double-click the `.pfx`, enter the password, and install it into Current User → Personal.

## 7. Configure Point-to-Site

1. Open the VPN gateway → Point-to-site configuration.
2. Address pool: `172.16.201.0/24`. It must not overlap the VNet or the local network.
3. Tunnel type: OpenVPN (SSL), or IKEv2 and OpenVPN (SSL). The working client profile was the Azure VPN Client certificate profile.
4. Authentication: Azure certificate.
5. Root certificates: name `AzureP2SRootCert`, public data from the Base-64 `.cer`.
6. Save.

### 7.1 Active-active zone gateway only

If the portal says a third public IP is required because the gateway is an availability-zone SKU in active-active mode:

1. Do not select `VPNgtwyIP1` or `VPNgtwyIP2`. Both are already used by the gateway instances.
2. Create a new Standard, Static public IP named `leetamVPNgtwy-P2S-IP`.
3. If "Create new" inside the P2S blade fails, create that public IP first, then choose it as an existing IP under Public IP address for User VPN configuration.
4. Confirm the address pool and root certificate are still filled, then Save.

Download `vpnclientconfiguration.zip` after the configuration succeeds.

## 8. Install the client and connect

1. Install Azure VPN Client from the Microsoft Store, or from https://aka.ms/azvpnclientdownload.
2. Confirm `AzureP2SClientCert` is in Current User → Personal on this Windows machine.
3. Extract the ZIP. It contains `AzureVPN`, `Generic`, and `OpenVPN`.
4. Use `AzureVPN`. Do not use the built-in Windows VPN, Generic, or OpenVPN for this setup.
5. In Azure VPN Client, choose Import and select:
   - `AzureVPN/azurevpnconfig.xml`, or
   - `AzureVPN/azurevpnconfig_cert.xml`
6. Certificate information: `AzureP2SClientCert`.
7. Save. The profile name is `leetamVNET`.
8. Connect.

Expected result: status connected, client address in `172.16.201.0/24`. This build received `172.16.201.2` on interface `leetamVNET`.

## 9. Verify and SSH

In PowerShell on the connected Windows client:

```powershell
Test-NetConnection 10.0.0.4 -Port 22
```

Passing result from this build:

```text
ComputerName     : 10.0.0.4
RemoteAddress    : 10.0.0.4
RemotePort       : 22
InterfaceAlias   : leetamVNET
SourceAddress    : 172.16.201.2
TcpTestSucceeded : True
```

`InterfaceAlias` must be `leetamVNET`. If it is Wi-Fi or Ethernet, the test did not go through the tunnel.

Then SSH with the username and key created with the VM:

```powershell
ssh <username>@10.0.0.4
ssh -i <private-key> <username>@10.0.0.4
```

No public IP is required for this path.

## 10. Add another Windows client

1. On the original machine, create another client certificate signed by `AzureP2SRootCert`, or export the existing `.pfx`.
2. Install that `.pfx` into Current User → Personal on the new machine.
3. Install Azure VPN Client and import the same `AzureVPN/azurevpnconfig.xml`.
4. Select that machine's client certificate and connect.
5. Repeat the `Test-NetConnection` check. The source address will differ inside `172.16.201.0/24`.

Do not upload client certificates to Azure. Only the root public certificate belongs on the gateway.

## 11. Failure checks

| Symptom | Check |
| --- | --- |
| P2S save blocked on active-active gateway | Third Standard static public IP created and selected. Root public data pasted. |
| Import fails | File is from the `AzureVPN` folder, not Generic or OpenVPN. |
| Connect has no certificate | `AzureP2SClientCert` is in Current User → Personal, and it chains to `AzureP2SRootCert`. |
| Connected but SSH fails | Source address is in `172.16.201.0/24` and interface is `leetamVNET`. VM private IP is still `10.0.0.4`. NSG or host firewall allows TCP 22 from the P2S pool. |
| `TcpTestSucceeded` false on another interface | VPN is not the selected route. Disconnect other VPNs and reconnect `leetamVNET`. |

## 12. Out of scope

- Cert password. It was chosen at `.pfx` export and is not in the source threads.
- VM admin username and SSH private-key path. Not recorded.
- Resource group name, VM name, and gateway name. Not recorded as final values.
- Public SSH. This SOP keeps the VM without a public IP.
