---
tags:
  - security
  - scan
  - troubleshooting
  - proxmox
date: 2026-01-13
---

# Scan rdp.tizr.kz (57.128.216.206)

rdp.tizr.kz
57.128.216.206

22/tcp    open  ssh
25/tcp    open  smtp
111/tcp   open  rpcbind
3128/tcp  open  squid-http
8006/tcp  open  wpl-analytics
55445/tcp open  unknown



PORT      STATE SERVICE        VERSION

**22/tcp    open  ssh            OpenSSH 9.2p1 Debian 2+deb12u5 (protocol 2.0)**
| ssh-hostkey:
|   256 3e:ee:55:d8:80:0b:ed:0e:15:6f:c1:df:b4:38:ce:7d (ECDSA)
|_  256 cc:1d:ca:9d:a8:96:e4:43:97:e5:66:e6:ec:50:38:ea (ED25519)

**25/tcp    open  smtp           Postfix smtpd**
|_smtp-commands: kz-server, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, SMTPUTF8, CHUNKING
| ssl-cert: Subject: commonName=kz-server
| Subject Alternative Name: DNS:kz-server
| Not valid before: 2025-02-24T13:25:47
|_Not valid after:  2035-02-22T13:25:47
|_ssl-date: TLS randomness does not represent time

**111/tcp   open  rpcbind        2-4 (RPC #100000)**
| rpcinfo:
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|_  100000  3,4          111/udp6  rpcbind

**3128/tcp  open  http           Proxmox Virtual Environment REST API 3.0**
|_http-server-header: pve-api-daemon/3.0
|_http-title: Site doesn't have a title.

**8006/tcp  open  wpl-analytics?**
| fingerprint-strings:
|   HTTPOptions:
|     HTTP/1.0 501 method 'OPTIONS' not available
|     Cache-Control: max-age=0
|     Connection: close
|     Date: Mon, 14 Apr 2025 09:00:20 GMT
|     Pragma: no-cache
|     Server: pve-api-daemon/3.0
|     Expires: Mon, 14 Apr 2025 09:00:20 GMT
|   Help, Kerberos, TerminalServerCookie:
|     HTTP/1.0 400 bad request
|     Cache-Control: max-age=0
|     Connection: close
|     Date: Mon, 14 Apr 2025 09:00:36 GMT
|     Pragma: no-cache
|     Server: pve-api-daemon/3.0
|     Content-Length: 11
|     Expires: Mon, 14 Apr 2025 09:00:36 GMT
|     request
|   LDAPSearchReq, LPDString:
|     HTTP/1.0 400 bad request
|     Cache-Control: max-age=0
|     Connection: close
|     Date: Mon, 14 Apr 2025 09:00:47 GMT
|     Pragma: no-cache
|     Server: pve-api-daemon/3.0
|     Content-Length: 11
|     Expires: Mon, 14 Apr 2025 09:00:47 GMT
|     request
|   RTSPRequest:
|     HTTP/1.0 400 bad request
|     Cache-Control: max-age=0
|     Connection: close
|     Date: Mon, 14 Apr 2025 09:00:20 GMT
|     Pragma: no-cache
|     Server: pve-api-daemon/3.0
|     Content-Length: 11
|     Expires: Mon, 14 Apr 2025 09:00:20 GMT
|_    request

**55445/tcp open  ms-wbt-server  Microsoft Terminal Services**
| rdp-ntlm-info:
|   Target_Name: WIN-P2U2QQMCUF0
|   NetBIOS_Domain_Name: WIN-P2U2QQMCUF0
|   NetBIOS_Computer_Name: WIN-P2U2QQMCUF0
|   DNS_Domain_Name: WIN-P2U2QQMCUF0
|   DNS_Computer_Name: WIN-P2U2QQMCUF0
|   Product_Version: 10.0.20348
|_  System_Time: 2025-04-14T09:01:43+00:00
| ssl-cert: Subject: commonName=WIN-P2U2QQMCUF0
| Not valid before: 2025-02-24T10:13:08
|_Not valid after:  2025-08-26T10:13:08
|_ssl-date: 2025-04-14T09:01:45+00:00; 0s from scanner time.
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port8006-TCP:V=7.95%I=7%D=4/14%Time=67FCCEA4%P=x86_64-pc-linux-gnu%r(HT
SF:TPOptions,D7,"HTTP/1\.0\x20501\x20method\x20'OPTIONS'\x20not\x20availab
SF:le\r\nCache-Control:\x20max-age=0\r\nConnection:\x20close\r\nDate:\x20M
SF:on,\x2014\x20Apr\x202025\x2009:00:20\x20GMT\r\nPragma:\x20no-cache\r\nS
SF:erver:\x20pve-api-daemon/3\.0\r\nExpires:\x20Mon,\x2014\x20Apr\x202025\
SF:x2009:00:20\x20GMT\r\n\r\n")%r(RTSPRequest,E3,"HTTP/1\.0\x20400\x20bad\
SF:x20request\r\nCache-Control:\x20max-age=0\r\nConnection:\x20close\r\nDa
SF:te:\x20Mon,\x2014\x20Apr\x202025\x2009:00:20\x20GMT\r\nPragma:\x20no-ca
SF:che\r\nServer:\x20pve-api-daemon/3\.0\r\nContent-Length:\x2011\r\nExpir
SF:es:\x20Mon,\x2014\x20Apr\x202025\x2009:00:20\x20GMT\r\n\r\nbad\x20reque
SF:st")%r(Help,E3,"HTTP/1\.0\x20400\x20bad\x20request\r\nCache-Control:\x2
SF:0max-age=0\r\nConnection:\x20close\r\nDate:\x20Mon,\x2014\x20Apr\x20202
SF:5\x2009:00:36\x20GMT\r\nPragma:\x20no-cache\r\nServer:\x20pve-api-daemo
SF:n/3\.0\r\nContent-Length:\x2011\r\nExpires:\x20Mon,\x2014\x20Apr\x20202
SF:5\x2009:00:36\x20GMT\r\n\r\nbad\x20request")%r(TerminalServerCookie,E3,
SF:"HTTP/1\.0\x20400\x20bad\x20request\r\nCache-Control:\x20max-age=0\r\nC
SF:onnection:\x20close\r\nDate:\x20Mon,\x2014\x20Apr\x202025\x2009:00:36\x
SF:20GMT\r\nPragma:\x20no-cache\r\nServer:\x20pve-api-daemon/3\.0\r\nConte
SF:nt-Length:\x2011\r\nExpires:\x20Mon,\x2014\x20Apr\x202025\x2009:00:36\x
SF:20GMT\r\n\r\nbad\x20request")%r(Kerberos,E3,"HTTP/1\.0\x20400\x20bad\x2
SF:0request\r\nCache-Control:\x20max-age=0\r\nConnection:\x20close\r\nDate
SF::\x20Mon,\x2014\x20Apr\x202025\x2009:00:36\x20GMT\r\nPragma:\x20no-cach
SF:e\r\nServer:\x20pve-api-daemon/3\.0\r\nContent-Length:\x2011\r\nExpires
SF::\x20Mon,\x2014\x20Apr\x202025\x2009:00:36\x20GMT\r\n\r\nbad\x20request
SF:")%r(LPDString,E3,"HTTP/1\.0\x20400\x20bad\x20request\r\nCache-Control:
SF:\x20max-age=0\r\nConnection:\x20close\r\nDate:\x20Mon,\x2014\x20Apr\x20
SF:2025\x2009:00:47\x20GMT\r\nPragma:\x20no-cache\r\nServer:\x20pve-api-da
SF:emon/3\.0\r\nContent-Length:\x2011\r\nExpires:\x20Mon,\x2014\x20Apr\x20
SF:2025\x2009:00:47\x20GMT\r\n\r\nbad\x20request")%r(LDAPSearchReq,E3,"HTT
SF:P/1\.0\x20400\x20bad\x20request\r\nCache-Control:\x20max-age=0\r\nConne
SF:ction:\x20close\r\nDate:\x20Mon,\x2014\x20Apr\x202025\x2009:00:47\x20GM
SF:T\r\nPragma:\x20no-cache\r\nServer:\x20pve-api-daemon/3\.0\r\nContent-L
SF:ength:\x2011\r\nExpires:\x20Mon,\x2014\x20Apr\x202025\x2009:00:47\x20GM
SF:T\r\n\r\nbad\x20request");
Service Info: Host:  kz-server; OSs: Linux, Windows; CPE: cpe:/o:linux:linux_kernel, cpe:/o:microsoft:windows