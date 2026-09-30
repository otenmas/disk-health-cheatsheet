<p align="right"><a href="README.pt-BR.md">🇧🇷 Português</a></p>

# disk-health-cheatsheet

Quick reference for checking HDD and SSD health and integrity using terminal and PowerShell commands.

## Table of Contents

- [Before you start](#before-you-start)
- [Validation workflow (step by step)](#validation-workflow-step-by-step)
- [Windows (PowerShell / CMD)](#windows-powershell--cmd)
- [Linux (terminal)](#linux-terminal)
- [How to read SMART data](#how-to-read-smart-data)
- [Warning signs](#warning-signs)
- [Wiping, formatting and partitioning](#wiping-formatting-and-partitioning)
- [License](#license)

## Before you start

- Most commands need **administrator** (Windows) or **root/sudo** (Linux).
- **Back up your data first** if the disk already shows problems. Repair tools can make a failing disk worse.
- Commands marked as *read-only* do not change anything on the disk.
- Replace `/dev/sdX`, `/dev/nvme0` and `C:` with your own device or drive letter.
- ⚠️ Commands using `C:` act on **your own computer's disk**, not on the disk connected to the dock.
- If the disk came from a company, confirm that reusing it is authorized and that the old data has already been handled.

## Validation workflow (step by step)

A routine to validate discarded (or new) SATA disks connected through a USB dock/enclosure on Windows. Test **one disk at a time**.

### Step 1: Preparation

- Label the disk and write down its model, serial number and capacity.
- Open PowerShell **as Administrator**.
- Install smartmontools (see [Install smartmontools](#install-smartmontools)).
- Disable PC sleep and USB selective suspend so long tests are not interrupted.

### Step 2: Identification

```powershell
Get-Disk
smartctl --scan
smartctl -i /dev/sdX -d sat
```

- Check in `Get-Disk` that `BusType` is **USB**, and note the disk number.
- Confirm the **real model and serial number** in `smartctl -i` (compare with the label). The serial shown by Windows is usually a generic value when the disk is in a dock.
- Check that `SMART support is: Available` and `Enabled` appear.
- If Windows asks you to format the RAW disk, click **Cancel**.

### Step 3: Passive analysis (SMART)

```powershell
smartctl -a /dev/sdX -d sat
```

**Write down the initial values** and check:

- `overall-health`: should be **PASSED**
- `SSD_Life_Left` (remaining life, on SSDs)
- `Reallocated_Event_Count` or `Reallocated_Sector_Ct`: ideally **0**
- `Current_Pending_Sector` and `Offline_Uncorrectable`: should be **0**
- `CRC_Error_Count`: if high, suspect the cable or the dock

### Step 4: Active tests (SMART self-tests)

```powershell
smartctl -t short /dev/sdX -d sat
smartctl -l selftest /dev/sdX -d sat

smartctl -t long /dev/sdX -d sat
smartctl -l selftest /dev/sdX -d sat
```

- The short test takes about 2 minutes. The long test can take hours.
- Do not unplug or touch the dock during the test.
- The expected result is **Completed without error**.

### Step 5: Re-read SMART and compare

```powershell
smartctl -a /dev/sdX -d sat
```

Compare with the values you wrote down in Step 3. If reallocated, pending or uncorrectable counts **increased**, the disk is getting worse.

### Step 6: Surface scan (optional, HDD only)

Use Victoria in **read** mode (no remapping or writing). Many slow or erroring blocks indicate a degrading disk. On SSDs this step is unnecessary.

### Step 7: Verdict

| Situation | Decision |
|---|---|
| `PASSED`, self-tests without errors, reallocated/pending/uncorrectable at 0 and stable | **Approved** |
| A few reallocated sectors, stable, and tests without errors | **Use with caution** (unimportant data, with backups) |
| `FAILED`, any self-test error, pending/uncorrectable above 0, or growing values | **Rejected** |

### Step 8: Wiping and formatting

Only for approved disks. See [Wiping, formatting and partitioning](#wiping-formatting-and-partitioning).

### Step 9: Performance validation

Run **CrystalDiskMark** ("All" button) and compare the results with the manufacturer's specification. Through a dock, the limit is usually the dock or the USB port, not the disk.

### Step 10: Record keeping

For each disk, write down: serial number, model, power-on hours, reallocated sectors, remaining life (SSD), test results and verdict.

## Windows (PowerShell / CMD)

Open PowerShell **as Administrator**.

### Install smartmontools

Windows does not show full SMART attributes natively. Install [smartmontools](https://www.smartmontools.org/) with winget:

```powershell
winget install smartmontools.smartmontools
```

Close and reopen PowerShell, then verify the installation:

```powershell
smartctl --version
smartctl --scan
```

If the command is not recognized, call it by its full path:

```powershell
& "C:\Program Files\smartmontools\bin\smartctl.exe" --scan
```

Graphical alternative: CrystalDiskInfo.

### Identify the disk

```powershell
# List disks, partitions and volumes
Get-Disk
Get-Partition
Get-Volume
```

```powershell
# Confirm the disk is on USB (replace 2 with your disk number)
Get-Disk -Number 2 | Select-Object Number, FriendlyName, BusType, Size, PartitionStyle
```

### SMART with smartctl (USB dock/enclosure)

`/dev/sdX` follows the `Get-Disk` order: disk 0 = `sda`, disk 1 = `sdb`, disk 2 = `sdc`, and so on.

```powershell
# Disk information (real model and serial number)
smartctl -i /dev/sdc -d sat

# All SMART data
smartctl -a /dev/sdc -d sat
```

`-d sat` (*SCSI/ATA Translation*) lets the SMART command pass through the USB dock. If it fails or returns nothing, try:

```powershell
smartctl -i /dev/sdc -d sat,12
smartctl -i /dev/sdc -d usbjmicron
smartctl -i /dev/sdc -d usbsunplus
```

If none of them work, the dock does not pass SMART through. Use another dock or connect the disk directly to the motherboard.

#### How to read the result

- `SMART overall-health self-assessment test result`: **PASSED** is expected. **FAILED** means the disk is rejected.
- `SSD_Life_Left`: remaining life in %. The closer to 100, the better.
- `Reallocated_Event_Count`: ideally **0**.
- `SATA_CRC_Error_Count` (ID 199): ideally **0**. If it grows, suspect the cable or the dock.

### SMART self-tests

```powershell
# Short test (~2 min) and result
smartctl -t short /dev/sdc -d sat
smartctl -l selftest /dev/sdc -d sat

# Long test (may take hours) and result
smartctl -t long /dev/sdc -d sat
smartctl -l selftest /dev/sdc -d sat
```

### Health through Windows

```powershell
# Health and operational status of all physical disks
Get-PhysicalDisk | Select-Object DeviceId, FriendlyName, BusType, MediaType, HealthStatus, OperationalStatus, Size

# Or just one disk (replace 2 with your disk number)
Get-PhysicalDisk | Where-Object DeviceId -eq 2
```

```powershell
# Temperature, wear, error counters and power-on hours (read-only)
Get-PhysicalDisk | Get-StorageReliabilityCounter |
  Select-Object DeviceId, Temperature, Wear, ReadErrorsTotal, WriteErrorsTotal, PowerOnHours
```

> On USB disks, it is normal for `MediaType` to show **Unspecified** and for `Get-StorageReliabilityCounter` to return nothing. `smartctl` is the reliable source.

### File system check (only if the disk has a drive letter)

Does not apply to RAW disks. Replace `C:` with the letter of the disk you are testing.

```powershell
# Online scan, does not lock the volume (read-only)
Repair-Volume -DriveLetter C -Scan

# Same idea using chkdsk
chkdsk C: /scan

# Check if the volume is flagged as "dirty"
fsutil dirty query C:
```

```powershell
# Fix errors (may require reboot for the system drive)
chkdsk C: /f

# Fix errors AND scan for bad sectors (slow, HDD only)
chkdsk C: /f /r
```

> ⚠️ Avoid `chkdsk /r` on SSDs. It is slow and unnecessary; `/f` is enough.

### Windows system file integrity

These commands check the **Windows installed on your PC**, not the disk in the dock.

```powershell
sfc /scannow
DISM /Online /Cleanup-Image /CheckHealth
DISM /Online /Cleanup-Image /ScanHealth
DISM /Online /Cleanup-Image /RestoreHealth
```

### Disk errors in the event log

```powershell
Get-WinEvent -FilterHashtable @{LogName='System'; ProviderName='disk'} -MaxEvents 50
```

### Quick performance test

```powershell
# Replace "c" with the disk's drive letter (only works on a disk with a letter)
winsat disk -drive c
```

## Linux (terminal)

### Install the tools

```bash
# Debian / Ubuntu
sudo apt install smartmontools nvme-cli

# Fedora
sudo dnf install smartmontools nvme-cli
```

### Identify disks

```bash
lsblk -o NAME,SIZE,TYPE,MODEL,MOUNTPOINT
sudo smartctl --scan
```

### SMART with smartctl

```bash
# Device information
sudo smartctl -i /dev/sdX

# Overall health verdict (PASSED / FAILED)
sudo smartctl -H /dev/sdX

# Attributes table
sudo smartctl -A /dev/sdX

# Everything
sudo smartctl -a /dev/sdX
```

### SMART self-tests

```bash
# Short test (~2 minutes)
sudo smartctl -t short /dev/sdX

# Long test (from minutes to hours, depending on disk size)
sudo smartctl -t long /dev/sdX

# View results after the test finishes
sudo smartctl -l selftest /dev/sdX
```

### NVMe drives

```bash
sudo smartctl -a /dev/nvme0
sudo nvme smart-log /dev/nvme0
```

### File system check

```bash
# The partition must be UNMOUNTED
sudo umount /dev/sdX1

# Check only, no changes (read-only)
sudo fsck -n /dev/sdX1

# Check and repair
sudo fsck -f /dev/sdX1
```

### Bad sectors

```bash
# Read-only test (safe, default mode)
sudo badblocks -sv /dev/sdX
```

> ⚠️ Never use `badblocks -w` (destructive write test) on a disk with data. It is also not recommended for SSDs.

### Kernel messages and I/O errors

```bash
sudo dmesg | grep -i -E "error|i/o|ata|nvme|sector"
journalctl -k -p err
```

### Performance and usage

```bash
# Quick read speed test
sudo hdparm -Tt /dev/sdX

# Live I/O statistics (package: sysstat)
iostat -x 2
```

## How to read SMART data

### HDD and SATA SSD attributes

| ID  | Attribute               | What it means                                                                                     |
|-----|-------------------------|---------------------------------------------------------------------------------------------------|
| 5   | Reallocated_Sector_Ct   | Bad sectors already remapped. Rising values are a bad sign.                                       |
| 9   | Power_On_Hours          | Total hours the disk has been powered on.                                                         |
| 12  | Power_Cycle_Count       | How many times the disk has been powered on and off.                                              |
| 177 | Wear_Leveling_Count     | SSD wear (meaning varies by manufacturer).                                                        |
| 187 | Reported_Uncorrect      | Errors that could not be corrected.                                                               |
| 194 | Temperature_Celsius     | Current temperature.                                                                              |
| 196 | Reallocated_Event_Count | **Very important.** How many times the disk moved data from a failing area to a spare one. Zero is ideal. |
| 197 | Current_Pending_Sector  | Sectors waiting to be remapped. Should be 0.                                                      |
| 198 | Offline_Uncorrectable   | Sectors that failed during offline scan. Should be 0.                                             |
| 199 | UDMA_CRC_Error_Count    | Usually a bad **cable or connection**, not the disk itself (some disks call it `SATA_CRC_Error_Count`). |
| 231 | SSD_Life_Left           | Remaining SSD life, in %.                                                                         |
| 233 | Media_Wearout_Indicator | Remaining SSD life (Intel and some others).                                                       |
| 234 | Flash_Writes_GiB        | Total GiB written to the disk.                                                                    |

> Attribute IDs and names vary between manufacturers. Trust the **name** and check it in your own disk's output.

### NVMe fields

| Field                           | What it means                                                       |
|---------------------------------|---------------------------------------------------------------------|
| Critical Warning                | Should be `0`.                                                      |
| Percentage Used                 | SSD life used (100% = rated endurance reached).                     |
| Available Spare                 | Spare blocks left. Should be well above the threshold.              |
| Media and Data Integrity Errors | Should be `0`.                                                      |

## Warning signs

- SMART overall health says **FAILED**
- `Reallocated_Sector_Ct`, `Current_Pending_Sector` or `Offline_Uncorrectable` above 0 and growing
- Repeated I/O errors in `dmesg` or the Windows event log
- Very slow reads, freezes, or unusual noises such as clicking (HDD)
- SSD `Percentage Used` close to 100% or `Available Spare` near the threshold

If you see any of these, **back up your data immediately** and plan a replacement.

## Wiping, formatting and partitioning

> ⚠️ **These commands erase everything and cannot be undone.** Only use them on disks that passed the tests, and double-check the disk number. Getting the number wrong can wipe your own computer's disk.

```powershell
# 1. Confirm the disk number: BusType should be USB and IsBoot/IsSystem should be False
Get-Disk | Select-Object Number, FriendlyName, BusType, Size, PartitionStyle, IsBoot, IsSystem
```

```powershell
# 2. Full wipe: removes partitions and the RAW state (replace 2 with the disk number)
# If it returns the error "The disk has not been initialized", skip to step 3.
Clear-Disk -Number 2 -RemoveData -RemoveOEM
```

```powershell
# 3. Initialize the disk with a GPT partition table
Initialize-Disk -Number 2 -PartitionStyle GPT
```

```powershell
# 4. Create a partition using all the space, assign a letter and format as exFAT
New-Partition -DiskNumber 2 -UseMaximumSize -AssignDriveLetter | Format-Volume -FileSystem exFAT -NewFileSystemLabel "External_SSD"
```

```powershell
# 5. Check the result
Get-Volume
```

Notes:

- **exFAT** works well on Windows, macOS and Linux. For Windows-only use, switch to `-FileSystem NTFS`.
- On an **HDD**, add `-Full` to `Format-Volume` for a full format (slower, but it writes to the whole disk and works as an extra test). On an **SSD**, use the quick format.
- Formatting is not a secure erase. If the disk held sensitive data, use the manufacturer's secure erase tool.

## License

This project is licensed under the [MIT License](LICENSE).
