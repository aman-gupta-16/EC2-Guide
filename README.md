# EC2-Guide



## 1️⃣ Setup EC2

* Launch **Ubuntu t3.micro**
* Open ports in **Security Group**:

  * `22` (SSH)
  * `80` (HTTP)
  * `443` (HTTPS if needed)

Update server:

```bash
sudo apt update && sudo apt upgrade -y
```

Install Node.js 22, NPM, Git, NGINX, PM2:

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs git nginx
sudo npm install -g pm2
```

---

## 2️⃣ Clone Repos from GitHub

### Setup SSH (fix for *Invalid username or token* error)

```bash
ssh-keygen -t ed25519 -C "your-email@example.com"
cat ~/.ssh/id_ed25519.pub
```

* Add key to **GitHub → Settings → SSH keys**.
* Test:

```bash
ssh -T git@github.com
```

✅ You should see: `Hi username! You've successfully authenticated`.

Clone backend + frontend:

```bash
mkdir ~/apps && cd ~/apps
mkdir backend frontend
cd backend
git clone git@github.com:username/backend-repo.git
cd ~/apps/frontend
git clone git@github.com:username/frontend-repo.git
```

---

## 3️⃣ Backend (Node/Express)

1. Copy `.env` manually (not pushed to GitHub).
2. Install + run with PM2:

```bash
cd ~/apps/backend
npm install
pm2 start npm --name backend -- start   # or: pm2 start index.js --name backend
pm2 save
```

---

## 4️⃣ Frontend (Next.js CSR)

> ⚠️ Building on EC2 can freeze (`t3.micro` low memory).
> Solution: **Build locally, push `.next`, `package.json`, `public` to GitHub, then pull on EC2.**

### Local machine:

```bash
npm install
npm run build
git add .next package.json package-lock.json public
git commit -m "Add production build"
git push
```

### On EC2:

```bash
cd ~/apps/frontend
git pull
npm install --production
pm2 start npm --name frontend -- start # or :pm2 start "npm start" --name forgeApi (before this remove output: 'export' from next.config.js this file)
pm2 save
```

---

## 5️⃣ Configure NGINX

Edit config:

```bash
sudo nano /etc/nginx/sites-available/default
```

Paste:

```nginx
server {
    listen 80;
    server_name 65.2.31.13;   # replace with Public IP or domain
    
#backend
    location /api {
        proxy_pass http://localhost:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }


#frontend
    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

Test & restart:

```bash
sudo nginx -t
sudo systemctl restart nginx
```

---

## 6️⃣ Test

* Frontend → `http://<ec2 public IP>/`
* Backend → `http://<ec2 public IP>/api/...`

---

## ⚡ Common Errors & Fixes

* **GitHub auth error** → Use SSH key, not HTTPS.
* **`.env` missing** → Manually copy `.env` file (never push to GitHub).
* **Build freezes on EC2** → Build locally, push `.next`, then pull on server.
* **App stops after SSH exit** → Use **PM2** (`pm2 start`, `pm2 save`, `pm2 startup`).
* **Nginx not working** → Run `sudo nginx -t` to debug, then `sudo systemctl restart nginx`.

---

✅ Done! Your MERN stack is live on a single EC2 instance.

---


Note:

## 📌 Elastic IP

* By default, your EC2 public IP changes every time you stop/start the instance.
* To make your app always reachable at the same IP, assign an **Elastic IP**:

  1. Go to **AWS EC2 → Elastic IPs**.
  2. Allocate a new Elastic IP.
  3. Associate it with your EC2 instance.
* Now your server has a **permanent public IP**.

---

## 📌 Security Group

* Security Groups control inbound/outbound traffic for your EC2.
* To make your Next.js app accessible:

  1. Open your instance’s **Security Group** in AWS console.
  2. Add an **Inbound Rule**:

     * Type: **HTTP**
     * Protocol: **TCP**
     * Port: **80** (if using Nginx) or **3000** (if exposing app directly)
     * Source: **0.0.0.0/0** (or restrict to your IP for security).
* This ensures the world can reach your app on the right port.

