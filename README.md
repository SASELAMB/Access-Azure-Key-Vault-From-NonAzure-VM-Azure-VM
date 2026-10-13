<<draft>
# Azure Key Vault from a Non‑Azure VM using Azure Arc (Hybrid, Private Endpoint)

> Demo: a .NET console application running on an **on‑premises (non‑Azure) Windows machine** reads secrets from **Azure Key Vault** with **no passwords, client secrets or certificates in code**. Authentication uses the **Azure Arc‑enabled server's system‑assigned managed identity**. Traffic stays private over VPN / ExpressRoute to a **Key Vault Private Endpoint**.

![Azure](https://img.shields.io/badge/Azure-Key%20Vault-0078D4?logo=microsoftazure)
![Azure Arc](https://img.shields.io/badge/Azure%20Arc-Enabled%20Server-0078D4)
![.NET](https://img.shields.io/badge/.NET-6.0-512BD4?logo=dotnet)
![PowerShell](https://img.shields.io/badge/PowerShell-5.1%2B-5391FE?logo=powershell)

---

## Table of contents
- [Why this pattern](#why-this-pattern)
- [Architecture](#architecture)
- [How authentication works](#how-authentication-works)
- [Prerequisites](#prerequisites)
- [Step 1 – Create the Key Vault and secrets](#step-1--create-the-key-vault-and-secrets)
- [Step 2 – Private Endpoint and DNS](#step-2--private-endpoint-and-dns)
- [Step 3 – Onboard the on‑prem server to Azure Arc](#step-3--onboard-the-on-prem-server-to-azure-arc)
- [Step 4 – Grant the Arc machine access to Key Vault (RBAC)](#step-4--grant-the-arc-machine-access-to-key-vault-rbac)
- [Step 5 – Verify the managed identity token (PowerShell)](#step-5--verify-the-managed-identity-token-powershell)
- [Step 6 – Console application (.NET)](#step-6--console-application-net)
- [Validation](#validation)
- [Troubleshooting](#troubleshooting)
- [Security notes](#security-notes)
- [References](#references)

---

## Why this pattern

Applications on servers outside Azure usually reach Key Vault with a **service principal + client secret/certificate**. That secret becomes one more thing to store, rotate and protect.

With **Azure Arc‑enabled servers**, the on‑prem machine gets a **managed identity in Microsoft Entra ID** (formerly Azure AD), just like an Azure VM:

| Approach | Secret on the server? | Rotation effort | Private network |
|---|---|---|---|
| Service principal + client secret | ✅ Yes | Manual | Optional |
| Service principal + certificate | ✅ Yes (cert) | Manual | Optional |
| **Azure Arc managed identity (this repo)** | ❌ **No** | **Automatic** | **Private Endpoint** |

---

## Architecture

```
                       ┌──────────────────────────────────────────────────────┐
                       │  Contoso Azure Hub VNet   demo-lz-vnet-eu-01         │
                       │  (192.168.0.0/16)  – App Hub subscription, East US   │
                       │                                                      │
                       │  demo-lz-snet-01 (192.168.1.0/24)                    │
                       │   ┌─────────────┐   ┌──────────────────────────────┐ │
                       │   │  AKV        │   │ DNS Private Resolver         │ │
                       │   │  Private    │   │ (inbound endpoint)           │ │
                       │   │  Endpoint   │   └──────────────────────────────┘ │
                       │   └──────▲──────┘   Private DNS zone:                │
                       │          │ HTTPS 443  privatelink.vaultcore.azure.net│
                       │  GatewaySubnet (192.168.0.0/24)                      │
                       │   ┌──────┴──────┐                                    │
                       │   │ VPN Gateway │  demo-vnetgw-eu-01                 │
                       │   └──────▲──────┘                                    │
                       └──────────┼───────────────────────────────────────────┘
                                  │  S2S VPN / ExpressRoute
 ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
                                  │
                       ┌──────────┴───────────────────────────────────────────┐
                       │  Contoso On‑prem                                     │
                       │   Proxy (443) ──► Web Server  [Azure Arc agent]      │
                       │                   └─ ARCAKVConsoleApp (.NET)         │
                       │                   DB Server   [Azure Arc agent]      │
                       │   AD DS DNS: conditional forwarder                   │
                       │     vault.azure.net ──► DNS Private Resolver inbound │
                       └──────────────────────────────────────────────────────┘
```

**Key building blocks**

| Component | Purpose |
|---|---|
| **Azure Key Vault** (`demoapp-dev-akv-001`, Standard, RBAC) | Stores the secrets (`Secret`, `CertSecret`) |
| **Private Endpoint** in the hub VNet | Gives Key Vault a private IP address (e.g. `192.168.1.6`) |
| **Private DNS zone** `privatelink.vaultcore.azure.net` | Resolves the vault name to the private IP |
| **DNS Private Resolver** (inbound endpoint) | Lets on‑prem DNS resolve the Azure private DNS zone |
| **On‑prem DNS conditional forwarder** | Forwards `vault.azure.net` to the resolver's inbound IP |
| **VPN Gateway / ExpressRoute** | Private connectivity between on‑prem and Azure |
| **Azure Arc Connected Machine agent** | Projects the server into Azure and gives it a managed identity |
| **ARCAKVConsoleApp** | .NET console app that reads secrets with `Azure.Identity` |

---

## How authentication works

1. The console app asks the **local Arc agent identity endpoint** (`http://localhost:40342`) for a token.
2. The agent replies with a **401 challenge** that points to a key file which **only local administrators** can read (proof the caller is on the machine).
3. The app sends the request again with the key file content and receives an **Entra ID access token** for `https://vault.azure.net`.
4. The app calls Key Vault (`GetSecret`) through the **Private Endpoint**, presenting the token.
5. Key Vault **validates the token** with Entra ID and checks **RBAC** (*Key Vault Secrets User/Officer*).
6. Key Vault **returns the secret**.

> `Azure.Identity` (`DefaultAzureCredential` / `ManagedIdentityCredential`) does steps 1–3 automatically on an Arc‑enabled server.

---

## Prerequisites

- An Azure subscription with permission to create Key Vault, networking and role assignments
- An on‑prem / non‑Azure **Windows Server or Windows 10/11** machine with outbound access to Azure Arc endpoints (directly or through a proxy)
- Site‑to‑site **VPN or ExpressRoute** into a hub VNet (for the private‑endpoint path)
- **Visual Studio 2022** or the .NET SDK (**.NET 6+**)
- Az PowerShell module (`Install-Module Az`)
- To run the app, an account that is a **local Administrator** or a member of the **Hybrid agent extension applications** group (required to read the Arc token key file)

---

## Step 1 – Create the Key Vault and secrets

```powershell
$rg       = "demo-contoso-eu2-rg-01"
$location = "eastus2"
$kv       = "demoapp-dev-akv-001"

New-AzResourceGroup -Name $rg -Location $location
New-AzKeyVault -Name $kv -ResourceGroupName $rg -Location $location -EnableRbacAuthorization

# Sample secrets used by the console app
Set-AzKeyVaultSecret -VaultName $kv -Name "Secret"     -SecretValue (ConvertTo-SecureString "<your-value>" -AsPlainText -Force)
Set-AzKeyVaultSecret -VaultName $kv -Name "CertSecret" -SecretValue (ConvertTo-SecureString "<your-value>" -AsPlainText -Force)
```

---

## Step 2 – Private Endpoint and DNS

1. **Key Vault → Networking → Private endpoint connections → + Private endpoint**. Put it in the hub subnet (for example `demo-lz-snet-01`) and integrate it with the private DNS zone `privatelink.vaultcore.azure.net`.
2. **Key Vault → Networking → Firewalls and virtual networks**: set **Disable public access** (or *selected networks*) so that only the private endpoint is used.
3. Deploy an **Azure DNS Private Resolver** with an **inbound endpoint** in the hub VNet.
4. On the on‑prem **AD DS DNS server** (DNS Manager → *Conditional Forwarders*), add a conditional forwarder:

   | Domain | Forward to |
   |---|---|
   | `vault.azure.net` | DNS Private Resolver inbound IP |
   | `privatelink.vaultcore.azure.net` *(optional)* | DNS Private Resolver inbound IP |

5. From the on‑prem server, check resolution:

   ```powershell
   nslookup demoapp-dev-akv-001.vault.azure.net
   # Expected: an alias to demoapp-dev-akv-001.privatelink.vaultcore.azure.net -> 192.168.1.x (private IP)
   ```

---

## Step 3 – Onboard the on‑prem server to Azure Arc

**Azure portal → Azure Arc → Machines → + Add/Create → Add a machine → Add a single server → Generate script**

Choose the subscription, resource group, region, OS (**Windows**) and connectivity method (*Public endpoint*, *Proxy server* or *Private endpoint*). Then run the generated script as **Administrator** on the server. It boils down to:

```powershell
# Download and install the Azure Connected Machine agent
[Net.ServicePointManager]::SecurityProtocol = [Net.ServicePointManager]::SecurityProtocol -bor 3072
Invoke-WebRequest -UseBasicParsing -Uri "https://aka.ms/azcmagent-windows" `
    -TimeoutSec 30 -OutFile "$env:TEMP\install_windows_azcmagent.ps1"
& "$env:TEMP\install_windows_azcmagent.ps1"
if ($LASTEXITCODE -ne 0) { exit 1 }

# Connect the machine to Azure Arc
& "$env:ProgramW6432\AzureConnectedMachineAgent\azcmagent.exe" connect `
    --resource-group  "<RESOURCE_GROUP>" `
    --tenant-id       "<TENANT_ID>" `
    --location        "<LOCATION>" `
    --subscription-id "<SUBSCRIPTION_ID>" `
    --cloud           "AzureCloud"
```

Complete the browser sign‑in when prompted. The machine then appears under **Azure Arc → Machines** with status **Connected**.

```powershell
azcmagent show   # Agent status should be "Connected"
```

---

## Step 4 – Grant the Arc machine access to Key Vault (RBAC)

**Key Vault → Access control (IAM) → + Add role assignment**

| Field | Value |
|---|---|
| Role | **Key Vault Secrets User** (read only, least privilege). The demo used *Key Vault Secrets Officer*. |
| Assign access to | **Managed identity** |
| Managed identity | **Machine – Azure Arc** → select your server |

```powershell
$arc = Get-AzConnectedMachine -ResourceGroupName $rg -Name "<ARC_MACHINE_NAME>"
New-AzRoleAssignment -ObjectId $arc.IdentityPrincipalId `
    -RoleDefinitionName "Key Vault Secrets User" `
    -Scope (Get-AzKeyVault -VaultName $kv).ResourceId
```

---

## Step 5 – Verify the managed identity token (PowerShell)

Run this **elevated** on the Arc‑enabled server to confirm that the machine can obtain a Key Vault token:

```powershell
$apiVersion = "2020-06-01"
$resource   = "https://vault.azure.net"
$endpoint   = "{0}?resource={1}&api-version={2}" -f $env:IDENTITY_ENDPOINT, $resource, $apiVersion
# IDENTITY_ENDPOINT is usually http://localhost:40342/metadata/identity/oauth2/token

$secretFile = ""
try {
    Invoke-WebRequest -Method GET -Uri $endpoint -Headers @{Metadata='True'} -UseBasicParsing
} catch {
    $wwwAuthHeader = $_.Exception.Response.Headers["WWW-Authenticate"]
    if ($wwwAuthHeader -match "Basic realm=.+") {
        $secretFile = ($wwwAuthHeader -split "Basic realm=")[1]
    }
}
Write-Host "Secret file path: " $secretFile`n

$secret   = cat -Raw $secretFile
$response = Invoke-WebRequest -Method GET -Uri $endpoint `
            -Headers @{Metadata='True'; Authorization="Basic $secret"} -UseBasicParsing
if ($response) {
    $token = (ConvertFrom-Json -InputObject $response.Content).access_token
    Write-Host "Access token: " $token
}

# Optional: sign in to Az PowerShell as the machine identity
Connect-AzAccount -Identity
Get-AzKeyVaultSecret -VaultName "demoapp-dev-akv-001" -Name "Secret" -AsPlainText
```

The secret file path should look like `C:\ProgramData\AzureConnectedMachineAgent\Tokens\<guid>.key`, followed by a JWT access token.

---

## Step 6 – Console application (.NET)

### Create the project

```bash
dotnet new console -n ARCAKVConsoleApp
cd ARCAKVConsoleApp
dotnet add package Azure.Identity
dotnet add package Azure.Security.KeyVault.Secrets
```

### `Program.cs`

```csharp
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;

Console.WriteLine("Hello, World!");

// On an Azure Arc-enabled server, ManagedIdentityCredential picks up the
// Arc agent identity endpoint automatically (IDENTITY_ENDPOINT / IMDS_ENDPOINT).
var opts = new DefaultAzureCredentialOptions
{
    ExcludeManagedIdentityCredential = false,       // use the Arc managed identity
    ExcludeInteractiveBrowserCredential = true
};

var client = new SecretClient(
    new Uri("https://demoapp-dev-akv-001.vault.azure.net/"),
    new DefaultAzureCredential(opts));

// Single read
// Console.WriteLine(client.GetSecret("CertSecret").Value.Value);

// Repeated reads (demo: show the identity/token is reused)
for (int i = 0; i < 100; i++)
{
    Console.WriteLine(client.GetSecret("Secret").Value.Value);
}

Console.ReadKey();
```

> 💡 For production, use `new ManagedIdentityCredential()` directly so the app can't silently fall back to a developer or user credential.

### Run

Run **as Administrator** (or as a member of *Hybrid agent extension applications*):

```bash
dotnet run
```

Expected output:

```
Hello, World!
<secret value>
<secret value>
...
```

---

## Validation

| Check | Command / Location | Expected |
|---|---|---|
| Arc agent connected | `azcmagent show` / Portal → Azure Arc → Machines | **Connected** |
| DNS resolves privately | `nslookup demoapp-dev-akv-001.vault.azure.net` | Private IP (`192.168.1.x`) through `privatelink` |
| Token issued | Step 5 script | JWT printed |
| RBAC working | Console app | Secret value printed |
| Public access blocked | Browse to the vault's secrets from outside the network | *"You are unauthorized to view these contents"* / 403 |

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `CredentialUnavailableException` / access denied reading the `.key` file | App is not running elevated | Run as Administrator or add the account to **Hybrid agent extension applications** |
| `403 Forbidden` from Key Vault | No RBAC role, or role not yet propagated | Assign **Key Vault Secrets User** to the Arc machine identity and wait about 5 minutes |
| `403 … Public network access is disabled` | DNS resolves to the **public** IP | Fix the conditional forwarder / private DNS resolver, then flush DNS (`ipconfig /flushdns`) |
| `nslookup` returns the public IP | Forwarder missing or pointing to the wrong IP | Forward `vault.azure.net` to the DNS Private Resolver **inbound** IP |
| Timeouts on 443 | Proxy / firewall / NSG blocking | Allow 443 from on‑prem to the private endpoint subnet; configure the proxy with `azcmagent config set proxy.url` |
| Arc onboarding fails | Outbound Arc endpoints blocked | Check `azcmagent check` and the Arc network requirements |

---

## Security notes

- ✅ **No credentials in code or config**: the identity is tied to the machine and rotated by Azure.
- ✅ **Least privilege**: prefer *Key Vault Secrets User* over *Officer*/*Administrator*.
- ✅ **Private only**: disable public network access on the vault once the private endpoint works.
- ⚠️ **Any local admin on the server can get tokens** for this identity. Harden and limit admin access to the machine.
- ⚠️ **Never print secrets or tokens** in production logs. The `Console.WriteLine` of secret values and tokens is for demo only.
- ⚠️ Don't commit tenant IDs, subscription IDs or tokens. Use placeholders as in this README.

---

## Repository structure (suggested)

```
.
├── README.md
├── docs/
│   └── architecture-akv-hybrid.svg       # Architecture diagram
├── scripts/
│   ├── AzureARCOnboarding.ps1            # Arc agent install + connect
│   └── ARCAKVHybrid.ps1                  # Managed identity token test
└── src/
    └── ARCAKVConsoleApp/
        ├── ARCAKVConsoleApp.csproj
        └── Program.cs
```

---

## References

- [Azure Arc‑enabled servers overview](https://learn.microsoft.com/azure/azure-arc/servers/overview)
- [Authenticate against Azure resources with Azure Arc‑enabled servers (managed identity)](https://learn.microsoft.com/azure/azure-arc/servers/managed-identity-authentication)
- [Azure Key Vault private link](https://learn.microsoft.com/azure/key-vault/general/private-link-service)
- [Azure DNS Private Resolver](https://learn.microsoft.com/azure/dns/dns-private-resolver-overview)
- [Azure Key Vault RBAC roles](https://learn.microsoft.com/azure/key-vault/general/rbac-guide)
- [Azure.Identity client library for .NET](https://learn.microsoft.com/dotnet/api/overview/azure/identity-readme)

---

*Based on the demo recording "Azure Key Vault | Arc Enabled | Console Application | Non‑Azure VM" (25 Aug 2023), presented by Saravanan Selamban.*
