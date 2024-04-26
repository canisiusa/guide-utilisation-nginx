
# Guide d'utilisation de Nginx et Docker sur différentes architectures

## Configuration de Nginx

### Serveur pour Application React

```cree le ficher conf nginx
nano /etc/nginx/conf.d/votredomaine.com.conf
```
```nginx
server {
    listen 80;
    server_name votre_domaine.com;

    location / {
        root /var/www/mon-projet-frontend;
        try_files $uri $uri/ /index.html;
        index index.html index.htm;
    }
}
```

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
