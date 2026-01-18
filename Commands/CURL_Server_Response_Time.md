---
tags:
  - curl
  - linux
  - commands
  - networking
date: 2026-01-13
---

# CURL Server Response Time Test

Тест времени ответа сервера

```bash
curl -s -w 'Testing Website Response Time for: %{url_effective}\n\nLookup Time:\t\t%{time_namelookup}\nConnect Time:\t\t%{time_connect}\nAppCon Time:\t\t%{time_appconnect}\nRedirect Time:\t\t%{time_redirect}\nPre-transfer Time:\t%{time_pretransfer}\nStart-transfer Time:\t%{time_starttransfer}\n\nTotal Time:\t\t%{time_total}\n' -o /dev/null https://dev.monro.kube.dev001.ru
```
