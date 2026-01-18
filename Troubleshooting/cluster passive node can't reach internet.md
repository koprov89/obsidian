---
tags:
  - checkpoint
  - troubleshooting
  - networking
date: 2026-01-13
---

# Cluster Passive Node Can't Reach Internet

С какого-то момента дефолтное поведение.  Решается так:
Временно:

`fw ctl set int fwha_cluster_hide_active_only 0`

Постоянно:
Открываем файл **$FWDIR/boot/modules/fwkern.conf** и пишем туда
```
fwha_cluster_hide_active_only=0
```