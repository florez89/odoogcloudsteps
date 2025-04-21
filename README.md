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
db_password = odoo
proxy_mode = True
logfile = /var/log/odoo/odoo.log
log_level = debug # (esto es opcional, porque coloca mucha informacion)

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
```
Version más completa:

server {
    listen 80;
    server_name accit.online www.accit.online;

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


### Step 17: Restart Nginx

reemplazo el config de Nginx por esta así no queda en bucle la app web por redirección dominio:

\# 🔁 Redirección de HTTP a HTTPS
server {
    listen 80;
    server_name accit.online www.accit.online;

    if ($host = www.accit.online) {
        return 301 https://$host$request_uri;
    }

    if ($host = accit.online) {
        return 301 https://$host$request_uri;
    }

    return 301 https://$host$request_uri;
}

\# 🔁 Definición de upstream para Odoo
upstream odoo {
    server 127.0.0.1:9069 weight=1 fail_timeout=0;
}

\# 🔒 Configuración HTTPS con proxy hacia Odoo
server {
    listen 443 ssl;
    server_name accit.online www.accit.online;

    ssl_certificate /etc/letsencrypt/live/accit.online/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/accit.online/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

    location / {
        proxy_pass http://odoo;

       \# Encabezados para WebSocket y proxy reverso
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port $server_port;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_buffering off;
        proxy_cache off;
        chunked_transfer_encoding off;
    }
}


### Step 18: Ingresar al log 

Inicio putty
Acceder al contenedor: docker exec -it odoo17fordocker-web-1 bash

Usar tail -f para ver el log: tail -f /var/log/odoo/odoo.log

para saber el nombre del contenedor ejecuro el comando: docker ps

son estos: 
![image](https://github.com/user-attachments/assets/314deca8-5878-460a-bb33-cabc109e8326)

### Step 19: agregar un dominio adicional: caso, academiago.online

✅ Paso 1: Configurar dbfilter en el contenedor Odoo
Abrí el archivo odoo.conf, que en tu instalación está en:

./config/odoo.conf

Y reemplazá la línea (o agregá si no está):

dbfilter = ^%d$

Esto significa que Odoo va a buscar bases de datos que coincidan con el subdominio o dominio usado en la URL. Ejemplos:

Si entrás a academiago.online 👉 solo cargará la base academiago.

Si entrás a accit.online 👉 solo cargará la base accit.

✅ Paso 2 (opcional): Permitir múltiples bases desde un dominio
Si querés que accit.online tenga acceso a todas las bases, se puede hacer una ligera modificación. Usá esta configuración:

dbfilter = ^%d$|^accit$

Con esto:

academiago.online 👉 solo podrá acceder a academiago

accit.online 👉 podrá acceder a todas las bases, incluyendo accit y academiago

También podés directamente quitar el dbfilter y dejarlo solo en la configuración de Nginx como te muestro abajo, si querés más flexibilidad.

- Pregunta a chat gpt: no deberia crear una carpeta en /etc/nginx/sites-available/ para academiago y mantener la de accit en sudo nano /etc/nginx/sites-available/Odoo?
  ¡Buena pregunta! Y sí, es una práctica totalmente válida (y recomendable) separar la configuración de cada dominio en archivos distintos dentro de /etc/nginx/sites-available/.

  ✅ ¿Por qué separar cada dominio en un archivo?
Separar la configuración de Nginx en archivos individuales por dominio te da:

Mejor organización: cada archivo se enfoca en un sitio específico (accit.online, academiago.online, etc.).

Facilidad de mantenimiento: si tenés que hacer cambios en un dominio, no afectás a los demás.

Escalabilidad: cuando tu servidor aloja muchos dominios (multisite), es más limpio y controlado.

🛠 Cómo hacerlo:
1. Dejás el archivo actual como Odoo para accit.online
sudo nano /etc/nginx/sites-available/Odoo

2. Creás uno nuevo para academiago.online
sudo nano /etc/nginx/sites-available/academiago
Y dentro de ese archivo pegás algo como esto:
server {
    listen 80;
    server_name academiago.online www.academiago.online;

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

3. Activás el nuevo sitio
sudo ln -s /etc/nginx/sites-available/academiago /etc/nginx/sites-enabled/
4. Verificás y reiniciás
sudo nginx -t
sudo systemctl restart nginx
5. instalo ssl certbort (en este paso no ejecuto  sudo apt install certbot python3-certbot-nginx -y porque ya hice esa intalación en accit.online)
   sudo certbot --nginx -d academiago.online -d www.academiago.online
6. sudo systemctl restart nginx

si queda en blucle la app pongo este archivo en la config de nginx mas completo:

reemplazo el config de Nginx por esta así no queda en bucle la app web por redirección dominio:

# 🔁 Redirección de HTTP a HTTPS server { listen 80; server_name accit.online www.accit.online;

if ($host = www.accit.online) {
    return 301 https://$host$request_uri;
}

if ($host = accit.online) {
    return 301 https://$host$request_uri;
}

return 301 https://$host$request_uri;
}

# 🔁 Definición de upstream para Odoo upstream odoo { server 127.0.0.1:9069 weight=1 fail_timeout=0; }

# 🔒 Configuración HTTPS con proxy hacia Odoo server { listen 443 ssl; server_name accit.online www.accit.online;

ssl_certificate /etc/letsencrypt/live/accit.online/fullchain.pem; # managed by Certbot
ssl_certificate_key /etc/letsencrypt/live/accit.online/privkey.pem; # managed by Certbot
include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

location / {
    proxy_pass http://odoo;

   \# Encabezados para WebSocket y proxy reverso
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Port $server_port;

    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";

    proxy_buffering off;
    proxy_cache off;
    chunked_transfer_encoding off;
}
}

NOTA: donde salga accit.online, debo colocar academiago.online
para que tome cambio los cambios en odoo.config, debo ejecutar: docker-compose restart

Nota: la diferencia entre docker-compose restart y docker-compose down docker-compose up, es que docker-compose restart reinicia el contenedor, y con la otra vuelve hacer un reload de yaml y hace cambios en los contendores. Este ultimo caso es si se hace cambios en el yaml


Como solucione el bucle web en academiago.online?

PD: use la misma config de accit.online solo reemplazando los nombre y dio errores:

acomode el archivo nginx de academiago como lo tenía en accit, ya que al instalar el modulo web de odoo quedaba en bucle. La configuración que coloque es:

"
# 🔁 Redirección de HTTP a HTTPS server { listen 80; server_name academiago.online www.academiago.online;

if ($host = www.academiago.online) {
    return 301 https://$host$request_uri;
}

if ($host = academiago.online) {
    return 301 https://$host$request_uri;
}

return 301 https://$host$request_uri;
}

# 🔁 Definición de upstream para Odoo upstream odoo { server 127.0.0.1:9069 weight=1 fail_timeout=0; }

# 🔒 Configuración HTTPS con proxy hacia Odoo server { listen 443 ssl; server_name academiago.online www.academiago.online;

ssl_certificate /etc/letsencrypt/live/academiago.online/fullchain.pem; # managed by Certbot
ssl_certificate_key /etc/letsencrypt/live/academiago.online/privkey.pem; # managed by Certbot
include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

location / {
    proxy_pass http://odoo;

   \# Encabezados para WebSocket y proxy reverso
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Port $server_port;

    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";

    proxy_buffering off;
    proxy_cache off;
    chunked_transfer_encoding off;
}
}
"

me da los siguientes errores:
"

root@vmi2550185:~# sudo nginx -t
nginx: [warn] duplicate value "TLSv1.2" in /etc/letsencrypt/options-ssl-nginx.conf:11
nginx: [warn] duplicate value "TLSv1.3" in /etc/letsencrypt/options-ssl-nginx.conf:11
nginx: [emerg] "ssl_prefer_server_ciphers" directive is duplicate in /etc/letsencrypt/options-ssl-nginx.conf:12
nginx: configuration file /etc/nginx/nginx.conf test failed
"
ayudame a solucionarlo e indicame porque el error?


ChatGPT Plus
Gracias por compartir la configuración. El error que te da Nginx tiene que ver con la inclusión duplicada de parámetros SSL en /etc/letsencrypt/options-ssl-nginx.conf, que ya define directivas como:

ssl_protocols TLSv1.2 TLSv1.3;

ssl_prefer_server_ciphers on;

🔍 ¿Qué significa el error?
Nginx te dice que estás repitiendo configuraciones SSL, en particular:

vbnet
Copiar
Editar
nginx: [warn] duplicate value "TLSv1.2"
nginx: [warn] duplicate value "TLSv1.3"
nginx: [emerg] "ssl_prefer_server_ciphers" directive is duplicate
Esto sucede cuando:

Ya están definidas esas directivas dentro de options-ssl-nginx.conf (archivo de configuración generado por Certbot).

Y también las estás declarando manualmente en tu archivo de sitio (academiago) o en otro lugar (como nginx.conf).

✅ Solución
Lo más común y limpio es no repetir ninguna directiva SSL si ya incluyes options-ssl-nginx.conf. Ese archivo ya tiene todo lo necesario. Por lo tanto:

🔧 Revisa tu archivo /etc/nginx/sites-available/academiago y asegúrate de que NO estás agregando extra SSL settings como estos:

nginx
Copiar
Editar
ssl_protocols TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers on;
Y que solo estés incluyendo esta línea:

nginx
Copiar
Editar
include /etc/letsencrypt/options-ssl-nginx.conf;
📦 Recomendación
Ya que estás usando Certbot, la configuración ideal sería así (te la reescribo limpia y sin errores):

nginx
Copiar
Editar
\# 🔁 Redirección HTTP → HTTPS
server {
    listen 80;
    server_name academiago.online www.academiago.online;

    if ($host = www.academiago.online) {
        return 301 https://$host$request_uri;
    }

    if ($host = academiago.online) {
        return 301 https://$host$request_uri;
    }

    return 301 https://$host$request_uri;
}

\# 🔁 Upstream para odoo
upstream odoo {
    server 127.0.0.1:9069 fail_timeout=0;
}

\# 🔒 HTTPS con Certbot
server {
    listen 443 ssl;
    server_name academiago.online www.academiago.online;

    ssl_certificate /etc/letsencrypt/live/academiago.online/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/academiago.online/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://odoo;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port $server_port;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_buffering off;
        proxy_cache off;
        chunked_transfer_encoding off;
    }
}
🧪 Luego de hacer los cambios:
Guarda el archivo (CTRL + O, luego ENTER y CTRL + X si estás usando nano).

Verifica la configuración:

bash
Copiar
Editar
sudo nginx -t
Si no hay errores, recarga Nginx:

bash
Copiar
Editar
sudo systemctl reload nginx
¿Querés que revise también si tenés esas directivas duplicadas en /etc/nginx/nginx.conf por si acaso?







Tú dijiste:

root@vmi2550185:~# sudo nginx -t
nginx: [emerg] duplicate upstream "odoo" in /etc/nginx/sites-enabled/academiago:18
nginx: configuration file /etc/nginx/nginx.conf test failed
root@vmi2550185:~#


ChatGPT Plus
¡Perfecto, gracias por el dato! El nuevo error es muy claro:

duplicate upstream "odoo"

Esto significa que ya definiste una sección upstream odoo en otro archivo (probablemente en el archivo de configuración de accit o en el archivo Odoo en /etc/nginx/sites-available/), y estás intentando volver a definir el mismo nombre (odoo) en academiago.

🛠️ ¿Cómo lo solucionás?
Tienes dos opciones simples:

✅ Opción 1: Usar un nombre diferente para el upstream
Renombrá el bloque upstream en academiago por otro nombre, por ejemplo odoo_academiago:

nginx
Copiar
Editar
\# 🔁 Upstream para odoo (nombre único)
upstream odoo_academiago {
    server 127.0.0.1:9069 fail_timeout=0;
}
Y luego cambiá el proxy_pass dentro del location /:

nginx
Copiar
Editar
proxy_pass http://odoo_academiago;
✅ Opción 2: No uses el bloque upstream y pasá directamente la IP
Si no necesitás configurar varios servidores o balanceo, podés simplificar y no usar upstream, así:

nginx
Copiar
Editar
location / {
    proxy_pass http://127.0.0.1:9069;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Port $server_port;

    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";

    proxy_buffering off;
    proxy_cache off;
    chunked_transfer_encoding off;
}
✅ Paso final:
Después de hacer uno de estos cambios:

Guardá el archivo.

Verificá la configuración:

bash
Copiar
Editar
sudo nginx -t
Si todo OK:

bash
Copiar
Editar
sudo systemctl reload nginx
¿Querés que te prepare la versión corregida con nombre de upstream distinto (odoo_academiago) o la simplificada sin upstream?







Tú dijiste:
ese error se debera porque en accit lo tengo asi?
"
\# 🔁 Redirección de HTTP a HTTPS 
server { listen 80; server_name accit.online www.accit.online;

if ($host = www.accit.online) {
    return 301 https://$host$request_uri;
}

if ($host = accit.online) {
    return 301 https://$host$request_uri;
}

return 301 https://$host$request_uri;
}

\# 🔁 Definición de upstream para Odoo upstream odoo { server 127.0.0.1:9069 weight=1 fail_timeout=0; }

\# 🔒 Configuración HTTPS con proxy hacia Odoo server { listen 443 ssl; server_name accit.online www.accit.online;

ssl_certificate /etc/letsencrypt/live/accit.online/fullchain.pem; # managed by Certbot
ssl_certificate_key /etc/letsencrypt/live/accit.online/privkey.pem; # managed by Certbot
include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

location / {
    proxy_pass http://odoo;

   \# Encabezados para WebSocket y proxy reverso
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Port $server_port;

    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";

    proxy_buffering off;
    proxy_cache off;
    chunked_transfer_encoding off;
}
}
"


ChatGPT Plus
¡Exactamente! 💡

Ese error se debe 100% a que en tu archivo de configuración de accit ya definiste este bloque:

nginx
Copiar
Editar
upstream odoo {
    server 127.0.0.1:9069 weight=1 fail_timeout=0;
}
Y luego, volvés a definirlo con el mismo nombre odoo en el archivo de configuración de academiago. Como Nginx carga todos los archivos desde sites-enabled/, no permite duplicados de upstream con el mismo nombre.

✅ ¿Cómo se soluciona?
Como dijimos antes, dos formas:

✅ Opción rápida: cambiar el nombre del upstream en academiago
nginx
Copiar
Editar
\# en academiago
upstream odoo_academiago {
    server 127.0.0.1:9069 fail_timeout=0;
}
Y más abajo:

nginx
Copiar
Editar
proxy_pass http://odoo_academiago;
✅ Opción simple: quitar upstream y usar la IP directamente en proxy_pass:
nginx
Copiar
Editar
proxy_pass http://127.0.0.1:9069;
✅ Luego hacé:
bash
Copiar
Editar
sudo nginx -t
sudo systemctl reload nginx
Y ya te debería cargar todo bien 😎



### TESTING: YOUR APP
```bash
http://<your-domain>:9069 (en el paso 8)

despues del paso 14 ya puedo entrar a la pagina de inicio de sesion de odoo en midominio.com
```


