
# Guide d'utilisation de Nginx et Docker sur différentes architectures

## Configuration de Nginx

### Serveur pour Application React

```cree le ficher conf nginx
nano /etc/nginx/conf.d/votredomaine.com.conf
```
```server {
    listen 80;
    server_name example.com www.example.com;  # Adjust to your domain
    # Redirect HTTP traffic to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    # SSL configuration
    listen 443 ssl http2;
    server_name example.com www.example.com;  # Adjust to your domain

    ssl_certificate /path/to/your/fullchain.pem;  # Path to your SSL certificate
    ssl_certificate_key /path/to/your/privkey.pem;  # Path to your SSL certificate key
    ssl_session_timeout 5m;
    ssl_protocols TLSv1.2 TLSv1.3;  # Recommended protocols
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384';
    ssl_prefer_server_ciphers on;

    root /path/to/your/react/app/build;
    index index.html;
    try_files $uri $uri/ /index.html;

    access_log /var/log/nginx/react_access.log;
    error_log /var/log/nginx/react_error.log;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";
    add_header X-XSS-Protection "1; mode=block";

    gzip on;
    gzip_disable "msie6";
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_buffers 16 8k;
    gzip_http_version 1.1;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
}

```
`sudo nginx -t`
`sudo systemctl reload nginx`

### Installation de Certbot pour SSL

```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d votre_domaine.com -d www.votre_domaine.com
sudo certbot renew --dry-run
```

### Docker et Docker Compose sur Ubuntu

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt update
sudo apt install docker-ce
sudo systemctl status docker
sudo usermod -aG docker ${USER}
sudo curl -L "https://github.com/docker/compose/releases/download/v2.10.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version
```

### Configuration API Nginx

```cree le ficher conf nginx
nano /etc/nginx/conf.d/api.votredomaine.com.conf
```

```nginx
server {
    listen 80;
    server_name api.votredomaine.com;

    location / {
        proxy_pass http://localhost:3030;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
`sudo nginx -t`
`sudo systemctl reload nginx`

### Diagnostic de l'erreur 500

- Vérifiez les logs de Nginx : `sudo tail -f /var/log/nginx/error.log`
- Examinez les logs de l'application
- Assurez-vous que la configuration de Nginx est correcte : `sudo nginx -t`
- Vérifiez les permissions de fichiers
- Testez les composants de l'infrastructure

### Utilisation de Docker Buildx pour la compatibilité multi-architecture

```bash
docker buildx create --name mybuilder --use
docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
docker buildx build --platform linux/amd64,linux/arm64 -t monnomutilisateur/monimage:latest . --push
```
