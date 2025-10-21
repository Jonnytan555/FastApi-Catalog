# EC2 Deploy Pack (GitHub Actions)

This pack wires your GitHub repo to auto-deploy to an AWS EC2 instance


## 1) Add these files to your repo
- `.github/workflows/deploy-ec2.yml`
- `server/systemd/myapi.service.example`
- `server/nginx/myapi.nginx.example`
- `.env.example`

## 2) Create GitHub Secrets
- `EC2_HOST` — e.g. `ec2-3-91-xx-xx.compute-1.amazonaws.com`
- `EC2_USER` — `ubuntu` (Ubuntu) or `ec2-user` (Amazon Linux)
- `EC2_SSH_KEY` — contents of your private PEM
- `EC2_APP_DIR` — e.g. `/opt/myapi`
- `APP_SERVICE_NAME` — e.g. `myapi`

## 3) One-time prep on EC2
```bash
sudo mkdir -p /opt/myapi/releases
sudo chown -R $USER:$USER /opt/myapi
sudo apt-get update && sudo apt-get install -y python3-venv python3-pip
```

Create systemd unit from template (edit paths/module):
```bash
sudo tee /etc/systemd/system/myapi.service >/dev/null <<'UNIT'
# paste contents of server/systemd/myapi.service.example here (edited)
UNIT
sudo systemctl daemon-reload
sudo systemctl enable myapi
sudo systemctl start myapi
sudo systemctl status myapi --no-pager -l
```

(Optional) Nginx reverse proxy:
```bash
sudo apt-get install -y nginx
sudo tee /etc/nginx/sites-available/myapi >/dev/null <<'NGINX'
# paste contents of server/nginx/myapi.nginx.example here (edited)
NGINX
sudo ln -sf /etc/nginx/sites-available/myapi /etc/nginx/sites-enabled/myapi
sudo nginx -t && sudo systemctl reload nginx
```

## 4) Workflow flow
- Rsync -> `/opt/myapi/releases/<sha>`
- venv + pip install
- copy `/opt/myapi/.env` into release (if present)
- `current` symlink switch
- restart systemd service

## 5) Entry point
Default unit uses: `uvicorn app.main:app`. Change to your module path (e.g. `src.api.main:app`).

## 6) Rollback
```bash
cd /opt/myapi
ls -al releases
ln -sfn /opt/myapi/releases/<old-sha> /opt/myapi/current
sudo systemctl restart myapi
```

## 7) Troubleshooting
- `journalctl -u myapi -n 200 --no-pager -l`
- ensure security groups allow :80 (Nginx)
- ensure `.env` exists on server at `/opt/myapi/.env` or committed (not recommended)
