# NestJS Application Deployment Guide

This guide covers setting up a NestJS application using Docker and PostgreSQL, deploying it with Nginx, managing database backups, and automating deployment with GitHub Actions.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Local Development Setup](#local-development-setup)
  - [Docker and PostgreSQL](#docker-and-postgresql)
  - [NestJS and Prisma Setup](#nestjs-and-prisma-setup)
- [Dockerization](#dockerization)
- [Nginx Configuration](#nginx-configuration)
- [Database Backup Management](#database-backup-management)
- [GitHub Actions Automation](#github-actions-automation)
- [Server Setup](#server-setup)
- [Security Considerations](#security-considerations)

## Prerequisites

- Docker and Docker Compose
- Node.js and npm
- Git
- Access to an AWS account with S3
- A server with Nginx installed

## Local Development Setup

### Docker and PostgreSQL

1. **Create a `docker-compose.yml` file** in your project root:
   ```yaml
   version: '3.8'
   services:
     postgres:
       image: postgres:latest
       environment:
         POSTGRES_DB: dbname
         POSTGRES_USER: user
         POSTGRES_PASSWORD: password
       ports:
         - "5432:5432"
       volumes:
         - db-data:/var/lib/postgresql/data
     app:
       build: .
       ports:
         - "3000:3000"
       environment:
         DATABASE_URL: postgresql://user:password@postgres/dbname
       depends_on:
         - postgres
   volumes:
     db-data:


### NestJS and Prisma Setup

1. **Install Prisma CLI and client**
 ```bash
 npm install @prisma/cli --save-dev
 npm install @prisma/client
```

2. **Initialize Prisma**
   ```bash
   npx prisma init
    ```
   
3. **Configure your .env file:**
  ```bash
  DATABASE_URL="postgresql://user:password@localhost:5432/dbname?schema=public"
  ```

4. **Update the schema.prisma file:**
   ```plaintext
         datasource db {
        provider = "postgresql"
        url      = env("DATABASE_URL")
      }
      generator client {
        provider = "prisma-client-js"
      }
    ```
5. **Generate the Prisma client**
    ```bash
      npx prisma generate
    ```

## Dockerization
1. **Create a Dockerfile in the project root:**
     ```dockerfile
      FROM node:14-alpine
      RUN addgroup -S appgroup && adduser -S appuser -G appgroup
      WORKDIR /usr/src/app
      COPY package*.json ./
      RUN npm install
      COPY . .
      RUN npm run build
      USER appuser
      EXPOSE 3000
      CMD ["node", "dist/main"]
    ```
2. **Build and push the Docker image:**
   ```bash
     docker build -t yourusername/nestjs-app:latest .
    docker push yourusername/nestjs-app:latest
   ```

## Database Backup Management

1. **Installer les Dépendances Nécessaires**
   Vous aurez besoin des packages suivants :

- **@nestjs/schedule** pour la planification des tâches cron.
- **pg_dump** dans votre système pour le backup de PostgreSQL.
- **aws-sdk** pour l'interaction avec AWS S3.
  
Installez les packages avec npm ou yarn :
```bash
npm install @nestjs/schedule aws-sdk
npm install @types/node --save-dev
```

2. **Configuration du Module de Planification**
Créez un module de planification dans votre application NestJS :

```typescript
// src/schedule/schedule.module.ts
import { Module } from '@nestjs/common';
import { ScheduleModule } from '@nestjs/schedule';
import { TasksService } from './tasks.service';

@Module({
  imports: [
    ScheduleModule.forRoot(),
  ],
  providers: [TasksService],
})
export class AppScheduleModule {}

```

3. **Service pour la Tâche Cron**
```typescript
// src/schedule/tasks.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { Cron } from '@nestjs/schedule';
import { exec } from 'child_process';
import { S3 } from 'aws-sdk';

@Injectable()
export class TasksService {
  private readonly logger = new Logger(TasksService.name);
  private s3 = new S3();

  @Cron('0 0 * * 0') // Exécute la tâche chaque dimanche à minuit
  async handleCron() {
    const date = new Date().toISOString().slice(0, 10); // YYYY-MM-DD
    const fileName = `backup-${date}.sql`;
    const filePath = `/tmp/${fileName}`;

    // Commande pour créer un backup de la base de données
    exec(`pg_dump -U your_db_user -d your_db_name -f ${filePath}`, async (error, stdout, stderr) => {
      if (error) {
        this.logger.error(`Backup error: ${error.message}`);
        return;
      }

      this.logger.debug('Database backup created successfully.');

      // Upload to S3
      const fileContent = await fs.promises.readFile(filePath);
      const params = {
        Bucket: 'your_bucket_name',
        Key: `bdd/${fileName}`,
        Body: fileContent,
      };

      this.s3.upload(params, (err, data) => {
        if (err) {
          this.logger.error(`Failed to upload backup to S3: ${err.message}`);
        } else {
          this.logger.debug(`Backup successfully uploaded to S3: ${data.Location}`);
        }
      });
    });
  }
}
```
## GitHub Actions Automation
1. **Préparer le Serveur**
Sur votre serveur, assurez-vous que Docker et Docker Compose sont installés et configurés. Vous devriez aussi configurer l'authentification sans mot de passe pour SSH afin de permettre à GitHub Actions de se connecter et exécuter les commandes nécessaires.

2. **Configuration des Secrets GitHub**
   Ajoutez les secrets suivants à votre repository GitHub pour sécuriser les informations sensibles :

  - **DOCKER_USERNAME** : votre nom d'utilisateur Docker Hub.
  - **DOCKER_PASSWORD** : votre mot de passe Docker Hub.
  - **HOST** : l'adresse IP ou le nom de domaine de votre serveur.
  - **SSH_KEY** : votre clé privée SSH, utilisée pour se connecter au serveur (assurez-vous que la clé correspondante est ajoutée aux clés SSH autorisées sur le serveur).
  - **DEPLOY_PATH** : le chemin sur le serveur où se trouve votre docker-compose.yml.
    
3. **Créer le Fichier de Workflow GitHub Actions**
Créez un fichier `**.github/workflows/deploy.yml**` dans votre projet avec le contenu suivant :
```yaml
name: Deploy to Production Server

on:
  push:
    branches:
      - main  # Définir la branche qui déclenche le workflow

jobs:
  build_and_push:
    runs-on: ubuntu-latest
    steps:
      - name: Check Out Code
        uses: actions/checkout@v2

      - name: Log in to Docker Hub
        uses: docker/login-action@v1
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and Push Docker image
        run: |
          docker build -t ${{ secrets.DOCKER_USERNAME }}/nestjs-app:latest .
          docker push ${{ secrets.DOCKER_USERNAME }}/nestjs-app:latest

  deploy:
    runs-on: ubuntu-latest
    needs: build_and_push
    steps:
      - name: Deploy to Server
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.HOST }}
          username: root  # ou tout autre utilisateur non-root ayant les permissions nécessaires
          key: ${{ secrets.SSH_KEY }}
          script: |
            cd ${{ secrets.DEPLOY_PATH }}
            docker-compose pull
            docker-compose down
            docker-compose up -d
```
