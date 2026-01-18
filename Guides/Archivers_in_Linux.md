---
tags:
  - linux
  - archiver
  - commands
date: 2026-01-13
---

# Основные кейсы применения архиваторов в терминале Linux

## Введение
Архиваторы в Linux используются для сжатия и упаковки файлов. Основные инструменты: `tar`, `gzip`, `bzip2`, `xz`, `zip`, `unzip`. Ниже описаны основные кейсы.

## Создание архива с tar
`tar` - основной архиватор. Для создания архива:

```bash
tar -cvf archive.tar file1 file2 dir/
```

- `-c`: создать архив
- `-v`: verbose
- `-f`: указать имя файла

## Сжатие архива
### С gzip
```bash
tar -czvf archive.tar.gz file1 file2 dir/
```

- `-z`: использовать gzip

### С bzip2
```bash
tar -cjvf archive.tar.bz2 file1 file2 dir/
```

- `-j`: использовать bzip2

### С xz
```bash
tar -cJvf archive.tar.xz file1 file2 dir/
```

- `-J`: использовать xz

## Распаковка архива
### Распаковка tar
```bash
tar -xvf archive.tar
```

- `-x`: извлечь

### Распаковка сжатого tar
```bash
tar -xzvf archive.tar.gz  # для gzip
tar -xjvf archive.tar.bz2  # для bzip2
tar -xJvf archive.tar.xz   # для xz
```

## Просмотр содержимого архива
```bash
tar -tvf archive.tar
```

- `-t`: список файлов

## Работа с zip
### Создание zip-архива
```bash
zip archive.zip file1 file2 dir/
```

### Распаковка zip
```bash
unzip archive.zip
```

### Просмотр содержимого zip
```bash
unzip -l archive.zip
```

## Дополнительные кейсы
- **Сжатие одного файла**: `gzip file.txt` (создаст file.txt.gz)
- **Распаковка**: `gunzip file.txt.gz`
- **Рекурсивное сжатие**: `gzip -r dir/`
- **Проверка архива**: `tar -tvf archive.tar` для tar, `unzip -t archive.zip` для zip

## Советы
- Используйте `tar` для больших архивов с несколькими файлами.
- Для совместимости с Windows используйте `zip`.
- Проверьте дисковое пространство перед распаковкой.