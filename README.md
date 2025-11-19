# 🚀 **Déploiement du Projet Coursa – Proxmox**

<p align="center">
  <img width="1355" height="686" alt="image" src="https://github.com/user-attachments/assets/1b7fd177-2409-4314-8053-04425bc5678a" />
</p>

<p align="center">
  <b>Backend : NestJS • Frontend : Next.js • Base de données : Prisma• Déploiement : Proxmox + Docker</b>
</p>

---

# 📚 **Sommaire**

1. [Prérequis](#-prérequis)
2. [Installation des outils](#-installation-des-outils)
3. [Installation Docker & Docker Compose](#-installer-docker-et-docker-compose)
4. [Clonage des projets GitHub](#-cloner-les-projets-github)
5. [Dockerfile Backend/Frontend](#-exemple-de-dockerfile)
6. [docker-compose.yml complet](#-docker-composeyml-backend--frontend--postgresql)
7. [Lancement des services](#-lancer-les-services-docker)
8. [Reverse Proxy (NGINX Proxy Manager)](#-reverse-proxy-nginx-proxy-manager)
9. [Déploiement sur Proxmox (VM/LXC)](#-déploiement-complet-sur-proxmox)
10. [Mises à jour continue](#-mise-à-jour-après-push-github)
11. [Vérification des services](#-vérifier-les-services)
12. [Conclusion](#-conclusion)

---

# 📌 **Prérequis**

### ✔ Serveur Proxmox

🔗 **Panel Proxmox :** [https://ns3107703.ip-54-36-179.eu:8006](https://ns3107703.ip-54-36-179.eu:8006)

Vous devez disposer de :

* Un serveur **Proxmox VE** fonctionnel
* Une VM ou un conteneur **LXC Ubuntu**

<img width="1355" height="519" alt="image" src="https://github.com/user-attachments/assets/05f90c96-9f82-46da-99f0-bb21a07b115c" />

<img width="719" height="538" alt="image" src="https://github.com/user-attachments/assets/00e4da86-38ec-4cf7-8e39-ac1d130f0702" />

* **Docker + Docker Compose** installés
* Vos projets GitHub :

| Projet           | Lien                                                                                                       |
| ---------------- | ---------------------------------------------------------------------------------------------------------- |
| Backend NestJS   | [https://github.com/melotrex/coursa-backend](https://github.com/melotrex/coursa-backend)                   |
| Frontend Next.js | [https://github.com/melotrex/coursa_frontend_senegal](https://github.com/melotrex/coursa_frontend_senegal) |

---

# 📌 **Installation des outils**

*Connecter *

<img width="715" height="177" alt="image" src="https://github.com/user-attachments/assets/a87530fb-29f3-4e85-bf8b-88444bc6dfa1" />


```bash
sudo apt update && sudo apt upgrade -y
sudo apt install git curl wget htop net-tools -y
```

### 🖼 Capture d'installation des outils

<p align="center">
  <img src="https://github.com/user-attachments/assets/7e8faa90-f71a-4769-a166-38be1ddeafc5" width="480" />
</p>

---

# 📌 **Vérification des outils installés**

<p align="center">
  <img src="https://github.com/user-attachments/assets/31d8721f-2d3b-4e8e-9a58-cedfd4627a50" width="480" />
</p>

<p align="center">
  <img src="https://github.com/user-attachments/assets/74581638-3f87-4c0f-b130-5ec241602a90" width="260" />
</p>

---

# 📌 **Installer Docker et Docker Compose**

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install ca-certificates curl gnupg git -y
```

Ajout du dépôt Docker :

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo tee /etc/apt/keyrings/docker.asc > /dev/null

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Installation :

```bash
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y
```

Vérification :

```bash
docker --version
docker compose version
```

---

# 📌 **Cloner les projets GitHub**

### Backend NestJS

```bash
git clone https://github.com/melotrex/coursa-backend
cd coursa-backend
```

### Frontend NextJS

```bash
git clone https://github.com/melotrex/coursa_frontend_senegal
cd coursa_frontend_senegal
```

---

# 📌 **Exemple de Dockerfile**

## 🔹 Backend NestJS

```Dockerfile
FROM node:20


RUN apt-get update && apt-get install -y wget postgresql-client \
    && wget https://github.com/jwilder/dockerize/releases/download/v0.6.1/dockerize-linux-amd64-v0.6.1.tar.gz \
    && tar -C /usr/local/bin -xzvf dockerize-linux-amd64-v0.6.1.tar.gz \
    && rm dockerize-linux-amd64-v0.6.1.tar.gz


WORKDIR /app

COPY package*.json ./

RUN npm install -force

RUN npm install -g prisma

COPY prisma ./prisma

RUN npx prisma generate

COPY . .

RUN npm run build

EXPOSE 3000

# CMD npx prisma migrate dev && npm run start:dev
# CMD ["dockerize", "-wait", "tcp://postgres:5432", "-timeout", "60s", "sh", "-c", "npx prisma migrate dev && npx prisma db seed && npm run start:dev"]
CMD ["dockerize", "-wait", "tcp://postgres:5432", "-timeout", "60s", "sh", "-c", "npx prisma migrate deploy && npm run build && npm run start"]
```

## 🔹 Frontend Next.js

```Dockerfile
# Step 1: Build stage
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package info and install deps
COPY package*.json ./
RUN npm install --force

# Copy the rest of the app
COPY . .

# Build the Next.js project
RUN npm run build

# Step 2: Production stage
FROM node:18-alpine AS runner

WORKDIR /app

ENV NODE_ENV=production
ENV PORT=5173

# Copy only necessary files from builder
COPY --from=builder /app/public ./public
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./package.json
COPY --from=builder /app/next.config.mjs ./next.config.mjs

# Expose the desired port
EXPOSE 5173

# Start the app
CMD ["npm", "start"]
```

---

# 📌 **docker-compose.yml – Backend + Frontend + PostgreSQL**

```yaml
services:
  frontend:
    build:
      context: ./
      dockerfile: Dockerfile
    ports:
      - "5173:5173"
    container_name: frontend
    networks:
      - app-net

networks:
  app-net:
    driver: bridge

volumes:
  postgres_data:
```

---

# 📌 **Lancer les services Docker**

```bash
docker compose up -d --build
```

Vérifier :

```bash
docker ps
```

---

# 📌 **Reverse Proxy – NGINX Proxy Manager (Recommandé)**

Permet d'avoir :

* [https://app.coursa.com](https://app.coursa.com) → Frontend (3000)
* [https://api.coursa.com](https://api.coursa.com) → Backend (3001)

Installation :

```bash
docker compose up -d
```

Ajouter vos domaines → activer **SSL Let's Encrypt**.

---

# 📌 **Déploiement complet sur Proxmox**

## 1️⃣ Créer une VM Ubuntu

* CPU : 2 vCPU
* RAM : 2–4 Go
* Disque : 20 Go
* Réseau : `vmbr0`

## 2️⃣ Installer Docker & outils

→ (voir étapes précédentes)

## 3️⃣ Cloner projets GitHub

## 4️⃣ Lancer Docker

## 5️⃣ Configurer DNS + Proxy (optionnel)

---

# 📌 **Mise à jour après push GitHub**

### Backend

```bash
cd coursa-backend
git pull
docker compose up -d --build backend
```

### Frontend

```bash
cd coursa_frontend_senegal
git pull
docker compose up -d --build frontend
```

---

# 📌 **Vérifier les services**

```bash
docker ps
```

Vous devez voir :

* ✔ Backend : port 3001
* ✔ Frontend : port 3000
* ✔ Prisma: port 5432

---

# 🎯 **Conclusion**

Votre infrastructure **Coursa** est maintenant entièrement opérationnelle :

✨ Backend NestJS
✨ Frontend NextJS
✨ Base PostgreSQL
✨ Déployés avec Docker
✨ Hébergés sur Proxmox
✨ Option SSL + Proxy disponible





