# HomeHub on a Raspberry Pi 3 (Debian 11, native virtualenv)

This guide runs HomeHub directly from a Python **virtualenv** on a 64-bit Debian 11
(bullseye) Raspberry Pi 3. Docker is **not** required. Gunicorn serves the app on
`127.0.0.1:5000`, and **Nginx** publishes it on port 80, so you open
`http://<pi-ip>` instead of `http://<pi-ip>:5000`.

Docker was only used as a local Debian 11 arm64 test bed while writing this guide; the
verification results are at the bottom.

If you would rather automate steps 2–10, use the Ansible playbook in `deploy/ansible/`.
It supports both 64-bit `aarch64` and `x86_64` targets; see
[Deploy with Ansible](#15-deploy-with-ansible).

> **Read this first**
>
> - **Debian 11 reached end-of-life on 31 August 2026.** It no longer receives security
>   updates. The steps below keep an existing Bullseye machine working, but if you are
>   installing from scratch, use Debian 12 or 13 instead (the same steps apply, minus the
>   archive-repository section).
> - **Use a 64-bit OS.** Run `uname -m`; it must print `aarch64`. If it prints `armv7l`,
>   you have a 32-bit system. The current Python packages no longer publish 32-bit ARM
>   wheels, so a 32-bit install means compiling SQLAlchemy and Pillow on the Pi, which is
>   not covered here. Reinstall with 64-bit Debian 11 or 64-bit Raspberry Pi OS.
> - This guide targets the **Pi 3 Model B/B+ (1 GB RAM)**. The Model A+ has 512 MB; it can
>   work, but close other services and expect less headroom.
> - HomeHub is designed for a **trusted home LAN**. The Nginx setup below is a plain HTTP
>   reverse proxy; it does not add HTTPS. Do not port-forward it to the internet.

---

## 1. Check the operating system

```bash
uname -m          # must be aarch64
cat /etc/debian_version
getconf LONG_BIT  # must be 64
free -h
df -h /
```

You should have at least 2 GB free for the app, Python, and Python packages; 4 GB is
comfortable. A Pi 3 with the Lite (no desktop) image is the best fit.

---

## 2. Point apt at a working Debian mirror

Because Debian 11 is EOL, `deb.debian.org` and `security.debian.org` no longer serve usable
Bullseye indexes. The Yandex mirror is usually faster than the official archive, so list it
first and keep `archive.debian.org` as a fallback. Apt reads the file from the top down:

```bash
# Keep a copy of the old list
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak

# Replace the Debian entries; keep any archive.raspberrypi.org line
sudo tee /etc/apt/sources.list >/dev/null <<'EOF'
deb http://mirror.yandex.ru/debian bullseye main
deb http://mirror.yandex.ru/debian bullseye-updates main
deb http://mirror.yandex.ru/debian-security bullseye-security main

deb http://archive.debian.org/debian bullseye main
deb http://archive.debian.org/debian bullseye-updates main
deb http://archive.debian.org/debian-security bullseye-security main
EOF

# On Raspberry Pi OS, add your existing line back, for example:
# grep 'raspberrypi.org' /etc/apt/sources.list.bak | sudo tee -a /etc/apt/sources.list

# Bullseye's Release files are past Valid-Until on every mirror
echo 'Acquire::Check-Valid-Until "false";' | \
  sudo tee /etc/apt/apt.conf.d/99homehub-archive

sudo apt-get -o Acquire::Retries=5 update
```

If your file had no active Debian lines (only comments), the heredoc above is enough; add
your Raspberry Pi line back afterwards.

The Yandex mirror is a copy of the same Debian archive. Apt still verifies the repository
signatures, so this does not weaken package verification. There will be no new security
updates for Debian 11. If `mirror.yandex.ru` is unreachable from your network, delete its
three lines and run `sudo apt-get update` again to use the archive alone.

---

## 3. Install the system tools

```bash
sudo DEBIAN_FRONTEND=noninteractive apt-get -o Acquire::Retries=5 install -y \
  ca-certificates curl git tzdata ffmpeg ghostscript
```

`ffmpeg` and `ghostscript` are only needed for the Media Downloader and PDF Compressor.
You can leave them out if you will not use those features, but the guide assumes the full
set.

Set the timezone so due dates, reminders, and recurring items use the correct "today":

```bash
sudo timedatectl set-timezone Europe/Berlin   # use your own timezone
date
```

A list of names is available with `timedatectl list-timezones`.

---

## 4. Install Python 3.12 without compiling it

Debian 11 ships Python 3.9. HomeHub's pinned packages need Python 3.11 or newer, so use
[`uv`](https://docs.astral.sh/uv/), which downloads a prebuilt 64-bit Python 3.12 quickly,
without compiling it. No compiler or `python3-dev` packages are needed.

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source "$HOME/.local/bin/env"
uv --version
uv python install 3.12
```

If `source` is not available in your shell, log out and back in, or run:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

---

## 5. Get HomeHub and create the virtualenv

Run this as your normal user, **not** as root:

```bash
cd "$HOME"
git clone https://github.com/surajverma/homehub.git
cd homehub

uv venv --python 3.12 .venv
source .venv/bin/activate
uv pip install -r requirements.txt

python -V
```

`python -V` should report Python 3.12.x. `which python` should point at
`/home/<you>/homehub/.venv/bin/python`.

This venv does not include `pip` by default; use `uv pip` for package commands. If you
prefer plain `pip`, create the venv with `uv venv --python 3.12 --seed .venv` instead.

The venv uses the Python that `uv` installed under `~/.local/share/uv/`. Keep that
directory; deleting it will break the venv. Do not commit `.venv` to git (it is already
ignored).

---

## 6. Build the stylesheet

The project does not keep the compiled `static/output.css` in git, so it must be built
once. On a Pi 3, use the standalone Tailwind binary instead of installing Node.js:

```bash
cd "$HOME/homehub"

curl -fsSL -o "$HOME/.local/bin/tailwindcss" \
  https://github.com/tailwindlabs/tailwindcss/releases/download/v3.4.19/tailwindcss-linux-arm64
chmod +x "$HOME/.local/bin/tailwindcss"

tailwindcss -i ./static/input.css -o ./static/output.css --minify
ls -lh static/output.css
```

If `tailwindcss: command not found`, use the full path:
`"$HOME/.local/bin/tailwindcss" -i ./static/input.css -o ./static/output.css --minify`.

The `Browserslist: caniuse-lite is outdated` message is harmless. Rebuild the CSS after
pulling code changes that add or change Tailwind classes.

If you already have Node.js 14 or newer, `npm install && npm run build:css` also works,
but it is unnecessary on a Pi.

---

## 7. Configure HomeHub

```bash
cd "$HOME/homehub"
cp config-example.yml config.yml
nano config.yml
```

At minimum, edit the instance name and family members. A minimal example:

```yaml
instance_name: "Family Hub"
password: "use-a-long-site-password"   # "" = no site password
family_members:
  - Alice
  - Bob
```

- `password` protects the whole site. Set one unless the Pi is on a network you fully
  trust.
- The **admin password** is separate and is set in step 11.
- On a small SD card, lower `max_upload_mb` (for example `256`) so uploads cannot fill the
  card.
- `config.yml` is gitignored and is re-read on the next page load; a restart is normally
  not needed.
- The site password is stored as plain text in `config.yml`, so keep that file private.
- For every option (feature toggles, themes, weather, languages), see the main
  [README.md](README.md).

Compile the translation catalogs (the Docker image normally does this during its build):

```bash
source .venv/bin/activate
flask translations compile
```

This is optional if `language: en` is the only language you use, but it is quick and
prevents a fallback to English later.

---

## 8. First start (foreground test)

```bash
cd "$HOME/homehub"
source .venv/bin/activate
python run.py
```

Find the Pi's address in another terminal:

```bash
hostname -I
```

Open `http://<pi-ip>:5000` in a browser on the same network. This port is temporary; after
step 10 the dashboard is at `http://<pi-ip>`. The health endpoint should answer:

```bash
curl -fsS http://127.0.0.1:5000/healthz
# {"ok":true}
```

Stop the test server with `Ctrl+C`. `python run.py` uses Flask's development server, so it
is fine for a quick check but should not be the permanent service. Use systemd below. In
the permanent setup, gunicorn listens only on localhost and Nginx is the only service
exposed to the LAN.

---

## 9. Run HomeHub permanently with systemd

The Docker image runs one gunicorn worker with four threads. Keep that shape natively:
the app has an in-process lock for recurring items and uses background threads for media
downloads, so do not increase the worker count.

Run the block below **as your normal user** (only the `tee` command uses `sudo`). It fills
in your username and home directory:

```bash
sudo tee /etc/systemd/system/homehub.service >/dev/null <<EOF
[Unit]
Description=HomeHub family dashboard
After=network.target

[Service]
Type=simple
User=$(id -un)
Group=$(id -gn)
WorkingDirectory=$HOME/homehub
Environment=PYTHONUNBUFFERED=1
Environment=PATH=$HOME/homehub/.venv/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
ExecStart=$HOME/homehub/.venv/bin/gunicorn wsgi:app -w 1 -k gthread --threads 4 -b 127.0.0.1:5000 --access-logfile - --error-logfile -
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

If you installed HomeHub somewhere other than `~/homehub`, edit the `WorkingDirectory` and
`ExecStart` paths before continuing.

The `PATH` line is required for the **Media Downloader**. Gunicorn is started from
`.venv/bin`, but the downloader runs `yt-dlp` by name, so the executable is looked up on
`PATH`. systemd services get a default `PATH` that does not include `.venv/bin`; without the
line above every download fails immediately and the page shows **Not available**. The
heredoc is unquoted, so `$HOME` is expanded when you run it. If you hand-edit the unit
instead, replace `$HOME` with your real home directory—systemd does not expand it there.

The bind address is `127.0.0.1` on purpose: gunicorn should not be reachable from other
machines. Nginx, configured in the next step, will forward requests to it.

Enable and start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now homehub
sudo systemctl status homehub --no-pager
```

Check it again:

```bash
curl -fsS http://127.0.0.1:5000/healthz
```

Follow the logs with:

```bash
sudo journalctl -u homehub -f
```

If the service fails, the most common causes are the wrong `WorkingDirectory`, the wrong
`User`, or a `data/` directory owned by another user—see Troubleshooting.

If you already created the unit from an earlier version of this guide, do not recreate it.
Add the missing `PATH` with a drop-in instead (replace `<user>`):

```bash
sudo systemctl edit homehub
```

In the editor, add:

```ini
[Service]
Environment=PATH=/home/<user>/homehub/.venv/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

Save, then reload and restart:

```bash
sudo systemctl daemon-reload
sudo systemctl restart homehub
```

At this point HomeHub answers only on the Pi itself. Continue to the next step to make it
reachable from other devices through Nginx.

---

## 10. Put HomeHub behind Nginx

Nginx listens on the normal HTTP port (80) and forwards requests to gunicorn on
`127.0.0.1:5000`. That gives you `http://<pi-ip>` with no port number, and keeps gunicorn
off the LAN.

Install Nginx:

```bash
sudo DEBIAN_FRONTEND=noninteractive apt-get -o Acquire::Retries=5 install -y nginx
```

Write the site configuration. The heredoc is quoted, so it is copied exactly as shown:

```bash
sudo tee /etc/nginx/sites-available/homehub >/dev/null <<'EOF'
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;

    # Keep this at least as large as max_upload_mb in config.yml.
    client_max_body_size 1024m;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_http_version 1.1;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 5s;
        proxy_read_timeout 300s;
        proxy_send_timeout 300s;
    }
}
EOF
```

`client_max_body_size` is Nginx's own upload limit. The example matches the default
`max_upload_mb: 1024`; if you lowered that in `config.yml`, lower this to the same value
(for example `256m`). If `max_upload_mb` is `0` (no app limit), Nginx still needs a finite
cap, so pick the largest upload you want to allow.

Enable the site, remove the default welcome page, and check the configuration:

```bash
sudo ln -sf /etc/nginx/sites-available/homehub /etc/nginx/sites-enabled/homehub
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
```

`nginx -t` must say `syntax is ok` and `test is successful`. Then start Nginx and apply
the configuration:

```bash
sudo systemctl enable --now nginx
sudo systemctl reload nginx
```

Test through Nginx:

```bash
curl -fsS http://127.0.0.1/healthz
# {"ok":true}
```

Open `http://<pi-ip>` in a browser on another device. You should see HomeHub without a
`:5000` suffix.

Confirm that only Nginx is exposed on the network:

```bash
sudo ss -ltnp | grep -E ':80|:5000'
```

You want Nginx listening on `*:80` (and `[::]:80`) and gunicorn on `127.0.0.1:5000`.
If gunicorn shows `0.0.0.0:5000` or `*:5000`, the systemd unit still has the old bind
address; edit it and run `sudo systemctl daemon-reload && sudo systemctl restart homehub`.

Nginx logs are in `/var/log/nginx/access.log` and `/var/log/nginx/error.log`; service
logs remain in `journalctl -u homehub`. Nginx does not add HTTPS, so keep this on your
trusted LAN.

---

## 11. Set the admin password (optional)

By default anyone can pick the admin user from the user switcher. To require a password:

```bash
cd "$HOME/homehub"
source .venv/bin/activate
flask set-admin-password
```

Run it as the same user the service runs as, otherwise `data/app.db` can become owned by
root. To remove it:

```bash
flask set-admin-password --clear
```

The site password in `config.yml` and the admin password are independent.

---

## 12. Updates and backups

### Back up first

Stop the service so SQLite is not writing while the files are copied:

```bash
sudo systemctl stop homehub
cd "$HOME"
tar -czf "homehub-backup-$(date +%F).tar.gz" \
  -C "$HOME/homehub" config.yml data uploads media pdfs
sudo systemctl start homehub
```

`data/` contains the database and `secret_key`; `uploads/`, `media/`, and `pdfs/` contain
user files. Keep a copy off the Pi.

### Update

```bash
cd "$HOME/homehub"
source .venv/bin/activate

git pull
uv pip install -r requirements.txt
tailwindcss -i ./static/input.css -o ./static/output.css --minify
flask translations compile

sudo systemctl restart homehub
sudo systemctl status homehub --no-pager
```

If `tailwindcss` is missing (for example after clearing `~/.local/bin`), re-run the
download command from step 6.

### Restore

```bash
sudo systemctl stop homehub
tar -xzf "$HOME/homehub-backup-YYYY-MM-DD.tar.gz" -C "$HOME/homehub"
sudo systemctl start homehub
```

---

## 13. Pi 3 tuning and storage

- **Use the Lite image.** A desktop environment consumes most of the 1 GB before HomeHub
  starts.
- **Watch memory during installs.** HomeHub itself is light when idle, but `uv pip install`
  and the Tailwind build are the memory peaks. If the kernel logs OOM kills
  (`sudo dmesg | grep -i oom`), add or enlarge swap:

  ```bash
  free -h
  sudo fallocate -l 1G /swapfile
  sudo chmod 600 /swapfile
  sudo mkswap /swapfile
  sudo swapon /swapfile
  echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
  ```

  Raspberry Pi OS already has a small swap file; use `sudo raspi-config` → Performance
  Options → Change Swap Size instead of adding a second one.

- **Avoid heavy SD-card writes.** A USB SSD is a good place for `~/homehub`, especially if
  you use uploads and media. At minimum, keep backups.
- **Do not put `data/` on NFS or SMB.** SQLite WAL expects a local filesystem. If you must
  use a network share, add `Environment=SQLITE_JOURNAL_MODE=DELETE` to the systemd unit
  and restart the service.
- **Disable features you do not need** in `config.yml` to save CPU and RAM:

  ```yaml
  feature_toggles:
    media_downloader: false
    pdf_compressor: false
  ```

  Media downloads and PDF compression are slow on a Pi 3; they work, but high-resolution
  video merging can take a long time.
- **Keep one gunicorn worker.** The `-w 1 -k gthread --threads 4` settings in the unit
  match the supported production setup. Do not raise `-w`.
- **Use a DHCP reservation** in your router (or a static address) so the bookmark to
  `http://<pi-ip>` keeps working.
- **Nginx is lightweight** and adds only a few MB of memory; keep gunicorn at one worker as
  described above.

---

## 14. Troubleshooting

**`uname -m` prints `armv7l`**

You are on a 32-bit OS. This guide needs 64-bit (`aarch64`). Back up your data and
reinstall 64-bit Debian 11 or Raspberry Pi OS. Thirty-two-bit ARM is not covered.

**`404 Not Found` from apt**

Debian 11 is EOL and the normal security pool is gone. Do step 2 and keep both the
`mirror.yandex.ru` and `archive.debian.org` entries. If apt reports `Release file ... is not
valid yet` or `Valid-Until`, confirm that `/etc/apt/apt.conf.d/99homehub-archive` contains
`Acquire::Check-Valid-Until "false";`. If one mirror is unreachable, remove its lines and run
`sudo apt-get update` again.

**`uv: command not found`**

Run `source "$HOME/.local/bin/env"` or open a new shell. The binary is installed in
`~/.local/bin/uv`.

**`No module named ...` or `flask: command not found`**

The venv is not active. Run:

```bash
cd "$HOME/homehub"
source .venv/bin/activate
uv pip install -r requirements.txt
```

**`gunicorn` service fails or restarts**

Check `sudo journalctl -u homehub -n 100 --no-pager`. Confirm the paths in the unit and
that `~/.local/share/uv/python` still exists. Run the same `gunicorn` command by hand from
the app directory to see the error.

**Media Downloader shows `Not available`**

That chip is the generic failure state; the background worker currently discards yt-dlp's
error message, so the journal may show nothing useful. Check what the service actually
sets (replace `<user>`):

```bash
sudo systemctl cat homehub | grep -i '^Environment='
sudo -u <user> env -i PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin \
  sh -c 'command -v yt-dlp || echo "yt-dlp not found on the service PATH"'
```

If it prints `yt-dlp not found`, add the `Environment=PATH=...` line from step 9 (or the
`systemctl edit` drop-in), then `sudo systemctl daemon-reload && sudo systemctl restart homehub`.

If `yt-dlp` is found, the site or the link is the problem. Run the same URL by hand to see
the real message:

```bash
cd "$HOME/homehub"
.venv/bin/yt-dlp -F "<the-url>"
```

MP3 extraction and MP4 merging also need `ffmpeg`, which step 3 installs.

**`502 Bad Gateway` from Nginx**

Nginx is running, but the app is not answering on `127.0.0.1:5000`. Check:

```bash
curl -fsS http://127.0.0.1:5000/healthz
sudo systemctl status homehub --no-pager
sudo tail -n 50 /var/log/nginx/error.log
sudo journalctl -u homehub -n 100 --no-pager
```

If you changed the gunicorn port, update `proxy_pass` in
`/etc/nginx/sites-available/homehub` to match, then run
`sudo nginx -t && sudo systemctl reload nginx`.

**Nginx shows the default "Welcome to nginx!" page**

The default site is still enabled. Remove it and reload:

```bash
sudo rm -f /etc/nginx/sites-enabled/default
sudo ln -sf /etc/nginx/sites-available/homehub /etc/nginx/sites-enabled/homehub
sudo nginx -t
sudo systemctl reload nginx
```

**`413 Request Entity Too Large` from Nginx**

Nginx's `client_max_body_size` is smaller than the upload. Set it in
`/etc/nginx/sites-available/homehub` to at least the `max_upload_mb` value in
`config.yml` (for example `256m` for 256 MB), then run
`sudo nginx -t && sudo systemctl reload nginx`.

**Port 5000 is already in use**

Check with `sudo ss -ltnp | grep :5000`. Either stop the other service or change the
gunicorn bind address to `127.0.0.1:8080` in the unit, change `proxy_pass` in the Nginx
site to `http://127.0.0.1:8080`, and restart both. For a quick test,
`PORT=8080 python run.py` works too.

**Port 80 is already in use**

Another web server, such as Apache, may be installed. Check with
`sudo ss -ltnp | grep :80`. Stop and disable the other server, or change Nginx's `listen`
lines to `8080` and open `http://<pi-ip>:8080` instead. Run
`sudo nginx -t && sudo systemctl reload nginx` after editing the file.

**`database is locked`**

Do not run `python run.py` and the systemd service at the same time. The service should use
one worker. If `data/` is on a network share, move it to local storage or set
`SQLITE_JOURNAL_MODE=DELETE`.

**`Permission denied` writing to `data/`, `uploads/`, `media/`, or `pdfs/`**

The service user must own the app directory:

```bash
sudo chown -R "$USER":"$USER" "$HOME/homehub"
sudo systemctl restart homehub
```

**Wrong date or time**

Set the system timezone with `sudo timedatectl set-timezone <Area/City>`, then
`sudo systemctl restart homehub`.

**`No space left on device`**

Run `df -h` and `du -sh ~/homehub/*`. Clear the uv cache with `uv cache clean`, remove old
backups, and lower `max_upload_mb` in `config.yml`.

**`cannot execute binary file` or `Exec format error` from the Tailwind binary**

You downloaded the wrong architecture. On a 64-bit Pi it must be
`tailwindcss-linux-arm64`, not `tailwindcss-linux-x64`.

**`Illegal instruction` when Python starts**

The `uv` Python build is for ARMv8. A Pi 3 supports it on a 64-bit OS; a 32-bit OS or a
different ARM board may not.

---

## 15. Deploy with Ansible

The manual steps above are the reference implementation. `deploy/ansible/` contains a
builtin-only Ansible role that performs steps 2–10 on Debian 11, 12, or 13. It is idempotent
and architecture-aware: the same playbook works on a 64-bit Raspberry Pi (`aarch64`) and a
normal 64-bit server (`x86_64`), and it refuses a 32-bit `armv7l` target before making
changes. It installs HomeHub natively with a virtualenv, gunicorn, and Nginx; Docker is not
used.

### Requirements

- A 64-bit target: `uname -m` must print `aarch64` or `x86_64`.
- Debian 11, 12, or 13. Debian 11 gets the EOL apt fix from step 2 automatically.
- SSH access and an account that can run `sudo`. Add `-K` if sudo prompts for a password.
- Outbound HTTPS to GitHub, `astral.sh`, and the Tailwind release page.
- Port 80 free if `homehub_manage_nginx` is true (the default).

The role was tested with `ansible-core` 2.18.1. It uses builtin modules only, so no Galaxy
collections are required.

### Files

```text
deploy/ansible/
├── ansible.cfg
├── inventory.example.yml
├── group_vars/all.yml
├── site.yml
└── roles/homehub/
    ├── defaults/main.yml
    ├── handlers/main.yml
    ├── tasks/
    │   ├── main.yml
    │   ├── preflight.yml
    │   ├── user.yml
    │   ├── apt.yml
    │   ├── app.yml
    │   ├── build.yml
    │   ├── service.yml
    │   ├── nginx.yml
    │   └── verify.yml
    └── templates/
        ├── homehub.service.j2
        └── homehub.nginx.j2
```

### Inventory

```yaml
all:
  children:
    homehub:
      hosts:
        pi:
          ansible_host: 192.168.1.50
          ansible_user: pi

        # test-server:
        #   ansible_host: 192.168.31.55
        #   ansible_user: dtsirul

      vars:
        homehub_user: "{{ ansible_user }}"
        # Pin a tag for reproducible deploys; a branch follows new commits.
        homehub_version: main
        homehub_server_name: "_"
```

Copy it and edit the hosts:

```bash
cd deploy/ansible
cp inventory.example.yml inventory.yml
$EDITOR inventory.yml
```

`homehub_user` defaults to the SSH user. The playbook does not create accounts unless
`homehub_create_user: true`; by default the account must already exist. `homehub_group`
defaults to `homehub_user`; set it if the account's primary group has a different name.

### Key variables

| Variable | Default | Purpose |
| --- | --- | --- |
| `homehub_user` | `ansible_user` | account that owns the checkout and runs the service |
| `homehub_group` | `homehub_user` | group that owns the checkout |
| `homehub_create_user` | `false` | create `homehub_user` if it is missing |
| `homehub_home` | `~<user>/homehub` | application directory |
| `homehub_version` | `main` | git tag, branch, or commit to deploy |
| `homehub_python_version` | `3.12` | Python version installed by `uv` |
| `homehub_bind` / `homehub_port` | `127.0.0.1` / `5000` | gunicorn binding |
| `homehub_server_name` | `_` | Nginx `server_name` |
| `homehub_nginx_port` | `80` | Nginx listen port |
| `homehub_manage_nginx` | `true` | write the site, remove the default site, reload Nginx |
| `homehub_client_max_body_size` | `1024m` | Nginx upload limit; keep it at least `max_upload_mb` |
| `homehub_fix_debian11_apt` | `true` on Debian 11 | rewrite apt sources to Yandex + archive |
| `homehub_tailwind_version` | `v3.4.19` | standalone Tailwind version |
| `homehub_config_force` | `false` | overwrite an existing `config.yml` (back it up first) |
| `homehub_language` | `ru` | UI language written into `config.yml` when non-empty |
| `homehub_install_build_tools` | `false` | install compilers and headers for source builds |
| `homehub_timezone` | empty | set the system timezone when non-empty |

### Run it

On a **brand-new host**, run the playbook directly the first time. `--check` cannot fully
simulate the first deploy because the skipped `git` clone would create files that later
tasks read. After the first apply, `--check --diff` is a safe preview.

```bash
cd deploy/ansible

# Apply (add -K if sudo needs a password)
ansible-playbook -i inventory.yml site.yml

# Preview later changes
ansible-playbook -i inventory.yml site.yml --check --diff
```

Use tags to run one part:

```bash
ansible-playbook -i inventory.yml site.yml --tags packages   # apt sources + system packages
ansible-playbook -i inventory.yml site.yml --tags app        # checkout, venv, dependencies, config
ansible-playbook -i inventory.yml site.yml --tags build      # Tailwind + translations
ansible-playbook -i inventory.yml site.yml --tags service    # systemd unit
ansible-playbook -i inventory.yml site.yml --tags nginx      # Nginx site
ansible-playbook -i inventory.yml site.yml --tags verify     # /healthz checks
```

`preflight.yml` and `user.yml` are tagged `always`, so validation runs even with a tag.

### What the playbook does

1. Verifies the target is Debian 11 or newer and 64-bit (`aarch64` or `x86_64`).
2. On Debian 11, rewrites `/etc/apt/sources.list` so every active Debian repository appears
   first on `mirror.yandex.ru` and then as an `archive.debian.org` duplicate. Comments and
   third-party repositories such as `archive.raspberrypi.org` are preserved. It also writes
   `/etc/apt/apt.conf.d/99homehub-archive` with `Acquire::Check-Valid-Until "false";` and
   refreshes the apt cache.
3. Installs `ca-certificates`, `curl`, `git`, `tzdata`, `ffmpeg`, `ghostscript`, and
   `nginx`.
4. Installs `uv` for the service user, installs Python 3.12, creates `.venv`, and runs
   `uv pip install -r requirements.txt`. It deliberately uses `install -r`, not `pip sync`,
   because `requirements.txt` lists direct packages only and `sync` would omit transitive
   dependencies such as `markupsafe`.
5. Downloads `tailwindcss-linux-x64` or `tailwindcss-linux-arm64` and rebuilds
   `static/output.css` only when its inputs change, syncs local `translations/*/LC_MESSAGES/messages.po`
   catalogs to the target host, and compiles the translation catalogs
   (`flask translations compile`) whenever `.po` files change or any `.mo` file is missing.
6. Checks out `homehub_version`, keeps an existing `config.yml` (copying `config-example.yml`
   when missing), and sets `language` in `config.yml` when `homehub_language` is configured.
7. Creates and owns `data/`, `uploads/`, `media/`, and `pdfs/`.
8. Writes the systemd unit with `Environment=HOME=...` and
   `Environment=PATH=.../.venv/bin:...`, then enables and starts the service.
9. Writes the Nginx site, removes the default site, validates the configuration, and
   reloads Nginx.
10. Waits for `/healthz` on gunicorn and through Nginx.

### Architecture handling

| `ansible_architecture` | Tailwind binary | Result |
| --- | --- | --- |
| `x86_64` | `tailwindcss-linux-x64` | supported |
| `aarch64` | `tailwindcss-linux-arm64` | supported |
| `armv7l` | — | fails with a 64-bit OS message |

`uv`, Python, pip wheels, and apt packages resolve the target architecture automatically.
The only architecture-specific download is the standalone Tailwind binary.

### Idempotency and preserved data

- Packages and services use declarative modules; the git checkout is fixed at
  `homehub_version`.
- Checksum markers in `~/.local/state/homehub/` prevent unnecessary dependency installs,
  stylesheet rebuilds, translation recompiles, and Tailwind downloads.
- Template changes notify `systemd daemon-reload`/restart and `nginx -t`/reload only when a
  file actually changes.
- `config.yml`, `data/`, `uploads/`, `media/`, and `pdfs/` are never overwritten.
- A second run on the Debian 11.11 `x86_64` test server finished with `changed=0` and
  `failed=0`.

### Updating HomeHub

1. Set `homehub_version` in `inventory.yml` to the tag or commit you want (for example
   `v0.4.1`), or leave it on `main` to follow the branch.
2. Run the playbook again. It updates the checkout, reinstalls dependencies only if
   `requirements.txt` changed, rebuilds CSS/translations only if their inputs changed, and
   restarts the service once.
3. Keep the existing `config.yml`, or set `homehub_config_force: true` to replace it. Set
   `homehub_config_src` to a local file to seed it from your machine.

### Manual steps that remain

- The admin password is interactive only:
  ```bash
  sudo -u <user> ~/homehub/.venv/bin/flask set-admin-password
  ```
- HTTPS is still out of scope; the generated Nginx site is HTTP only.
- The playbook does not create backups. Use the commands in step 12 before upgrading.

---

## Verification performed

The native steps in this guide were verified in October 2026 with a **Debian 11.11 arm64
container** running under QEMU on an amd64 Docker host. Docker was used only as a test bed;
it is not part of the runtime. This proves Debian 11 arm64 compatibility, not Pi 3 speed.

The Media Downloader path was additionally verified on a native **Debian 11.11 amd64**
server, using the exact arguments the app passes to `yt-dlp`. That confirms the download and
conversion steps, but not Pi 3 speed.

The Ansible playbook was verified on a fresh **Debian 11.11 amd64** server and on an
existing Debian 11.11 amd64 deployment. The fresh host was provisioned from scratch by the
playbook; the existing host was updated in place. A second run on the fresh host changed
nothing.

| Check | Result |
| --- | --- |
| Debian 11.11 `aarch64`, apt via `archive.debian.org` (manual guide) | pass |
| `uv` 0.13.5 + Python 3.12.15 virtualenv | pass |
| `uv pip install -r requirements.txt` (SQLAlchemy 2.1.5, Pillow 13.0.0) | pass |
| Tailwind v3.4.19 `linux-arm64` standalone build | pass, 44 KB `output.css` |
| `flask translations compile` | pass |
| gunicorn 26.2.0, `GET /healthz` | `200 {"ok":true}` |
| nginx 1.18 reverse proxy, `/healthz`, `/`, and `/static/output.css` via port 80 | `200` |
| `python run.py` on port 5001 | `200 {"ok":true}` |
| `yt-dlp` found through systemd's default `PATH` | fail—not found (fixed by the `PATH` line in step 9) |
| Media Downloader MP4 download + merge, Rutube, venv on `PATH` | pass, ~101 MB |
| Media Downloader MP3 extraction, Rutube, venv on `PATH` | pass, ~5.8 MB |
| `flask set-admin-password` and `--clear` | pass |
| Ansible `--syntax-check` (ansible-core 2.18.1) | pass |
| Ansible `--check --diff` on Debian 11.11 amd64 | pass |
| Ansible first apply on a fresh Debian 11.11 amd64 host | 60 ok, 26 changed, 0 failed |
| Ansible apt-source update (Yandex + archive) on the same host | 50 ok, 2 changed, 0 failed |
| Ansible second apply after the update (idempotency) | 49 ok, 0 changed, 0 failed |
| Ansible apt sources: `mirror.yandex.ru` first, `archive.debian.org` fallback | `apt-get update` pass |
| Ansible deployment, `/healthz` through Nginx | `200 {"ok":true}` |
| Actual Pi 3 hardware/performance | not tested |

The Nginx configuration was tested with nginx 1.18, the same series Debian 11 packages,
using the app behind it. The Debian 11 arm64 container run covers the native Python steps.
