# Copy an Azure Compute Gallery Image Across Tenants

A practical runbook for copying a VM image version from an Azure Compute Gallery in one Microsoft Entra tenant to a gallery in another tenant. It's tested with a Windows Server 2022, Gen2, Trusted Launch, Specialized image.

## Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Method 1: Gallery-to-gallery copy (recommended)](#method-1-gallery-to-gallery-copy-recommended)
- [Method 2: Export the VHD and re-import (fallback)](#method-2-export-the-vhd-and-re-import-fallback)
- [After the copy](#after-the-copy)
- [Troubleshooting](#troubleshooting)
- [Cleanup and security](#cleanup-and-security)
- [Lessons learned](#lessons-learned)
- [References](#references)

---

## Overview

Use **Method 1** whenever one account can sign in to both tenants. It copies the image inside Azure, with no temp disks, SAS URLs or downloads. Use **Method 2** only when no single identity has access to both tenants.

| | Method 1: Gallery-to-gallery | Method 2: VHD export and import |
| --- | --- | --- |
| Needs | Same identity in both tenants (user or multi-tenant app) | Any access to each tenant separately |
| Data path | Azure-internal | Disk export SAS, then AzCopy or Storage Explorer |
| Typical time | 30–90 min for ~127 GB to 5 regions | 1–3 h including copy and upload |
| Temp resources | None | Temp disk, storage account or helper VM |
| Cleanup | None | Cancel export, delete temp resources |

---

## Prerequisites

Collect these values before starting; every later command uses them.

| Item | Where to find it |
| --- | --- |
| Source tenant ID | Source portal > Microsoft Entra ID > Overview |
| Target tenant ID | Target portal > Microsoft Entra ID > Overview |
| Source image version resource ID | Source gallery > image definition > Versions > version > JSON View |
| Source replica regions | Source image version > Replication |
| Source definition settings: OS type, OS state, VM generation, security type, publisher/offer/SKU | Source image definition > Overview |
| Target subscription ID, resource group, gallery name | Target portal > Azure Compute Galleries > gallery > Overview |

**Permissions needed:**

- **Reader** on the source gallery (or image version)
- **Contributor** on the target gallery's resource group

> **Tip:** Run all commands from **Cloud Shell** in the portal. In PowerShell Cloud Shell, run each command as one line; `\` line continuation only works in Bash.

To read the source definition's settings in one go:

```bash
az sig image-definition show --ids "<SOURCE_IMAGE_DEFINITION_ID>" \
  --query "{os:osType, state:osState, gen:hyperVGeneration, features:features, publisher:identifier.publisher, offer:identifier.offer, sku:identifier.sku}" -o json
```

---

## Method 1: Gallery-to-gallery copy (recommended)

Azure copies the image version directly from the source gallery into the target gallery, using your sign-in to both tenants.

### Step 1: Create a matching image definition in the target gallery

The target definition must match the source **exactly**, or the copy fails. Security type is the most common mistake: a source with **Trusted launch** needs a target with **Trusted launch**, not **Trusted launch supported**.

```bash
az sig image-definition create \
  -g <target-rg> -r <target-gallery> -i <definition-name> -l <region> \
  --os-type Windows \
  --os-state <Generalized|Specialized> \
  --hyper-v-generation V2 \
  --features SecurityType=TrustedLaunch \
  --publisher <publisher> --offer <offer> --sku <sku>
```

- Omit `--features` if the source security type is Standard.
- The security type **cannot be changed** after creation; delete and recreate the definition instead.

### Step 2: Sign in to both tenants

```bash
az login --tenant <SOURCE_TENANT_ID> --use-device-code

# Check that you can read the source image version (should show Succeeded)
az sig image-version show --ids "<SOURCE_IMAGE_VERSION_ID>" \
  --query "{name:name, state:provisioningState}" -o table

az login --tenant <TARGET_TENANT_ID> --use-device-code
az account set --subscription <TARGET_SUB_ID>
az account show --query "{sub:name, tenant:tenantId}" -o table
```

### Step 3: Start the copy

```bash
az sig image-version create \
  --resource-group <target-rg> \
  --gallery-name <target-gallery> \
  --gallery-image-definition <definition-name> \
  --gallery-image-version <version> \
  --location <region> \
  --target-regions <region1> <region2> <region3> \
  --replica-count 1 \
  --storage-account-type Standard_LRS \
  --image-version "<SOURCE_IMAGE_VERSION_ID>" \
  --no-wait
```

- `--location` must be a region where the source version has a replica.
- `--no-wait` lets the copy continue in Azure if Cloud Shell closes.

### Step 4: Monitor until Succeeded

```bash
az sig image-version show \
  -g <target-rg> -r <target-gallery> -i <definition-name> -e <version> \
  --expand ReplicationStatus \
  --query "{state:provisioningState, regions:replicationStatus.summary[].{region:region,state:state,progress:progress}}" -o json
```

The copy is done when every region shows `Completed` and `state` is `Succeeded`. The overall state can trail the regions by 5–15 minutes.

---

## Method 2: Export the VHD and re-import (fallback)

The image becomes a managed disk in the source tenant, is copied as a VHD, and becomes a managed disk again in the target. Both ends use SAS URLs, so no sign-in inside a helper VM is needed.

### Step 1: Source tenant: create a disk from the image

The portal's disk form can't use a gallery image as its source, so use Cloud Shell:

```bash
az disk create -g <temp-rg> -n <temp-disk> -l <region> \
  --sku StandardSSD_LRS \
  --gallery-image-reference "<SOURCE_IMAGE_VERSION_ID>"
# If it errors on security type, add:
#   --security-type TrustedLaunch --hyper-v-generation V2
```

### Step 2: Source tenant: generate the export URL

Open **Disks** > the temp disk > **Disk Export**, set **14400** seconds, and select **Generate URL**. Copy only the **first URL** (the VHD). The VM Guest State URL isn't needed.

### Step 3: Target tenant: create an empty disk ready for upload

The upload size is the VHD's exact size in bytes: the disk size in bytes plus 512.

```bash
az disk create -g <target-rg> -n <upload-disk> -l <region> \
  --for-upload --upload-size-bytes <SIZE> \
  --sku StandardSSD_LRS --os-type Windows --hyper-v-generation V2

az disk grant-access -g <target-rg> -n <upload-disk> \
  --access-level Write --duration-in-seconds 86400 \
  --query accessSas -o tsv
```

### Step 4: Copy disk to disk with AzCopy

Run this from Cloud Shell, or from a helper VM in the same region if your network is unreliable:

```bash
azcopy copy '<EXPORT-URL>' '<WRITE-SAS-URL>' \
  --blob-type PageBlob --s2s-preserve-access-tier=false
```

When copying into a storage account instead, the destination format is:

```
https://<account>.blob.core.windows.net/<container>/<file>.vhd?<SAS token>
```

Put only the **token** after the `?`, not the full SAS URL.

### Step 5: Target tenant: finalize the disk

```bash
az disk revoke-access -g <target-rg> -n <upload-disk>
```

This is required; until you run it, the disk stays locked in upload mode.

### Step 6: Target tenant: create the image

1. Create the image definition exactly as in [Method 1, Step 1](#step-1-create-a-matching-image-definition-in-the-target-gallery).
2. Open the definition > **Add version** > Source: **Disks or snapshots** > OS disk: the uploaded disk.
3. On **Replication**, add the target regions, then **Review + create**.

---

## After the copy

1. Update `imageReference` in the ARM template:

   ```json
   "imageReference": {
     "id": "/subscriptions/<TARGET_SUB_ID>/resourceGroups/<target-rg>/providers/Microsoft.Compute/galleries/<target-gallery>/images/<definition-name>/versions/<version>"
   }
   ```

2. If labs deploy into other subscriptions in the target tenant, grant the deploying identity **Reader** on the target gallery.
3. Keep the VM settings consistent with the image:
   - A **Trusted launch** image needs `securityProfile.securityType: TrustedLaunch` on the VM.
   - A **Specialized** image needs **no** `osProfile` in the VM resource; a **Generalized** image needs one.
4. Deploy one test environment in each region you rely on. Confirm the VM boots, RDP works, and the extensions and startup scripts complete.

---

## Troubleshooting

When a version shows `Failed`, read the reason from the Activity Log, then delete the failed version before retrying:

```bash
az monitor activity-log list --resource-group <target-rg> --offset 3h \
  --query "[?status.value=='Failed'].{time:eventTimestamp, op:operationName.localizedValue, msg:properties.statusMessage}" -o json

az sig image-version delete -g <target-rg> -r <target-gallery> -i <definition-name> -e <version>
```

| Error or symptom | Cause | Fix |
| --- | --- | --- |
| *Source contains TrustedLaunch security data that cannot be used with TrustedLaunchSupported* | Target definition has the wrong security type | Delete and recreate the definition with `--features SecurityType=TrustedLaunch` |
| OSState, HyperVGeneration or Features mismatch | Target definition differs from the source | Compare both with `az sig image-definition show` and recreate the target |
| `LinkedAuthorizationFailed` / auxiliary token | Source-tenant sign-in missing or expired | Repeat both `az login` commands, then retry |
| `AuthorizationFailed` on the source scope | No read access to the source | Request Reader on the source gallery |
| `AADSTS50020` | The account doesn't exist in that tenant | Use Method 2, or a multi-tenant app registration |
| `AADSTS53003` | Blocked by Conditional Access | Sign in from an allowed location (for example, your laptop's Cloud Shell) |
| AzCopy: `sp is mandatory` | The SAS token was pasted with an extra `?` or as the full URL | Put only the token after `?` in the destination |
| AzCopy fails instantly, with no error on screen | Expired or cancelled export URL, rotated key, or network block | Check `~/.azcopy/<job>.log` for the HTTP status |
| `Unexpected token '\'` in Cloud Shell | Bash line continuation used in PowerShell | Run the command on one line, or switch to Bash |
| Region status Completed but state still Creating | Normal finalization delay | Wait 5–15 minutes |

---

## Cleanup and security

Method 1 leaves nothing to clean up. After Method 2, once a test deployment succeeds:

- [ ] Source: on the temp disk, open **Disk Export** and select **Cancel export**, then delete the temp resource group
- [ ] Target: delete the uploaded disk, any staging storage account, and any helper VM's resource group
- [ ] Delete any local `.vhd` copies

**Security notes:**

- Treat export URLs and SAS tokens as secrets. Anyone holding one can read the full disk, or write to the target, until it expires.
- Never paste SAS URLs into chats, tickets, email or commits. If one leaks, cancel the export (source) and rotate the storage key or revoke disk access (target).
- Use the shortest expiry that covers the copy, and restrict any helper VM's RDP rule to your own IP.

---

## Lessons learned

- **Try Method 1 first.** If the same account works in both tenants, it's the fastest and cleanest route by far.
- **Match the definition exactly.** "Trusted launch" and "Trusted launch supported" are different, and the security type can't be edited later.
- **Check the OS state.** A VM resource with no `osProfile` usually means the image is Specialized.
- **The portal can't create a disk from a gallery image.** Use `az disk create --gallery-image-reference`.
- **Cloud Shell defaults to PowerShell.** Keep commands on one line, or switch to Bash.
- **Region "Completed" isn't the end.** Wait for the version's `provisioningState` to show `Succeeded`.

---

## References

- [Create an image definition and image version](https://learn.microsoft.com/azure/virtual-machines/image-version)
- [Share gallery images across tenants using an app registration](https://learn.microsoft.com/azure/virtual-machines/share-using-app-registration)
- [Export an image version to a managed disk](https://learn.microsoft.com/azure/virtual-machines/managed-disk-from-image-version)
- [az sig image-version CLI reference](https://learn.microsoft.com/cli/azure/sig/image-version)
