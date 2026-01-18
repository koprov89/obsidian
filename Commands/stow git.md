---
tags:
  - git
  - stow
  - dotfiles
  - linux
  - commands
date: 2026-01-13
---

# Git Stow Dotfiles Management

Копируем конфиг
```bash
 cp -r ~/.config/someconfig .config/
```
 
 для удаления вложенных репозиториев. Плюс в конце нужен чтобы все делать за одну команду rm, тупо быстрее. 
 ```bash
find . -type d -name .git ! -path ./.git -exec rm -rf {} +
```

Если без +, то можно было бы сделать так. Эффект тот же

```bash
find . -type d -name .git ! -path ./.git -exec rm -rf {} \;
```

Да, я каждый раз это забываю
```bash
git init
git branch -m main
git remote add origin git@github.com:koprov89/dotfiles.git
git add .
git commit -m 'comment'
git push -u origin main
```