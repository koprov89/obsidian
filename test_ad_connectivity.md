---
tags:
  - checkpoint
  - active-directory
  - troubleshooting
  - commands
date: 2026-01-13
---

# Test Active Directory Connectivity

$FWDIR/bin/test_ad_connectivity -u "checkpoint_service" -c "kDW68n7vFocYXxJjA2FD" -D "CN=checkpoint service,OU=_Service,OU=MB,DC=maxbit,DC=local" -d maxbit.local -i 192.168.153.251 -b "DC=maxbit,DC=local" -o test.txt -s
