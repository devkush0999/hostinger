# Deployment Seekho: Zero se Production tak (NextDeal ke saath)

> Ye notes ek senior ki tarah likhe hain jo tumhare saath baith ke samjha raha hai. Har chapter mein teen cheezein hain: **concept**, **NextDeal mein ye kahan hai**, aur **homework**. Jahan `> Senior ki baat` likha hai, wo galtiyon se seekhi hui baat hai, kisi kitaab se nahi.
>
> Ek rule yaad rakhna: **production server pe koi bhi command chalane se pehle soch lo ki wo kya badlegi.** Shuru ke chapters mein hum sirf *dekhenge*, kuch change nahi karenge.

---

## Index

0. [Mindset: deployment asal mein hai kya](#0-mindset-deployment-asal-mein-hai-kya)
1. [Server aur VPS](#1-server-aur-vps)
2. [Linux essentials](#2-linux-essentials)
3. [SSH: server ka darwaza](#3-ssh-server-ka-darwaza)
4. [Networking: IP, port, DNS, firewall](#4-networking-ip-port-dns-firewall)
5. [Node app ko production mein chalana (build, env, PM2)](#5-node-app-ko-production-mein-chalana)
6. [Nginx reverse proxy + HTTPS](#6-nginx-reverse-proxy--https)
7. [Database aur Redis](#7-database-aur-redis)
8. [Security hardening](#8-security-hardening)
9. [Deploy process: release, reload, rollback](#9-deploy-process-release-reload-rollback)
10. [Logs, monitoring, alerts](#10-logs-monitoring-alerts)
11. [Docker: kab aur kyun](#11-docker-kab-aur-kyun)
12. [CI/CD: GitHub Actions se auto deploy](#12-cicd-github-actions-se-auto-deploy)
13. [Advanced: scaling, workers, disaster recovery](#13-advanced-scaling-workers-disaster-recovery)
14. [Cheat sheet + glossary](#14-cheat-sheet--glossary)

---

## 0. Mindset: deployment asal mein hai kya

Code likhna aadha kaam hai. Baaki aadha ye hai ki wo code **kisi aur ke computer pe, 24x7, bina tumhare dekhe, safely** chale. Usi ko deployment aur operations kehte hain.

Production ke teen sawal hamesha poochho:

1. **Ye chal raha hai?** (uptime, health check)
2. **Ye sahi chal raha hai?** (errors, logs, latency)
3. **Ye toot gaya toh wapas kaise laaunga?** (backup, rollback)

Teesra sawal sabse zyada log bhool jaate hain, aur wahi sabse mehnga padta hai.

> **Senior ki baat:** Junior poochta hai "deploy kaise karu?". Senior poochta hai "agar deploy fail hua toh 2 minute mein purana version wapas kaise aayega?". Jis din tum rollback pehle sochne lagoge, us din tum senior ban rahe ho.

---

## 1. Server aur VPS

### Concept

**Server** bas ek computer hai jo network pe requests ka jawab deta hai. Usme koi jaadu nahi hai.

Hosting ke options, sabse "managed" se sabse "khud sambhalo" tak:

| Type | Example | Tum kya sambhalte ho |
|---|---|---|
| Serverless / PaaS | Vercel, Render, Heroku | Sirf code |
| Managed containers | AWS ECS, Cloud Run | Code + container |
| **VPS** | **Hostinger VPS**, DigitalOcean, EC2 | **OS, security, process, DB, backups: sab kuch** |
| Dedicated / bare metal | Physical server | Hardware tak |

**VPS (Virtual Private Server):** Hostinger ki ek badi physical machine pe *hypervisor* (KVM) chalta hai jo usse kai virtual machines mein baant deta hai. Ek VM tumhari hai, jisme fixed vCPU, RAM aur disk milti hai, plus root access.

**Fayda:** sasta hai, poora control hai, aur jo chahe install kar sakte ho.
**Nuksaan:** sab tumhari zimmedari hai. Server hack hua, disk bhar gayi, ya process mar gaya, toh Hostinger nahi bachayega.

### NextDeal mein

- API ek Hostinger VPS pe PM2 ke andar chalti hai (`ecosystem.config.cjs`, port 4000).
- DB ka host `82.112.231.48` hai. **Ye confirm karna hai** ki ye wahi VPS hai ya alag.
- Pehle ek AWS EC2 (`13.203.227.88`) bhi use hua tha, ye purane Google OAuth redirect URL mein dikhta hai.

### Homework 1

hPanel mein jaake note karo: VPS plan ka naam, kitne vCPU, kitni RAM, kitni disk, OS kaunsa hai, aur VPS ka IP kya hai.

---

## 2. Linux essentials

Production servers 95% Linux hote hain. Isliye ye chapter sabse important hai, isko jaldi mat padhna.

### 2.1 Filesystem: sab kuch file hai

```
/            root (sabka baap)
├── etc/     configs        → /etc/nginx/, /etc/ssh/sshd_config
├── var/     badalta data   → /var/log/ (logs), /var/www/ (websites)
├── home/    users ke ghar  → /home/deploy/
├── root/    root user ka ghar
├── usr/     installed programs
├── tmp/     temporary (reboot pe saaf ho sakta hai)
└── opt/     third-party apps
```

Commands:

```sh
pwd                 # main kahan hoon
ls -la              # list, hidden files bhi (-a), details ke saath (-l)
cd /var/log         # folder badlo
cd ~                # apne home pe jao
cat file.txt        # poori file dikhao
less file.txt       # badi file scroll karke padho (q se bahar)
head -20 file       # pehli 20 lines
tail -f file.log    # file ke end ko live follow karo (logs ke liye best)
nano file.txt       # simple editor (Ctrl+O save, Ctrl+X exit)
cp a b; mv a b; rm a
mkdir -p a/b/c      # nested folders ek saath
```

> **Senior ki baat:** `rm -rf` chalane se pehle hamesha `pwd` aur `ls` karo. Ek galat space (`rm -rf / var/app`) poora server uda deta hai. Production pe main `rm` se pehle `ls` mein wahi path chala ke dekhta hoon.

### 2.2 Users aur permissions

Linux multi-user hai. **root** bhagwaan hai, sab kar sakta hai, isliye usse roz kaam karna khatarnak hai.

```sh
whoami
sudo <command>      # ek command root ban ke chalao
```

`ls -l` ka output aisa dikhta hai:

```
-rw-r--r-- 1 deploy deploy 1234 Oct 9 .env
│└┬┘└┬┘└┬┘   └─owner └─group
│ │  │  └── others: read
│ │  └───── group: read
│ └──────── owner: read+write
└────────── '-' = file, 'd' = directory
```

Numbers mein: r=4, w=2, x=1.

```sh
chmod 600 .env          # sirf owner padh/likh sake (secrets ke liye ZAROORI)
chmod 755 script.sh     # owner sab, baaki read+execute
chown deploy:deploy file
```

> **Senior ki baat:** `.env` mein Razorpay aur AWS ke secrets hain. Uski permission `600` honi chahiye aur owner app wala user. `chmod 777` kabhi mat karna. Wo "permission error theek karo" ka shortcut nahi hai, wo "sabko darwaza khol do" hai.

### 2.3 Processes

Jo bhi program chal raha hai, wo ek process hai jiska ek PID (number) hota hai.

```sh
ps aux | grep node      # node processes dhundo
top                     # live CPU/RAM (q se exit)
htop                    # top ka sundar version (apt install htop)
kill <PID>              # politely band karo (SIGTERM)
kill -9 <PID>           # zabardasti maaro (last resort)
```

**Signals:** `SIGTERM` ka matlab hai "please band ho jao, kaam samet lo". `SIGKILL` (-9) ka matlab hai "abhi mar jao". NextDeal ke `server.ts` mein `SIGTERM` handler hai jo DB aur Redis band karke exit karta hai. Isi ko *graceful shutdown* kehte hain.

### 2.4 Packages

Ubuntu pe `apt` use hota hai:

```sh
sudo apt update             # list refresh
sudo apt upgrade            # sab update
sudo apt install nginx
```

### 2.5 Services (systemd)

Jo cheez boot pe khud start honi chahiye (nginx, postgres, redis), wo **systemd service** hoti hai:

```sh
systemctl status nginx
sudo systemctl restart nginx
sudo systemctl enable nginx     # boot pe auto start
journalctl -u nginx -n 50       # us service ke last 50 logs
```

### 2.6 Disk aur memory

```sh
df -h           # disk kitni bhari hai
du -sh *        # kaunsa folder kitna bada hai
free -h         # RAM
```

> **Senior ki baat:** Production down hone ke top 3 boring kaaran: **disk full** (logs ne bhar di), **RAM khatam** (OOM killer ne process maar diya), aur **SSL certificate expire**. Fancy bugs baad mein aate hain. Jab bhi server ajeeb behave kare, pehle `df -h` aur `free -h` dekho.

### Homework 2

Server pe chalao aur output samjho (mujhe bhejo, secrets hata ke):

```sh
cat /etc/os-release | head -3
free -h; df -h /; nproc
ps aux --sort=-%mem | head -10      # sabse zyada RAM khaane wale
ls -la ~ /var/www 2>/dev/null
```

---

## 3. SSH: server ka darwaza

### Concept

**SSH** (Secure Shell) ek encrypted connection hai jisse tum server ka terminal apne laptop pe chalate ho. Port 22 default hai.

```sh
ssh root@<VPS_IP>
```

Login ke do tareeke hain:

1. **Password:** kamzor hai. Bots din-raat password guess karte rehte hain.
2. **SSH key:** strong hai. Tumhare laptop pe ek *private key* hoti hai (kabhi share nahi karte) aur server pe *public key* (jo share kar sakte ho).

Key kaise kaam karti hai: server ek challenge bhejta hai, tumhari private key usse sign karti hai, aur server public key se check kar leta hai. Password network pe jaata hi nahi.

### Key setup

Apne Mac pe:

```sh
ssh-keygen -t ed25519 -C "devesh-laptop"     # ~/.ssh/id_ed25519 (private) + .pub
ssh-copy-id root@<VPS_IP>                    # public key server pe daal do
ssh root@<VPS_IP>                            # ab password nahi maangega
```

Shortcut ke liye `~/.ssh/config` mein ye daalo:

```
Host nextdeal
    HostName <VPS_IP>
    User deploy
    IdentityFile ~/.ssh/id_ed25519
```

Ab bas `ssh nextdeal` likhna kaafi hai.

### Hardening (Chapter 8 mein detail se)

`/etc/ssh/sshd_config` mein:

```
PermitRootLogin no
PasswordAuthentication no
```

> **Senior ki baat:** SSH settings badalne se pehle **ek dusra terminal khula rakho** jisme already login ho. Phir `sudo systemctl reload ssh` karo aur *naye* terminal se login test karo. Agar galti hui toh purane terminal se theek kar loge. Warna khud ko server se bahar band kar doge, aur phir sirf Hostinger ka browser console bachega.

### Homework 3

SSH key banao, server pe daalo, aur `~/.ssh/config` set karo. Abhi password login band **mat** karna.

---

## 4. Networking: IP, port, DNS, firewall

### 4.1 IP aur port

- **IP:** building ka address (VPS ka public IP).
- **Port:** building ke andar flat number. Ek IP pe 65,535 ports hote hain.

| Port | Kaun |
|---|---|
| 22 | SSH |
| 80 | HTTP |
| 443 | HTTPS |
| 4000 | NextDeal Node app (PM2 config) |
| 5432 | PostgreSQL |
| 3306 | MySQL |
| 6379 | Redis |

### 4.2 `localhost` vs `0.0.0.0`

Ye beginner ka sabse bada confusion hai, aur security ka bhi.

- `127.0.0.1` / `localhost`: sirf **isi machine** ke andar se pahunch sakte ho.
- `0.0.0.0`: **har network interface** pe suno, yaani internet se bhi.

NextDeal `src/index.ts` mein `httpServer.listen(env.PORT, '0.0.0.0')` karta hai. Iska matlab hai ki agar firewall nahi hai toh `http://<VPS_IP>:4000` seedha internet pe khula hai, Nginx aur SSL ko bypass karke.

> **Senior ki baat:** Database (5432) aur Redis (6379) **kabhi bhi internet pe khule nahi hone chahiye.** Internet pe bots har second poore IP range scan karte hain. Khula Redis bina password ke matlab poora server compromise. Ye theory nahi hai, maine ye hote dekha hai.

Kaun kis port pe sun raha hai, ye dekho:

```sh
sudo ss -tlnp
# 127.0.0.1:6379   → sirf andar se (achha)
# 0.0.0.0:5432     → internet se bhi (KHATRA, agar firewall nahi hai)
```

### 4.3 DNS

DNS internet ki phonebook hai: `nextopson.com` → `A record` → IP.

```sh
dig +short nextopson.com
dig +short nextdeal.in
nslookup nextopson.com
```

Record types:

- **A:** domain → IPv4
- **CNAME:** domain → dusra domain
- **TXT:** verification (Google, email SPF/DKIM)
- **MX:** email server

**TTL** batata hai ki record kitni der cache rahega. Server migrate karne se ek din pehle TTL kam (300s) kar do, taaki switch jaldi propagate ho.

### 4.4 HTTP vs HTTPS

HTTP plain text hai: beech mein koi bhi OTP ya token padh sakta hai. HTTPS = HTTP + TLS encryption, jiske liye certificate chahiye. Let's Encrypt free certificate deta hai (Chapter 6).

### 4.5 Firewall (`ufw`)

```sh
sudo ufw status
sudo ufw allow OpenSSH        # PEHLE ye, warna khud lock ho jaoge
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

Iske baad 4000, 5432 aur 6379 bahar se band ho jaate hain, aur andar se sab chalta rehta hai.

Hostinger hPanel mein bhi ek alag firewall hota hai. Dono layers samajh lo.

### Homework 4

```sh
sudo ss -tlnp
sudo ufw status verbose
dig +short nextopson.com
```

Ye batao: kaunse ports `0.0.0.0` pe khule hain, aur domain kaunse IP pe point karta hai. Kya wo VPS ka hi IP hai?

---

## 5. Node app ko production mein chalana

### 5.1 Dev vs production

| | Dev | Production |
|---|---|---|
| Run | `tsx src/index.ts` (TS seedha) | Build kiya hua JS (`dist/index.cjs`) |
| Restart | nodemon (file change pe) | PM2 (crash pe) |
| Logs | Terminal | Files + rotation |
| Errors | Detail dikhao | User ko generic message, detail logs mein |

### 5.2 Build: NextDeal mein kya hota hai

```sh
npm run build-prod      # = npm i && rimraf dist && node esbuild.config.js
```

`esbuild` saara TypeScript code **ek file** `dist/index.cjs` mein bundle kar deta hai. Sirf `sharp`, `tesseract.js` aur `bcrypt` bahar rehte hain, kyunki unme native binaries hoti hain.

**Ek chhupa hua jaal (iss project mein real hai):** `esbuild.config.js` mein ye hai:

```js
define: { 'process.env.NODE_ENV': JSON.stringify(process.env.NODE_ENV || 'development') }
```

Matlab build ke **waqt** jo `NODE_ENV` tha, wahi code mein hardcode ho jaata hai. `build-prod` script khud `NODE_ENV` set nahi karta. Toh agar server pe bina `NODE_ENV=production` ke build kiya, toh bundle ke andar har `process.env.NODE_ENV === 'production'` wala check hamesha `false` ho jaayega, chahe PM2 runtime pe kuch bhi set kare. Aur minify bhi band rahega.

Sahi tareeka:

```sh
NODE_ENV=production npm run build-prod
```

> **Senior ki baat:** "Mere laptop pe chalta hai" wali bugs aksar build-time vs run-time config ke farak se aati hain. Har config ke liye poochho: **ye value kab padhi jaati hai, build pe ya start pe?**

### 5.3 Environment variables (`.env`)

Config code mein nahi, environment mein rehta hai (12-factor app ka principle). `-r dotenv/config` start pe `.env` padh leta hai.

**Priority ka niyam:** `dotenv` by default *pehle se set* variables ko override nahi karta. Isliye NextDeal mein:

- `.env` mein `PORT=4004` hai
- PM2 `ecosystem.config.cjs` mein `PORT: 4000` set karta hai
- **Jeetta 4000 hai**, kyunki PM2 ne process start hone se pehle hi set kar diya tha.

Isi tarah `env_production` mein `REDIS_HOST` etc. bhi `.env` pe bhaari padte hain. Ye samajh lo, warna debugging mein ghante jaayenge.

`envConfig.ts` mein `envalid` start pe check karta hai ki zaroori variables (jaise `SMTP_*`) hain ya nahi. Na hon toh app start hi nahi hoga. Ye achhi practice hai: **fail fast**.

### 5.4 PM2: process manager

Node process crash ho gaya toh kaun uthayega? Server reboot hua toh kaun start karega? PM2.

```sh
pm2 status                  # kya chal raha hai
pm2 logs nextdeal-dev       # live logs
pm2 logs --lines 200
pm2 describe nextdeal-dev   # poori detail: path, env, restarts
pm2 monit                   # live CPU/RAM
pm2 restart nextdeal-dev    # band karke chalu (downtime hota hai)
pm2 reload nextdeal-dev     # cluster mode mein zero-downtime
pm2 save                    # current list yaad rakho
pm2 startup                 # boot pe PM2 auto-start (ek baar setup)
```

NextDeal ka config (`ecosystem.config.cjs`) padhte hain:

```js
instances: 1, exec_mode: "fork"   // ek hi process
max_memory_restart: "800M"        // 800MB RAM cross hone pe restart
max_restarts: 20, min_uptime: "30s"
node_args: ["--max-old-space-size=1024"]
```

Dhyaan dene wali baatein:

- Production app ka naam bhi `"nextdeal-dev"` hai. Confusing hai, par kaam karta hai.
- `fork` + 1 instance ka matlab hai **ek hi CPU core** use ho raha hai, aur restart pe kuch second downtime hota hai.
- PM2 ka `max_memory_restart` (800M) **aur** code ka apna memory check (`index.ts`, 1GB heap) dono hain. Do jagah ek hi kaam karne se behaviour unpredictable hota hai. Ek hi system rakho, aur woh PM2 ho.

**Cluster mode kyun abhi nahi?** `instances: "max"` karoge toh har process apne cron jobs chalayega, yaani reminder emails 4 baar jaayengi. Socket.IO bhi toot jaayega, kyunki user ek process mein connected hai aur notification dusre process se jaati hai. Pehle Chapter 13 wala fix karna padega.

### Homework 5

```sh
pm2 status
pm2 describe <app-name>      # script path, cwd, restarts count, uptime
pm2 logs <app-name> --lines 100 --nostream
```

Batao: kitne restarts hue hain, uptime kitna hai, aur code kis folder se chal raha hai.

---

## 6. Nginx reverse proxy + HTTPS

### Concept

Node ko seedha internet pe kyun nahi khulte? Kyunki **Nginx** in kaamon mein Node se behtar hai:

- SSL/TLS handle karna
- Slow clients ko jhelna (Node ko free rakhna)
- Gzip, static files, rate limiting, request size limit
- Ek server pe kai apps/domains chalana

```
Internet :443 → Nginx → 127.0.0.1:4000 (Node)
```

Isko **reverse proxy** kehte hain.

### Config (Socket.IO ke saath)

`/etc/nginx/sites-available/nextdeal`:

```nginx
server {
    server_name api.nextopson.com;          # apna domain

    client_max_body_size 50m;               # image upload; app bhi 50mb leta hai

    location / {
        proxy_pass http://127.0.0.1:4000;
        proxy_http_version 1.1;

        # WebSocket (Socket.IO) ke liye ZAROORI
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Asli client info Node tak pahunchao
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_read_timeout 300s;
    }

    listen 80;
}
```

```sh
sudo ln -s /etc/nginx/sites-available/nextdeal /etc/nginx/sites-enabled/
sudo nginx -t                  # config test (HAMESHA reload se pehle)
sudo systemctl reload nginx
```

### HTTPS (Let's Encrypt)

```sh
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d api.nextopson.com
sudo certbot renew --dry-run   # auto-renew test
```

Certbot config mein 443 block khud jod deta hai aur 80 ko 443 pe redirect kar deta hai. Certificate 90 din chalta hai, aur renewal ek timer automatically karta hai. Phir bhi `--dry-run` check kar lo.

### `trust proxy`

Nginx ke peeche Node ko har request `127.0.0.1` se aati dikhti hai. Asli IP `X-Forwarded-For` header mein hoti hai. NextDeal `app.set('trust proxy', true)` karta hai, matlab *koi bhi* client fake `X-Forwarded-For` bhej ke apna IP badal sakta hai aur rate limit bypass kar sakta hai. Nginx ke peeche sahi setting ye hai:

```js
app.set('trust proxy', 'loopback');   // sirf local Nginx pe bharosa
```

> **Senior ki baat:** `nginx -t` ke bina reload karna waisa hi hai jaise compile kiye bina production pe push karna. Ek typo aur saari sites down.

### Homework 6

```sh
systemctl status nginx --no-pager
ls /etc/nginx/sites-enabled/
sudo nginx -T 2>/dev/null | grep -E "server_name|proxy_pass|listen"
sudo certbot certificates
```

Isse pata chalega ki abhi Nginx hai ya nahi, aur certificate kab expire hoga.

---

## 7. Database aur Redis

### 7.1 PostgreSQL basics

NextDeal ab Postgres pe hai (`src/database/dataSource.ts`, TypeORM). Kuch cheezein samjho:

- **Connection pool:** `extra.max: 20`, yaani app max 20 connections kholega. PM2 cluster mein 4 process = 80 connections. Postgres ka default `max_connections` 100 hai, toh dhyaan rakhna.
- **`statement_timeout: 30000`:** 30 second se lambi query apne aap kill ho jaayegi. Achhi safety hai.
- **`synchronize: false`:** TypeORM khud schema nahi badlega. **Ye hamesha false rehna chahiye production mein.** Schema migrations se badalta hai (`src/database/migrations/`).

```sh
sudo -u postgres psql
\l                 -- databases
\c nextdeal        -- connect
\dt                -- tables
\d "UserAuth"      -- table structure
SELECT count(*) FROM "Property";
\q
```

### 7.2 Backups: sabse important section

**3-2-1 rule:** data ki 3 copies, 2 alag media pe, aur 1 off-site (server ke bahar).

```sh
# Backup (custom format: compress hota hai, selective restore ho sakta hai)
pg_dump -Fc -h localhost -U <user> nextdeal > nextdeal_$(date +%F).dump

# Restore (NAYE database mein, kabhi production pe seedha nahi)
createdb nextdeal_restore_test
pg_restore -d nextdeal_restore_test nextdeal_2026-10-09.dump
```

Roz cron se backup lo aur S3 pe bhejo (bucket `nextdeal-app-backup` already hai). Hostinger ka VPS snapshot bhi lo, lekin sirf usi pe depend mat raho.

> **Senior ki baat:** **Jo backup kabhi restore karke test nahi hua, wo backup nahi hai, wo ek ummeed hai.** Har mahine ek baar restore test karo. Maine companies dekhi hain jinke 2 saal ke backups khaali files nikle.

### 7.3 Migrations

Schema change ka proper tareeka:

1. Migration file likho (up + down)
2. Staging/local pe chalao
3. Production ka **backup lo**
4. Production pe migration chalao
5. Phir naya code deploy karo

Purana code naye schema ke saath bhi chalna chahiye (backward compatible). Column delete karna ho toh do deploy mein karo: pehle code se usage hatao, phir column.

### 7.4 Redis

NextDeal mein Redis do kaam karta hai: **Bull queues** (image processing, email, notifications) aur **cache** (location/geo).

```sh
redis-cli ping            # PONG
redis-cli info memory
redis-cli --scan --pattern 'bull:*' | head
```

Security: `/etc/redis/redis.conf` mein `bind 127.0.0.1` aur `requirepass` set karo. PM2 config mein `REDIS_PASSWORD: ""` hai, toh agar Redis kabhi bahar khula toh koi bhi andar ghus sakta hai.

Memory: Redis RAM mein rehta hai. `maxmemory` set nahi kiya toh Redis aur Node dono RAM ke liye ladenge.

### Homework 7

```sh
sudo ss -tlnp | grep -E '5432|3306|6379'
ls -la /var/backups 2>/dev/null; crontab -l; sudo crontab -l
```

Batao: DB isi VPS pe hai ya `82.112.231.48` alag machine hai? Aur abhi koi automatic backup chal raha hai ya nahi?

---

## 8. Security hardening

Do levels pe security hoti hai: **server** aur **app**. Dono zaroori hain.

### 8.1 Server checklist

```sh
# 1. Non-root user banao
adduser deploy
usermod -aG sudo deploy
# deploy user ke liye SSH key setup karo (Chapter 3)

# 2. SSH hardening (/etc/ssh/sshd_config)
#    PermitRootLogin no
#    PasswordAuthentication no
sudo systemctl reload ssh      # dusra terminal khula rakh ke!

# 3. Firewall (Chapter 4)
sudo ufw allow OpenSSH && sudo ufw allow 80,443/tcp && sudo ufw enable

# 4. Brute-force protection
sudo apt install fail2ban

# 5. Automatic security updates
sudo apt install unattended-upgrades
sudo dpkg-reconfigure unattended-upgrades
```

App bhi root se nahi chalni chahiye. Agar app hack hui toh attacker ko utni hi power milegi jitni app user ki hai.

### 8.2 Secrets

- `.env` ki permission `600` honi chahiye, aur wo git mein **kabhi** nahi jaana chahiye.
- Jo secret ek baar git mein chala gaya, wo **leaked** maano. File delete karne se kuch nahi hota, history mein rehta hai. **Rotate karo**, yaani naya key banao aur purana disable karo.
- NextDeal mein abhi rotate karne laayak: Firebase service-account JSON (`src/config/`), Twilio creds (`twilioSmsService.ts`), Google Maps key (`envConfig.ts`).

### 8.3 App-level issues (NextDeal ke real issues)

Ye `REVIEW.md` mein detail se hain. Deployment seekhte waqt yaad rakho ki **server kitna bhi secure ho, app ka khula darwaza sab bekaar kar deta hai.**

| Issue | Kyun khatarnak |
|---|---|
| Direct-buy premium bina payment verify | Free premium |
| Google login bina token verify | Kisi ka bhi account |
| Apple token signature check nahi | Kisi ka bhi account |
| Admin export bina auth | Users ke mobile numbers leak |
| Socket room koi bhi join kare | Doosre ki notifications |
| `CORS *` + credentials, `OTP_BYPASS_ENABLED=true` | Attack surface badhta hai |

> **Senior ki baat:** Security ek baar ka kaam nahi, aadat hai. Har naye route pe teen sawal poochho: kaun call kar sakta hai, kiska data dikh raha hai, aur client ki kis value pe main bharosa kar raha hoon.

---

## 9. Deploy process: release, reload, rollback

### 9.1 Manual deploy (seedha tareeka)

```sh
ssh nextdeal
cd /path/to/nextdealAppServer
git fetch && git log --oneline HEAD..origin/master   # kya naya aa raha hai, PEHLE dekho
git pull
npm ci                                # lockfile se exact versions
NODE_ENV=production npm run build-prod
pm2 reload ecosystem.config.cjs --env production
pm2 logs --lines 50                   # errors?
curl -s localhost:4000/health/deep    # sab connected?
```

Problem ye hai ki build fail hua toh `dist/` already delete ho chuka hota hai (`rimraf dist`), aur beech mein restart hua toh app hi nahi chalegi.

### 9.2 Release folders (pro tareeka)

```
/srv/nextdeal/
├── releases/
│   ├── 2026-10-08_1530/
│   └── 2026-10-09_1100/     ← naya
├── current -> releases/2026-10-09_1100   (symlink)
└── shared/.env               (har release isse link karti hai)
```

Steps:

1. Naye folder mein code laao, `npm ci` aur build karo. Purana version tab tak chal raha hai.
2. Build pass ho gaya? Toh `ln -sfn` se `current` symlink naye folder pe kar do.
3. `pm2 reload`
4. Health check fail hua? Symlink purane folder pe wapas karo aur reload. **Ye 10 second ka rollback hai.**

### 9.3 Zero downtime

`fork` mode + 1 instance ka matlab hai ki reload pe bhi thoda downtime hoga. Zero downtime ke liye do cheezein chahiye: cluster mode (2+ instances) **aur** app ka graceful shutdown sahi hona (abhi `index.ts` aur `server.ts` dono apne shutdown handlers lagate hain, ye theek karna padega).

### 9.4 Deploy ke rules

- **Friday shaam deploy mat karo.**
- Deploy ke baad 15 minute logs dekho.
- Har deploy ka note rakho: kya gaya, kab gaya, kisne kiya.
- DB migration aur code deploy alag-alag steps hain.

> **Senior ki baat:** Achha deploy boring hota hai. Agar deploy mein adrenaline aa raha hai toh process mein kuch galat hai.

---

## 10. Logs, monitoring, alerts

### 10.1 Logs

PM2 logs `./logs/` mein likhta hai (config mein `out_file`, `error_file`). **Logs ki rotation nahi hogi toh disk bhar jaayegi.**

```sh
pm2 install pm2-logrotate
pm2 set pm2-logrotate:max_size 50M
pm2 set pm2-logrotate:retain 14
```

Nginx logs `/var/log/nginx/access.log` aur `error.log` mein hain.

Kaam ki commands:

```sh
grep -i error logs/error.log | tail -50
grep " 500 " /var/log/nginx/access.log | wc -l
journalctl -u nginx --since "1 hour ago"
```

### 10.2 Health checks

NextDeal mein do endpoints hain:

- `/health`: process zinda hai? (hamesha 200 deta hai, DB down ho tab bhi)
- `/health/deep`: DB + Redis ka check

Problem: `/health/deep` Redis down hone pe bhi 200 deta hai. Monitoring ke liye ye 503 dena chahiye, warna alert kabhi bajega hi nahi.

### 10.3 Uptime monitoring (free se shuru karo)

UptimeRobot ya Better Stack ka free plan lo, `https://<domain>/health/deep` har 1-5 min pe hit karwao, aur alert email/Telegram pe mangwao.

Iske baad error tracking ke liye **Sentry** (exception stack traces) aur metrics ke liye Prometheus + Grafana, ya Hostinger ka built-in monitoring.

> **Senior ki baat:** Users se pehle tumhe pata chalna chahiye ki server down hai. Agar customer support se pata chal raha hai, toh monitoring nahi hai.

---

## 11. Docker: kab aur kyun

### Concept

Docker app ko uske **poore environment** ke saath ek *image* mein pack kar deta hai: Node version, OS libraries, code. Phir wo kahin bhi same chalti hai.

- **Image:** recipe (read-only)
- **Container:** chalti hui image
- **Compose:** kai containers (app + db + redis) ek file se

### Kab use karein

**Docker ke fayde:** "mere pe chalta hai" khatam, `sharp` / `tesseract` jaisi native deps ka jhanjhat khatam, rollback = purani image chalao, aur dev aur prod ek jaise.
**Kab zaroori nahi:** ek chhota VPS, ek app, aur team Docker nahi jaanti. Tab PM2 bhi theek hai.

### NextDeal ka haal

- `compose.local.yml` / `compose.postgres.local.yml`: tumhare local setup, achhe hain.
- `Dockerfile` (production stage): **toota hua hai.** Node 20 use karta hai, `dist/index.js` chalata hai jo build nahi hota (`.cjs` banta hai), aur `tsconfig-paths` register nahi karta. Iske upar `--only=production` se dev deps hat jaati hain.
- `docker-compose.prod.yml`: MySQL pe hai, jabki app ab Postgres pe hai.

Toh abhi production Docker pe **nahi** hai. Docker pe le jaana ek achha future step hai, lekin pehle Chapter 1-10 pakke karo.

---

## 12. CI/CD: GitHub Actions se auto deploy

- **CI (Continuous Integration):** har push pe tests + typecheck automatically chalein.
- **CD (Continuous Deployment):** tests pass hon toh server pe deploy automatically ho jaaye.

Pehla step, sirf CI (`.github/workflows/ci.yml`):

```yaml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 24, cache: npm }
      - run: npm ci
      - run: npx tsc --noEmit
      - run: npm test
```

Doosra step, deploy. GitHub runner SSH se server pe jaake Chapter 9 wala script chalata hai. SSH private key GitHub **Secrets** mein rehti hai, code mein nahi. Server pe ek alag `deploy` user ho jiski permissions limited hon.

> **Senior ki baat:** CD tab tak mat lagao jab tak manual deploy aur rollback 5 baar bina tension ke na kar lo. Automation ek toote process ko sirf tez se tod deta hai.

---

## 13. Advanced: scaling, workers, disaster recovery

### 13.1 API aur workers alag karo

Abhi ek hi Node process teen kaam kar raha hai: HTTP API, Socket.IO, aur cron + Bull workers (image processing, OCR, emails). Matlab heavy image processing API ko slow karti hai, aur API restart hone pe jobs beech mein ruk jaati hain.

Target ye hai:

```
pm2: nextdeal-api      (cluster, 2+ instances, ENABLE_BACKGROUND_JOBS=false)
pm2: nextdeal-worker   (fork, 1 instance, sirf Bull processors + cron)
```

Code mein `ENABLE_BACKGROUND_JOBS` flag already hai, toh ye kaafi kam effort ka kaam hai.

### 13.2 Socket.IO ko multiple instances pe chalana

Do cheezein chahiye:

1. **Redis adapter** (`@socket.io/redis-adapter`), taaki process A se bheja event process B ke user tak pahunche.
2. **Sticky sessions**, taaki ek client hamesha usi process pe jaaye (Nginx `ip_hash` ya PM2 ke saath alag setup).

### 13.3 Vertical vs horizontal scaling

- **Vertical:** bada VPS lo (zyada RAM/CPU). Simple hai, aur shuru mein yahi karo.
- **Horizontal:** kai servers + load balancer. Iske liye app *stateless* honi chahiye: sessions, sockets aur uploads kisi ek server ke disk pe nahi hone chahiye.

### 13.4 Managed services

Jab team chhoti ho, toh DB khud chalana sabse risky kaam hai. Managed Postgres (backups, failover, updates sab unka) aksar paise ke laayak hota hai. Images ke liye S3 + CDN (CloudFront) se server ka load kam hota hai.

### 13.5 Disaster recovery

Do numbers decide karo:

- **RPO (Recovery Point Objective):** kitna data khona chalega? (jaise 24 ghante → daily backup)
- **RTO (Recovery Time Objective):** kitni der down reh sakte ho? (jaise 2 ghante)

Phir ek **runbook** likho: "VPS mar gaya toh step 1, 2, 3...". Saal mein ek baar usse naye server pe practice karo.

### 13.6 Incident handling

1. **Ghabrao mat.** Pehle impact samjho: kitne users, kaunsa feature.
2. **Pehle restore karo, baad mein root cause dhundo.** Rollback ko embarrassment mat samjho.
3. Timeline note karte jao.
4. Baad mein *blameless postmortem* likho: kya hua, kyun hua, aage kaise rokein.

---

## 14. Cheat sheet + glossary

### Daily commands

```sh
# Health
pm2 status; pm2 monit
curl -s localhost:4000/health/deep
df -h; free -h; uptime

# Logs
pm2 logs <app> --lines 100
tail -f /var/log/nginx/error.log
journalctl -u nginx -n 50

# Network
sudo ss -tlnp
sudo ufw status
dig +short <domain>

# Nginx
sudo nginx -t && sudo systemctl reload nginx

# Deploy
git pull && npm ci && NODE_ENV=production npm run build-prod && pm2 reload ecosystem.config.cjs --env production
```

### Glossary

| Term | Matlab |
|---|---|
| VPS | Virtual machine jiska poora control tumhara hai |
| SSH | Encrypted remote terminal |
| Reverse proxy | Beech ka server (Nginx) jo requests app tak forward karta hai |
| TLS/SSL | HTTPS ka encryption |
| PM2 | Node process manager (restart, logs, cluster) |
| Fork vs cluster | Ek process vs kai process (multi-core) |
| Graceful shutdown | Kaam samet ke band hona |
| Migration | Versioned schema change |
| RPO / RTO | Kitna data / kitna time kho sakte ho |
| CI / CD | Auto test / auto deploy |
| Idempotent | Do baar chalao toh bhi result same (payments mein zaroori) |
| Stateless | Server pe koi user-specific state nahi, isliye scale ho sakta hai |

---

## Progress tracker

- [ ] Ch 1: VPS specs note kiye
- [ ] Ch 2: Linux homework output samjha
- [ ] Ch 3: SSH key setup
- [ ] Ch 4: Ports + DNS map banaya
- [ ] Ch 5: PM2 setup samjha (path, env, restarts)
- [ ] Ch 6: Nginx + SSL status pata kiya
- [ ] Ch 7: DB location + backup status pata kiya
- [ ] Ch 8: Server hardening (non-root, ufw, fail2ban)
- [ ] Ch 9: Ek release-folder deploy + rollback practice (staging pe)
- [ ] Ch 10: Uptime monitor + logrotate
- [ ] Ch 11: Docker production image theek ki (optional)
- [ ] Ch 12: GitHub Actions CI
- [ ] Ch 13: API/worker split

> **Aakhri baat:** Har chapter ka homework output mujhe bhejte jao. Hum tumhare asli server ke saath aage badhenge, kyunki kitaab ka server kabhi production jaisa nahi hota.
