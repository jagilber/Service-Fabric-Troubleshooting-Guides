# V6 (NVMe) VM SKU node bootstrap / data path initialization failures

## Summary

Newer Azure VM SKUs (the `_v6` families and other NVMe-capable sizes) present their
local temporary disk as a **raw, unformatted NVMe device** instead of the older
SCSI-based temporary disk. Service Fabric, by default, uses the local temporary
disk for the node data path (`FabricDataRoot` / `dataPath`). When the NVMe disk is
not initialized and formatted, the node bootstrap agent cannot create the data
path, the node fails to come up, and the cluster can become **stuck in `Updating` /
`Upgrading`** after the deployment or SKU change.

This most commonly appears immediately after a node type's VM SKU is changed or
scaled to a `_v6` size (for example `Standard_D*ads_v6`), or after adding a new
node type that uses an NVMe-capable SKU.

## Symptoms

- Cluster resource is stuck in **`Updating`** (classic / SFRP clusters) or node
  type provisioning stalls and never completes.
- **Newly provisioned VMSS instances / new node types are not added to the
  cluster** — the deployment may report success, but the nodes never join.
- New or reimaged nodes never join the cluster / do not appear in Service Fabric
  Explorer (SFX).
- The Service Fabric VM extension (node bootstrap agent) reports errors about a
  **missing data drive / data path** or an unavailable drive letter.
- The problem starts right after a node type SKU was changed to a `_v6` size, or a
  new node type using an NVMe-capable SKU was added.

## Cause

`_v6` and other NVMe-capable VM sizes deliver **raw, unformatted local NVMe temp
disks**. Unlike previous D/E series VMs (which exposed a pre-formatted SCSI
temporary disk), the guest OS sees the NVMe local disk with `PartitionStyle = RAW`
until it is explicitly initialized, partitioned, and formatted after the VM starts.

Because Service Fabric places the node data path on the local disk by default, a
raw/uninitialized NVMe disk means the data path cannot be created and node
bootstrap stalls. The disk-initialization logic that worked for SCSI temporary
disks does not automatically handle the NVMe device, so required partitions or the
file system are never created.

> Although NVMe local disks are most commonly associated with `_v6` SKUs, some
> earlier or non-`v6` series also use NVMe (for example `Fxmsv2-series` and
> `Lsv2`+). Any VM SKU with local NVMe disks is subject to the same
> initialization considerations.

## Applies to

- Classic Service Fabric clusters (SFRP) using a `_v6` / NVMe-capable node type SKU.
- Service Fabric managed clusters (SFMC) — **managed data disk support does not
  initialize NVMe disks automatically**; NVMe disks must be initialized and
  formatted yourself (including when the temporary disk is used for `dataPath`).
- Any node type whose SKU exposes local storage over the NVMe interface.

## Detection

1. **Confirm the node type SKU.** Check whether the affected node type uses a
   `_v6` size (or any NVMe-capable size). In the Azure portal, review the
   underlying Virtual Machine Scale Set (VMSS) SKU, or:

   ```powershell
   # Classic: inspect the VMSS backing the node type
   Get-AzVmss -ResourceGroupName <rg> -VMScaleSetName <nodeTypeName> |
       Select-Object -ExpandProperty Sku
   ```

2. **Check for raw NVMe disks on the VM.** Use the VMSS **Run Command**, serial
   console, or RDP to a node instance and run:

   ```powershell
   # Any disk still RAW has not been initialized/formatted
   Get-Disk | Select-Object Number, FriendlyName, PartitionStyle, BusType, HealthStatus

   # List NVMe physical disks specifically
   Get-PhysicalDisk | Where-Object BusType -eq 'NVMe' |
       Select-Object DeviceId, FriendlyName, MediaType, Size, BusType
   ```

   A local NVMe disk reporting `PartitionStyle = RAW` (and no data drive letter
   for the Service Fabric data path) confirms this issue.

   Real output captured from a fresh `Standard_D4ads_v6` VM (Windows Server 2022
   Azure Edition) shows the local temp NVMe disk as `RAW`:

   ```text
   Number FriendlyName                  PartitionStyle BusType HealthStatus
   ------ ------------                  -------------- ------- ------------
        1 Microsoft NVMe Direct Disk v2 RAW            NVMe    Healthy
        0 Virtual_Disk NVME Premium     GPT            NVMe    Healthy
   ```

   Disk 1 (`Microsoft NVMe Direct Disk v2`) is the local temp disk and is `RAW` /
   uninitialized — so the Service Fabric data path (for example `D:\SvcFab`)
   cannot be created.

3. **Review the VM extension / bootstrap logs** on the node for messages about a
   missing data drive, missing drive letter, or a data path that could not be
   created.

## Sample errors and log entries

The exact wording varies by runtime/extension version; use these as representative
signatures rather than exact-match strings.

**Cluster / deployment level**

- The Service Fabric cluster resource remains in `Updating` and the node type /
  VMSS extension provisioning reports a failed or timed-out state.
