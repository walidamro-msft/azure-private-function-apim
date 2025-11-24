
# Deploying a Private Azure Function App with APIM (External) Using VNET + Private Endpoints

This guide provides **step-by-step instructions** to deploy a **fully private Azure Function App**, accessible **only through Azure API Management (APIM)** using **VNET Integration**, **Private Endpoints**, **NSGs**, and **DNS Zones**.

This document reflects the exact configuration deployed in:

- **Resource Group:** `rg-private-apps`
- **VNET:** `vnet-apps-private`
- **Function App:** `fa-private-test01`
- **APIM:** `apim-my-apps-0001`
- **Storage Account:** `saprivatefunctions2`
- **App Service Plan:** `asp-private-funxction-app`

---

# **📘 Architecture Overview**

```
Internet
   |
   | (Public only through APIM)
   v
Azure API Management (External mode)
   |
   |-- VNET Integration into subnet-apim
   v
VNET: vnet-apps-private
   |
   |-- Private Endpoint → Function App
   |-- Private Endpoint → Storage Account (blob, queue)
   |-- Private DNS Zones
```

The Function App is fully private and cannot be accessed publicly.  
APIM exposes a **public API endpoint**, but it connects privately to the Function App through the VNET.

---

# **1. Create Resource Group**

```
rg-private-apps
```

---

# **2. Create the Virtual Network**

### VNET
- **Name:** `vnet-apps-private`
- **Address Space:** `10.0.0.0/16`

### Subnets created:
| Subnet Name | Address Prefix | Purpose |
|-------------|----------------|---------|
| `default` | 10.0.0.0/24 | General |
| `subnet-private-endpoints` | 10.0.1.0/24 | Private endpoints |
| `subnet-apim` | 10.0.2.0/24 | APIM VNET integration |

Ensure **no NSG** is automatically created for PE subnet.

---

# **3. Create the Storage Account (General Purpose v2)**

- **Name:** `saprivatefunctions2`
- **Kind:** StorageV2
- **Replication:** LRS
- **Public network access:** **Enabled temporarily**

You must create the storage account **before** creating the Function App in VNET-restricted scenarios.

---

# **4. Create the Function App (Plan: B1)**

### App Service Plan
- **Name:** `asp-private-funxction-app`
- **SKU:** **B1** (supports VNET Integration + Private Endpoints)
- **OS:** Windows
- **Region:** East US

### Function App
- **Name:** `fa-private-test01`
- **Runtime:** .NET isolated
- **Storage Account:** `saprivatefunctions2`

After the app is created, deploy your function code so that APIM can detect the API operations.

---

# **5. Create Private Endpoints**

Private Endpoints should be deployed into:

```
subnet-private-endpoints
```

### Create the following:

#### 5.1 Function App Private Endpoint
- **Name:** `pep-fa-private-test01`
- **Target Resource:** fa-private-test01
- **Subnet:** subnet-private-endpoints
- **Private DNS Zone:** `privatelink.azurewebsites.net`

#### 5.2 Storage Account Private Endpoints
You must create **one private endpoint per service**:

| Service | PE Name | DNS Zone |
|---------|---------|----------|
| Blob | `pep-saprivatefunctions2-blob` | `privatelink.blob.core.windows.net` |
| Queue | `pep-saprivatefunctions2-queue` | `privatelink.queue.core.windows.net` |

Azure automatically creates NICs for each:  
`pep-fa-private-test01.nic`, `pep-saprivatefunctions2-blob-nic`, etc.

---

# **6. Disable Public Network Access**

After all private endpoints are created:

### Storage Account → Networking
✔ Disable Public Access  
✔ Only allow via Private Endpoints

### Function App → Networking → Access Restrictions
Set:
```
Public network access: Disabled
```

This completely prevents public access.

---

# **7. Create NSG for APIM Subnet**

Create:

```
nsg-apim
```

Assign it to:

```
subnet-apim
```

Add the following inbound rules:

| Priority | Source | Port | Protocol | Description |
|----------|--------|------|----------|-------------|
| 100 | Service Tag: AzureCloud | 3443 | TCP | Allow APIM mgmt traffic |
| 200 | Any | 80 | TCP | Allow APIM health |
| 300 | Any | 443 | TCP | Allow SSL traffic |
| 400 | Any | 8080 | TCP | Allow APIM internal |

These ensure APIM can initialize properly.

---

# **8. Deploy APIM with VNET Integration**

### APIM Instance
- **Name:** `apim-my-apps-0001`
- **Tier:** Developer (for testing)
- **Network mode:** **External**
  - This exposes a public APIM gateway URL
  - APIM’s backend traffic uses VNET

### Configure VNET:
- **Connectivity Type:** Virtual Network
- **Type:** External
- **VNET:** `vnet-apps-private`
- **Subnet:** `subnet-apim`

Azure will require:
✔ A Public IP  
✔ NSG rules  
✔ DNS resolution

---

# **9. Connect APIM to Function App API**

Once APIM is deployed:

1. Go to **APIM → APIs**
2. Choose **Add API → Function App**
3. Select:
   - `fa-private-test01`
4. Import operations from your deployed functions
5. APIM now routes all calls **privately** via internal VNET → Private Endpoint

---

# **10. Test the API through APIM**

APIM exposes a public endpoint like:

```
https://apim-my-apps-0001.azure-api.net/alive
```

APIM requires a subscription key:

### **Via Header**
```
Ocp-Apim-Subscription-Key: <your-key>
```

### **Via Query String**
```
https://apim-my-apps-0001.azure-api.net/alive?subscription-key=<key>
```

You should receive your function’s response.

---

# **11. Final Security Configuration**

### ✔ Function App
- Public access **disabled**
- Only reachable through private endpoint

### ✔ Storage Account
- Public access **disabled**
- Blob + Queue private endpoints

### ✔ APIM
- Public interface allowed (External mode)
- Backend traffic is private

### ✔ VNET
- Traffic routed internally  
- DNS zones resolve `*.azurewebsites.net` privately

---

# **12. Validate Everything**

### Function App → Networking
- Public network access: **Disabled**
- Private endpoints: **1**
- Inbound addresses: `10.0.1.x`

### Storage Account → Networking
- Public access: **Disabled**
- Private Endpoints: **2**

### APIM → Network
- Connection status: Connected
- VNET: vnet-apps-private
- Subnet: subnet-apim

---

# **🎉 Deployment Complete**

You now have:

✔ A fully private Function App  
✔ Completely locked-down Storage Account  
✔ Private DNS zones  
✔ APIM exposed publicly but connected privately inside a VNET  
✔ Secure communication through private endpoints  
✔ No public exposure of your function app  

