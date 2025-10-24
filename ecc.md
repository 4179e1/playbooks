# NVIDIA GPU Uncorrectable Error 

## Issue

You might notice th application crashed with `uncorrectable EcC error encountered`.

The syslog or dmesg would contain Xid with errors like 

```
# dmesg  | grep Xid
[430954.509105] NVRM: Xid (PCI:0000:52:00): 48, pid='<unknown>', name=<unknown>, An uncorrectable double bit error (DBE) has been detected on GPU in the framebuffer at physAddr 0x18f31d80 partition 16, subpartition 3.
[430954.532443] NVRM: Xid (PCI:0000:52:00): 63, pid='<unknown>', name=<unknown>, Row Remapper: New row (0x0000000018f31d80) marked for remapping, reset gpu to activate.
[430954.533994] NVRM: Xid (PCI:0000:52:00): 94, pid='<unknown>', name=<unknown>, Contained: SM (0x1). RST: No, D-RST: No
```

nvidia-smi shows Remapped Rows Pending

```
# nvidia-smi -q -d ECC,ROW_REMAPPER -i 52:00.0

==============NVSMI LOG==============

Timestamp                                 : Fri Oct 24 11:07:12 2025
Driver Version                            : 550.90.07
CUDA Version                              : 12.4

Attached GPUs                             : 8
GPU 00000000:52:00.0
    ECC Mode
        Current                           : Unknown Error
        Pending                           : Unknown Error
    ECC Errors
        Volatile
            SRAM Correctable              : N/A
            SRAM Uncorrectable Parity     : N/A
            SRAM Uncorrectable SEC-DED    : N/A
            DRAM Correctable              : N/A
            DRAM Uncorrectable            : N/A
        Aggregate
            SRAM Correctable              : N/A
            SRAM Uncorrectable Parity     : N/A
            SRAM Uncorrectable SEC-DED    : N/A
            DRAM Correctable              : N/A
            DRAM Uncorrectable            : N/A
            SRAM Threshold Exceeded       : N/A
        Aggregate Uncorrectable SRAM Sources
            SRAM L2                       : N/A
            SRAM SM                       : N/A
            SRAM Microcontroller          : N/A
            SRAM PCIE                     : N/A
            SRAM Other                    : N/A
    Remapped Rows
        Correctable Error                 : 0
        Uncorrectable Error               : 1
        Pending                           : Yes                      <===
        Remapping Failure Occurred        : No
        Bank Remap Availability Histogram
            Max                           : 2559 bank(s)
            High                          : 1 bank(s)
            Partial                       : 0 bank(s)
            Low                           : 0 bank(s)
            None                          : 0 bank(s)
```

## Recovery Procedure


1. Stop all processes which use GPU, you can identify them by `nvidia-smi`

2. Stop the follow services:

```
systemctl stop cmd
systemctl stop slurmd
systemctl stop nvsm
systemctl stop dcgm-exporter
systemctl stop nvidia-dcgm
systemctl stop nvidia-fabricmanager
systemctl stop nvidia-persistenced 
rmmod nvidia_drm nvidia_modeset
```

3.  Reset the GPU via
```
nvidia-smi -r -i <GPU ID>       # <GPU ID> are numbers like 1
# or 
nvidia-smi -r             # for all GPUs
```

> If there are still process using GPU, please identify them by
> `sudo lsof /dev/nvidia*`

4. Restart the services

```
systemctl start nvidia-persistenced 
systemctl start nvidia-fabricmanager
systemctl start nvidia-dcgm
systemctl start dcgm-exporter
systemctl start nvsm
systemctl start slurmd
systemctl start cmd
```

5. Run a diagnostic on the GPU
```
 dcgmi diag -r diagnostic,pcie -i <GPU ID> -p diagnostic.test_duration=60
```

If the command succeed, we are done here.

However, if it detects another ECC, start over from step 1. 
In this scenario, we might have to repeat the steps for up to 8 time and eventually get a `Remapping Failure Occurred        : Yes` in `nvidia-smi -q -d ECC,ROW_REMAPPER -i <GPU ID>`.
If this is happening, the GPU need RMA.