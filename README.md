# M365-CLI

CLI scripts for Microsoft 365 (Entra ID + Microsoft Graph) that run **entirely from Azure Cloud Shell** using **`az rest`**. Automate user creation/deletion, license assignment/removal (including plan‑level disables), and export the tenant/user license inventory you need to fill bulk CSVs. 

## Prerequisites

- An active Microsoft 365 tenant and permissions to manage users and licenses (e.g., User Administrator, License Administrator). The APIs used here require delegated permissions such as `User.ReadWrite.All` and `LicenseAssignment.ReadWrite.All` when acting via `az rest`.   
- Azure Cloud Shell (Bash) with Azure CLI signed in: `az login`. All REST calls go through `az rest` (no third‑party CLIs). 

> Tip: This repo uses **Microsoft Graph v1.0** endpoints for users, licenses, and SKUs. Keep usage consistent with `https://graph.microsoft.com/v1.0`. 

---

## What These Scripts Do

- **Discovery (one‑CSV export)**  
  Exports tenant SKUs, SKU service plans, all users and their `usageLocation`, and each user’s assigned SKUs & plan statuses into **one CSV** (`discovery_all.csv`). This gives admins authoritative values for `AddSkus`, `RemoveSkus`, `DisablePlans`, and `UsageLocation` before running bulk changes. Uses:  
  - `GET /subscribedSkus` for inventory and service plans.   
  - `GET /users?$select=userPrincipalName,usageLocation` (paged).   
  - `GET /users/{id}/licenseDetails` (per user). 

- **Bulk Users & Licensing (CSV‑driven)**  
  Processes actions per row: **create**, **delete**, **assign**, **remove**. Accepts SKU **part numbers** or **GUIDs**, supports **plan‑level disables** when assigning a license, and enforces `usageLocation` before licensing (Graph requirement). Uses:  
  - `POST /users` to create users.   
  - `PATCH /users` to set `usageLocation`.   
  - `POST /users/{id|UPN}/assignLicense` to add/remove SKUs + `disabledPlans`. 

---

## Repo Structure

```
M365-CLI/
├─ discovery_one_csv.sh      # Read-only export to discovery_all.csv (one file)
├─ m365_csv.sh               # CSV-driven create/delete/assign/remove + plan disables
├─ m365_users.csv            # Sample operational CSV (actions to perform)
└─ README.md                 # This file
```

---

## Quick Start

### 1) Export authoritative values (single CSV)

```bash
bash discovery_one_csv.sh
# -> discovery_all.csv with sections:
#    TENANT_SKU, SKU_PLAN, USER, USER_LICENSE_PLAN
```

- **Why:** You’ll use `skuPartNumber` / `skuId` and `servicePlanName` / `servicePlanId` from this export to populate your operational CSV. `usageLocation` is included because Graph requires it for license assignment. 

### 2) Create a working CSV

Start from a template:

```bash
bash m365_csv.sh --init-csv --csv m365_users.csv
```

This template includes:  
`Action,FirstName,LastName,Alias,UserPrincipalName,Password,UsageLocation,AddSkus,RemoveSkus,DisablePlans,SkipLicense`

- `AddSkus` / `RemoveSkus`: comma‑separated `skuPartNumber` or `skuId` (see `discovery_all.csv` rows `TENANT_SKU`).   
- `DisablePlans`: either `PLAN_A,PLAN_B` for a single SKU or `SKU:PLAN_A,PLAN_B;SKU2:PLAN_X` for multiple SKUs (plan names or GUIDs; see `SKU_PLAN` rows).   
- `UsageLocation`: 2‑letter ISO (required before licensing). 

### 3) Apply changes

```bash
# Process the entire CSV (create/delete/assign/remove by row)
bash m365_csv.sh --csv m365_users.csv process

# Or a single action subset:
bash m365_csv.sh --csv m365_users.csv assign
```

> Under the hood, the script calls Graph v1.0: `POST /users`, `PATCH /users`, and `POST /users/{id}/assignLicense`, all via `az rest`. 

---

## Scripts & Usage

### discovery_one_csv.sh

**Purpose:** Export everything you need to fill the operational CSV into **one file**: `discovery_all.csv`.  
**Sections in the CSV:**

- `TENANT_SKU`: `skuPartNumber`, `skuId`, enabled/consumed counts.   
- `SKU_PLAN`: per-SKU `servicePlanName` and `servicePlanId`.   
- `USER`: `userPrincipalName`, `usageLocation`.   
- `USER_LICENSE_PLAN`: user’s assigned `skuPartNumber`/`skuId` and each plan’s provisioning status. 

**Run:**
```bash
bash discovery_one_csv.sh
```

### m365_csv.sh

**Purpose:** Create/delete users and add/remove licenses with optional **plan disables**, driven by a CSV.

**Key behaviors:**
- Creates users (`POST /users`) with `displayName`, `mailNickname`, `userPrincipalName`, and `passwordProfile`.   
- Ensures `usageLocation` is set (`PATCH /users`) before any license change (Graph requirement).   
- Adds/removes licenses and can disable selected plans in the same call (`/assignLicense`).   
- Resolves `skuPartNumber → skuId` and `servicePlanName → servicePlanId` from your tenant inventory (`/subscribedSkus`). 

**Generate template:**
```bash
bash m365_csv.sh --init-csv --csv m365_users.csv
```

**List SKUs and plans:**
```bash
bash m365_csv.sh --list-skus
bash m365_csv.sh --list-plans M365_BUSINESS_PREMIUM   # or a skuId GUID
```

**Process CSV:**
```bash
# All rows
bash m365_csv.sh --csv m365_users.csv process

# Only one action
bash m365_csv.sh --csv m365_users.csv assign
bash m365_csv.sh --csv m365_users.csv remove
bash m365_csv.sh --csv m365_users.csv create
bash m365_csv.sh --csv m365_users.csv delete

# Skip licensing globally for this run
bash m365_csv.sh --csv m365_users.csv --skip-license process
```

---

## Notes & Customization

- **Cloud Shell friendly:** Calls only `az rest`; avoids third‑party tools. Keep your session authenticated with `az login`.   
- **Graph paging:** User export follows `@odata.nextLink` until complete; replication delays for just‑created users are possible per Graph docs.   
- **Licensing rules:** Microsoft requires `usageLocation` on the user before license assignment; errors like *“invalid usage location”* are resolved by setting it first. 

---

## Troubleshooting

- **Script seems to “close” the shell:** Avoid `set -e` and unhandled `exit` on REST failures; these scripts catch errors and continue so the Cloud Shell session stays open.
- **SKU not found / plan name unknown:** Re‑run `discovery_one_csv.sh` and confirm the **exact** `skuPartNumber` and `servicePlanName` from your tenant (`/subscribedSkus`).   
- **License assignment fails:** Verify `usageLocation` (see `USER` rows in `discovery_all.csv`) and that the SKU has available enabled units. 

---

## Contributing

- Open an issue or a PR with your scenario and logs (omit real UPNs if sensitive).  
- Keep scripts Bash‑only with `az rest` to stay Cloud Shell compatible. This aligns with the lightweight approach shown in your **Azure-CLI** repo. 

---

## References

- Microsoft Graph REST API v1.0 overview (endpoint patterns and guidance).   
- List tenant SKUs / service plans: `GET /subscribedSkus`.   
- List users with `usageLocation`: `GET /users` (paging, `$select`).   
- Per‑user license details: `GET /users/{id}/licenseDetails`.   
- Create and update users: `POST /users`, `PATCH /users`.   
- Assign/remove licenses + disable service plans: `POST /users/{id}/assignLicense`. 

---
