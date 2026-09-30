<p align="right"><a href="README.pt-BR.md">🇧🇷 Português</a></p>

# disk-health-cheatsheet

Quick reference for checking HDD and SSD health and integrity using terminal and PowerShell commands.

## Table of Contents

- [Before you start](#before-you-start)
- [Windows (PowerShell / CMD)](#windows-powershell--cmd)
- [Linux (terminal)](#linux-terminal)
- [How to read SMART data](#how-to-read-smart-data)
- [Warning signs](#warning-signs)
- [License](#license)

## Before you start

- Most commands need **administrator** (Windows) or **root/sudo** (Linux).
- **Back up your data first** if the disk already shows problems. Repair tools can make a failing disk worse.
- Commands marked as *read-only* do not change anything on the disk.
- Replace `/dev/sdX`, `/dev/nvme0` and `C:` with your own device or drive letter.

## Windows (PowerShell / CMD)

Open PowerShell **as Administrator**.

### Quick health overview

```powershell
# Health and operational status of all physical disks
Get-PhysicalDisk | Select-Object FriendlyName, MediaType, HealthStatus, OperationalStatus, Size
```

```powershell
# Temperature, wear, error counters and power-on hours (read-only)
Get-PhysicalDisk | Get-StorageReliabilityCounter |
  Select-Object DeviceId, Temperature, Wear, ReadErrorsTotal, WriteErrorsTotal, PowerOnHours
```

```powershell
# List disks, partitions and volumes
Get-Disk
Get-Partition
Get-Volume
```

### File system check

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
winsat disk -drive c
```

### Full SMART data on Windows

Windows does not show full SMART attributes natively. Options:

- Install [smartmontools](https://www.smartmontools.org/) and use the same `smartctl` commands as in the Linux section.
- Use a GUI tool such as CrystalDiskInfo.

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

| ID  | Attribute                | What it means                                                    |
|-----|--------------------------|------------------------------------------------------------------|
| 5   | Reallocated_Sector_Ct    | Bad sectors already remapped. Rising values are a bad sign.      |
| 9   | Power_On_Hours           | Total hours the disk has been powered on.                        |
| 177 | Wear_Leveling_Count      | SSD wear (meaning varies by manufacturer).                       |
| 187 | Reported_Uncorrect       | Errors that could not be corrected.                              |
| 194 | Temperature_Celsius      | Current temperature.                                             |
| 197 | Current_Pending_Sector   | Sectors waiting to be remapped. Should be 0.                     |
| 198 | Offline_Uncorrectable    | Sectors that failed during offline scan. Should be 0.            |
| 199 | UDMA_CRC_Error_Count     | Usually a bad **cable or connection**, not the disk itself.      |
| 233 | Media_Wearout_Indicator  | Remaining SSD life (Intel and some others).                      |

### NVMe fields

| Field                     | What it means                                   |
|---------------------------|-------------------------------------------------|
| Critical Warning          | Should be `0`.                                  |
| Percentage Used           | SSD life used (100% = rated endurance reached). |
| Available Spare           | Spare blocks left. Should be well above threshold. |
| Media and Data Integrity Errors | Should be `0`.                            |

## Warning signs

- SMART overall health says **FAILED**
- `Reallocated_Sector_Ct`, `Current_Pending_Sector` or `Offline_Uncorrectable` above 0 and growing
- Repeated I/O errors in `dmesg` or the Windows event log
- Very slow reads, freezes, or unusual clicking noises (HDD)
- SSD `Percentage Used` close to 100% or `Available Spare` near the threshold

If you see any of these, **back up your data immediately** and plan a replacement.

## License

This project is licensed under the [MIT License](LICENSE).
