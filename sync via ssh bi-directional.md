---
tags:
  - ssh
  - unison
  - linux
  - systemd
  - config
date: 2026-01-13
---

# Bi-directional Sync via SSH with Unison

Используем утилиту unison. Она есть в репозиториях. Утилита должна быть установлена на обоих сервера - кто сихронизируется и с кем

В директории  **~/.config/systemd/user**
Создаем сервис в пространстве пользователя 

```bash
[Unit]
Description=Synology sync

[Service]
ExecStart=/usr/bin/unison -silent -auto ssh://gospodinnikto@gospodinnikto.synology.me//var/services/homes/gospodinnikto/Obsidian /home/aleksey/Obsidian
```

И таймер чтобы этот сервис запускать 
```bash
[Unit]
Description=Run synology_sync every 10 seconds

[Timer]
OnBootSec=10sec
OnUnitActiveSec=10sec
Unit=synology_sync.service

[Install]
WantedBy=default.target
```