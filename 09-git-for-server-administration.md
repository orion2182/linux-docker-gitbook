# 09. Git for Server Administration

> Config server tanpa Git = tidak ada history, tidak ada rollback. Bab ini Git seperlunya untuk admin, bukan developer.

## Tujuan Pembelajaran

- Init, commit, branch, pull, clone untuk config server
- SSH auth ke GitHub/GitLab + repo config
- Pola deploy berbasis Git yang aman

## 1. Git Basics

```bash
$ git --version
$ git config --global user.name "admin-vps"
$ git config --global user.email "admin@example.com"
$ git config --global init.defaultBranch main
$ git config --global pull.rebase false
```

Konsep: working dir → staging (`add`) → commit (snapshot) → remote (`push/pull`).

## 2. Repository

Mulai repo config di server / laptop:

```bash
$ mkdir -p ~/server-config && cd ~/server-config
$ git init
$ cat > README.md <<'EOF'
# server-config
Kumpulan config VPS + skrip. Tiap perubahan via commit.
EOF
$ git add README.md
$ git commit -m "init server-config"
$ git log --oneline -n 5
$ git status
```

Jangan `git init` di `/etc` langsung tanpa pola (lihat bagian Configuration Repository).

## 3. Branch

Branch = ruang aman coba config.

```bash
$ git branch
$ git checkout -b feat/nginx-tls
# ... edit nginx.conf ...
$ git add nginx/nginx.conf
$ git commit -m "feat: tambah TLS staging"
$ git checkout main
$ git merge feat/nginx-tls
$ git branch -d feat/nginx-tls
```

Aturan: `main` selalu bisa deploy. Eksperimen di branch.

## 4. Commit

Commit kecil, pesan jelas (imperatif):

```bash
$ git add -p
$ git commit -m "fix: naikkan client_max_body_size ke 20m"
$ git log --oneline -n 10
$ git show HEAD --stat
```

Format: `feat:`, `fix:`, `chore:`, `docs:` + apa + kenapa (bukan cuma "update").

## 5. Pull

Ambil + gabung perubahan remote.

```bash
$ git pull --ff-only
$ git fetch origin && git log --oneline main..origin/main
$ git pull --rebase --autostash
```

Di server production: `pull --ff-only` agar gagal daripada auto-merge berantakan. Resolve konflik di laptop, bukan di prod.

## 6. Clone

```bash
$ git clone git@github.com:org/server-config.git /opt/server-config
$ cd /opt/server-config && git remote -v
$ git clone --depth 1 git@github.com:org/app.git /opt/app
```

`--depth 1` untuk deploy cepat hemat disk. Untuk audit butuh full history, clone penuh.

## 7. SSH Authentication

Jangan pakai password/HTTPS token di server kalau bisa SSH key deploy.

```bash
$ ssh-keygen -t ed25519 -C "vps-prod-01-deploy" -f ~/.ssh/id_deploy
$ cat ~/.ssh/id_deploy.pub
# tempel ke Deploy Keys repo (read-only kalau cuma pull)
$ cat ~/.ssh/config
# Host github.com
#   IdentityFile ~/.ssh/id_deploy
$ ssh -T git@github.com
$ GIT_SSH_COMMAND="ssh -i ~/.ssh/id_deploy -o StrictHostKeyChecking=yes" git clone git@github.com:org/server-config.git
```

Permission: `chmod 600` private key, `chmod 644` public. Beda key per server.

## 8. Configuration Repository

Pola yang rapi:

```text
server-config/
  nginx/app.conf
  docker/compose.prod.yaml
  scripts/backup.sh
  systemd/backup.service
  README.md (cara apply tiap file)
```

Apply manual tapi terdokumentasi:

```bash
$ cd /opt/server-config && git pull --ff-only
$ sudo cp nginx/app.conf /etc/nginx/sites-enabled/app.conf
$ sudo nginx -t && sudo systemctl reload nginx
```

Naik level: symlink + skrip `apply.sh` yang copy + test + reload + rollback via `git revert`. Jangan auto-pull tanpa test di prod.

> Jangan commit secret (`.env`, `*.pem`, `id_*`)! Pakai `.gitignore` + secret manager (Bab 21).

```bash
$ cat .gitignore
# .env
# *.pem
# *.key
# .ssh/
```

## 9. Git-based Deployment

Alur minimal yang aman:

1. Push ke `main` dari laptop (CI lolos).
2. Di server: `git fetch && git diff main..origin/main` (review dulu).
3. `git pull --ff-only`, `docker compose up -d --build` (Bab 17/21).
4. Healthcheck + `docker ps`, gagal → `git reset --hard HEAD~1` + up lagi.
5. Tag rilis: `git tag v1.4.0 && git push --tags` untuk rollback jelas.

```bash
$ cd /opt/app
$ git fetch origin
$ git log --oneline -n 5
$ git pull --ff-only
$ sudo docker compose up -d --build
$ sudo docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

Untuk multi-server / zero-downtime, lanjut ke CI/CD + registry (Bab 23), Git di sini fondasinya.

## Latihan

1. Buat repo `server-config`, commit 3 config dummy dengan pesan yang benar.
2. Buat branch, rusak 1 config, merge kembali dengan benar.
3. Setup deploy key read-only, clone via SSH di VM lab.
4. Tulis `apply.sh` yang pull + `nginx -t` + reload, gagal → batal.

## Rangkuman

- Git = mesin waktu config server.
- Branch untuk coba, `main` selalu deployable, `pull --ff-only` di prod.
- Key per server, secret tidak pernah di-commit.