- The Service Fabric VM extension (node bootstrap agent) reports a **failed**
  handler status referencing disk / data-drive initialization.

**Service Fabric VM extension — node bootstrap / managed-disk initialization**

Found in the node's VM extension / `ServiceFabricNodeBootstrapAgent` trace files
(commonly under `C:\WindowsAzure\Logs\Plugins\...\` — exact path and version
folder vary by extension build). LUN, drive letter, and label values shown below
are examples (the label is caller-supplied); the message text itself is emitted
by the extension's disk-initialization script:

```text
Mounting temporary disk with LUN <lun>, drive letter <driveLetter> and label <label>
Temporary disk NVMeDirect not found with LUN <lun> (This expected in vms < v6 (non NVMe) or skus without temp disk)
Data disk not found with LUN <lun>
Partition not found for data disk LUN <lun>
Exception in Powershell Script
ManagedDiskInitializationFailed
```

The extension fails because the data path (for example `D:\SvcFab`) lives on a
temp disk that was never initialized/formatted, so the drive letter does not
exist. The node bootstrap agent surfaces a `DirectoryNotFoundException` when it
tries to create the data root directory:

```text
DataRoot directory used in this node is different from specified on the settings.
dataRootDirectory D:\SvcFab  PublicSettings.DataPath: D:\SvcFab

ERROR:
Microsoft.Azure.ServiceFabric.Extension.Core.AgentException: Service Fabric
data drive for "dataPath":D:\SvcFab may not be available. Check
whether the drive is mounted correctly. Error:
System.IO.DirectoryNotFoundException: Could not find a part of the path 'D:'.
   at System.IO.__Error.WinIOError(Int32 errorCode, String maybeFullPath)
   at System.IO.Directory.InternalCreateDirectory(String fullPath, String path, Object dirSecurityObj, Boolean checkHost)
   at System.IO.Directory.InternalCreateDirectoryHelper(String path, Boolean checkHost)
   at Microsoft.Azure.ServiceFabric.Extension.Core.NodeBootstrapAgent.VerifyDataRootDriveAvailable()
   at Microsoft.Azure.ServiceFabric.Extension.Core.NodeBootstrapAgent.MoveNext()
--- End of stack trace from previous location where exception was thrown ---
   at System.Runtime.ExceptionServices.ExceptionDispatchInfo.Throw()
```

On other affected nodes of the same node type, the extension may instead report a
retry/wait while the first (failing) instance holds the bootstrap mutex:

```text
Mutex already exists; waiting up to 00:03:00 for another instance of this process to exit
```

**Guest agent generic logs (Kusto / platform)**

The Windows guest agent logs the Service Fabric extension failure with a message
indicating a missing data-drive path, for example when querying
`GuestAgentGenericLogs` filtered to the affected cluster's subscription /
resource group and `EventName contains 'ServiceFabric'`.

**Fabric / data-path driver (post-bootstrap)**

If the node partially bootstraps but the data path cannot be opened on the raw
NVMe device, the shared-log / data-path driver can fail to open the disk device
(the NVMe device interface path is not resolved correctly), which prevents
`Fabric.exe` from opening the node data root. This manifests as the node failing
to open and returning to the "not up" state after each attempt.

## Resolution

Pick the option that matches your cluster type and urgency.

### Option A — Fast mitigation: revert the node type SKU to a `_v5` (non-NVMe) size

The quickest way to restore a stuck cluster is to move the node type back to a
comparable **`_v5`** (or other SCSI temp-disk) SKU, which exposes a pre-formatted
temporary disk that Service Fabric can use without extra initialization.

- Classic (SFRP): update the node type SKU in the ARM template / deployment from
  the `_v6` size back to the equivalent `_v5` size and redeploy.
- Ensure the target SKU still has a local temporary disk of **32 GB or more** — SF
  requires local temporary storage; SKUs with **no temp storage are not supported**.

### Option B — Initialize the NVMe disk with a Custom Script Extension (CSE)

To keep using a `_v6` / NVMe SKU, add a **Custom Script Extension** to the VMSS
that initializes and formats the raw NVMe disk on every VM start/reimage, **before**
Service Fabric needs the data path. This applies to both classic and managed
clusters.

Because NVMe local (temp) disks are transient, the disk must be re-initialized
after user-initiated stops, deallocations, planned maintenance, and reimage
events. The CSE must therefore be **idempotent** — it should only initialize a
disk that is still RAW and skip disks that are already formatted.

> **Gotcha (confirmed by repro):** on `_v6` VMs the DVD/CD-ROM drive frequently
> already owns `D:`. Creating the data partition succeeds, but assigning `D:` to
> it fails with `The requested access path is already in use.` The script must
> first move the DVD drive to another letter — this is the same reason the
> Service Fabric VM extension contains "change DVD drive letter" logic.

The following script was **validated on a `Standard_D4ads_v6` VM (Windows Server
2022 Azure Edition)** — it moves the DVD off `D:`, initializes the raw NVMe temp
disk, formats it NTFS, and assigns `D:` (adapt the drive letter/label to your
data path):

```powershell
# initialize-sf-nvme-datadisk.ps1
# Idempotently initialize/format the local NVMe temp disk for the SF data path.
# Validated on Standard_D4ads_v6 (Windows Server 2022 Azure Edition).
$driveLetter = 'D'          # drive letter Service Fabric expects for the data path
$label       = 'SF-DataDisk'

# 1. On v6 VMs the DVD/CD-ROM drive often holds D:. Move it out of the way first,
#    otherwise assigning D: to the data partition fails with
#    "The requested access path is already in use."
$cd = Get-CimInstance -ClassName Win32_Volume -Filter "DriveType=5 AND DriveLetter='$driveLetter`:'"
if ($cd) {
    Write-Output "DVD/CD-ROM is using $driveLetter`: - reassigning it to Y:"
    Set-CimInstance -InputObject $cd -Property @{ DriveLetter = 'Y:' } | Out-Null
}

# 2. Target the local NVMe temp disk (non-system 'NVMe Direct' disk). Init if RAW.
$disk = Get-Disk |
    Where-Object { $_.BusType -eq 'NVMe' -and -not $_.IsSystem -and $_.FriendlyName -like '*NVMe Direct Disk*' } |
    Sort-Object Number | Select-Object -First 1
if (-not $disk) {
    Write-Warning 'No local NVMe temp disk found - verify the SKU has a local NVMe temp disk.'
    return
}
if ($disk.PartitionStyle -eq 'RAW') {
    Write-Output "Initializing raw NVMe disk $($disk.Number) ($($disk.FriendlyName))"
    Initialize-Disk -Number $disk.Number -PartitionStyle GPT
}

# 3. Create + format a data partition if one does not already exist (idempotent).
$data = Get-Partition -DiskNumber $disk.Number -ErrorAction SilentlyContinue |
    Where-Object { $_.Type -eq 'Basic' }
if (-not $data) {
    $data = New-Partition -DiskNumber $disk.Number -UseMaximumSize
    Format-Volume -Partition $data -FileSystem NTFS -NewFileSystemLabel $label -Confirm:$false -Force | Out-Null
}

# 4. Ensure the data partition owns the SF drive letter.
if ($data.DriveLetter -ne $driveLetter) {
    $data | Set-Partition -NewDriveLetter $driveLetter
}

Get-Volume -DriveLetter $driveLetter |
    Format-Table DriveLetter, FileSystemLabel, FileSystem, HealthStatus
```

After running, the SF data drive is a formatted NTFS volume on the NVMe temp disk
(captured from the repro VM):

```text
DriveLetter FileSystemLabel FileSystem HealthStatus  GB
----------- --------------- ---------- ------------  --
          D SF-DataDisk     NTFS       Healthy      220
```

Notes for Option B:
- Sequence the CSE so it runs **before** the Service Fabric extension on the VMSS,
  so the data path exists when node bootstrap runs.
- A `_v6` SKU can expose **more than one** local NVMe temp disk; select the intended
  disk deterministically (for example, the first NVMe local disk) rather than
  assuming a single device.
- The guest OS image must include NVMe driver support (most recent Windows Server
  images do).

### Option C — Use a managed data disk for the data path

Configure the node type to use a **managed data disk** as the data path instead of
the temporary disk. Even in this configuration on a `_v6` / NVMe SKU, the disk must
still be initialized/formatted via a Custom Script Extension (see Option B) — the
platform does not initialize NVMe disks automatically.

## Notes

- Service Fabric (and Reliable Collections) are designed to run on **local disks**.
  Keep the data path on local/managed disk and avoid pointing it at remote/XStore
  backed storage. See [Changing the default DataPath](../Cluster/Changing%20DataPath.md).
- SKUs with **no local temporary storage** are not supported for the SF data path.
  If you must use such a SKU, use a managed data disk (Option C).
- After mitigating with Option A (`_v5`), you can later move to a `_v6` SKU once the
  CSE-based NVMe initialization (Option B) is in place and validated.

## References

- [Deploy a Service Fabric cluster node type with managed data disks (NVMe note)](https://learn.microsoft.com/azure/service-fabric/service-fabric-managed-disk)
- [FAQ for Temp NVMe disks](https://learn.microsoft.com/azure/virtual-machines/enable-nvme-temp-faqs)
- [Enable NVMe interface on your VMs and VM scale sets](https://learn.microsoft.com/azure/virtual-machines/enable-nvme-interface)
- [NVMe overview](https://learn.microsoft.com/azure/virtual-machines/nvme-overview)
- [Dadsv6 sizes series](https://learn.microsoft.com/azure/virtual-machines/sizes/general-purpose/dadsv6-series)
- [Deploy a managed cluster with stateless node types — temporary disk support](https://learn.microsoft.com/azure/service-fabric/how-to-managed-cluster-stateless-node-type#temporary-disk-support)
- [Changing the default DataPath](../Cluster/Changing%20DataPath.md)
