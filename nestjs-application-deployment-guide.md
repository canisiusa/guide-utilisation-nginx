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

