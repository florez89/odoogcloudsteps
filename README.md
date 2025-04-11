# Google Cloud Setup for Odoo with Docker and Nginx

---
### Step 0: Watch the n8n Google installation video to configure your firewall for port 9069
### Start at minute 09:47
```bash
https://www.youtube.com/watch?v=_Ch_iB6dHsg&list=PLlOfVkvrHKVSt1-t6dMbXGX8a4iyAIxhz&index=4
```


### Step 1: Check Disk Space
```bash
df -h
```

### Step 2: Update System Packages
```bash
sudo apt update
sudo apt-get update
```


### Step 3: Install Docker
```bash
sudo apt install docker.io
sudo systemctl start docker
sudo systemctl enable docker
```

### Step 4: Add User to Docker Group
```bash
4.1 whoami (primero copio este para saber cual es mi usuario)
4.2 sudo usermod -aG docker (coloco el usuario obtenido con el comando whoami)
4.3 exit (cierro la terminal y vuelvo abrirla para que tome los efectos)
```
Login again and verify the user is added to the Docker group.

---

### Step 5: Install Docker Compose
```bash
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version
```

### Step 6: Install Git
```bash
sudo apt install -y git
```

### Step 7: Clone Odoo Repository
```bash
7.1) git clone https://github.com/florez89/odoo17fordocker.git
7.2) cd odoo17fordocker
7.3) git switch development/production (opcional)

(si quiero saber en que rama estoy, coloco el comando git branch
```

### Step 8: Start Docker Compose
```bash
Ejecuto uno de los siguientes comandos
docker-compose up (si quiero ver los logs) 
sudo docker-compose up -d (si no quiero ver los logs)
```



### Step 9: Restart Docker Compose (opcional) voy al paso 10 para crear el archivo.config
```bash
docker-compose down
docker-compose up
```

### Step 10: Configure Odoo
```bash
sudo nano ./config/odoo.conf
```
Add the following configuration:
```ini
[options]
addons_path = /mnt/extra-addons
admin_passwd = admin
db_host = db
db_port = 5432
db_user = odoo
db_password = odoo (esta clave esta en el archivo odoo pass de https://github.com/florez89/odoo17fordocker)
proxy_mode = True
```
Save and exit with `CTRL+O`, `ENTER`, `CTRL+X`.
quita este error: ![image](https://github.com/user-attachments/assets/de971575-42d9-4535-acfc-d356332bb723)


---

### Step 11: Install Nginx
```bash
sudo apt install -y nginx

compruebo entrando a midominio.com y me debe salir el mensaje por defecto de nginx
```

### Step 12: Configure Nginx for Odoo
```bash
sudo nano /etc/nginx/sites-available/Odoo
```
Add the following configuration:
```nginx
server {
    listen 80;
    server_name YOUR-DOMAIN-NAME;

    location / {
        proxy_pass http://localhost:9069;
        proxy_http_version 1.1;
        chunked_transfer_encoding off;
        proxy_buffering off;
        proxy_cache off;

        # Headers for WebSocket support
        proxy_set_header Connection 'Upgrade';
        proxy_set_header Upgrade $http_upgrade;

        # Additional headers for forwarding client info
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
Save and exit with `CTRL+O`, `ENTER`, `CTRL+X`.

### Step 13: Enable Nginx Site and Test Configuration
```bash
13.1) sudo ln -s /etc/nginx/sites-available/Odoo /etc/nginx/sites-enabled/
13.2) sudo nginx -t
```
### Step 14: Restart Nginx
```bash
sudo systemctl restart nginx

luego de esto dejo de ver el mensaje por defecto de nginx y muestra la pagina de inicio de sesion de odoo
```

### Step 15:Install Certbot and Set Up SSL with Certbot
```bash
15.1) sudo apt install certbot python3-certbot-nginx -y
15.2) sudo certbot --nginx -d YOUR-DOMAIN-NAME
```

### Step 16: Restart Nginx
```bash
sudo systemctl restart nginx
```
ya puedo abrir la pagina modo seguro

### TESTING: YOUR APP
```bash
http://<your-domain>:9069 (en el paso 8)

despues del paso 14 ya puedo entrar a la pagina de inicio de sesion de odoo en midominio.com
```
