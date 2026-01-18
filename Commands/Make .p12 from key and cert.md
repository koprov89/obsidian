---
tags:
  - openssl
  - linux
  - commands
  - security
date: 2026-01-13
---

# Create .p12 Certificate from Key and Cert

Сделать сертификат при наличии бандла:

```bash
openssl pkcs12 -export -out justletswork.p12 -inkey <key file>  -in <cert bundle>
```

Сделать сертификат из того что получается с letsencrypt, работает на тачке Ansible

```bash
openssl pkcs12 -export -in <fullchain> -inkey <private key> -out <p12 output file> -name <whatever> -CAfile <chain file> -caname <whatever> -passout pass:<password>
```

## Related Notes
- [[гайды/XRAY сервер установка]]
