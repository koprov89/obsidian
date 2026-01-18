---
tags:
  - vmware
  - virtualization
  - commands
date: 2026-01-13
---

# OVFTool Import to vCenter

импортировать машину ovf в cCenter, в данном случае [[google_auth]]

```
ovftool -ds="vcenter"  "google_auth.ovf" "vi://Administrator@maxbit.local@10.50.0.246/ESXI_vcenter/host/10.50.0.247"
```
