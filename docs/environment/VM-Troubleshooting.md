# VM Freezing Issue Resolution

## Problem
Ubuntu VMs (both 22.04 LTS and 24.04 LTS) experienced kernel soft lockups during installation and operation.

## Root Cause
VMware Tools incompatibility with newer Ubuntu kernels causing watchdog soft lockup errors.

## Resolution
1. Force restart VM when frozen
2. Immediately disable open-vm-tools service
3. Reboot system
4. System runs stable without VMware Tools

## Note
VMware Tools not required for lab environment - basic functionality works without it.
