# Runbook : Échec du pipeline CI/CD — Erreur de génération Prisma Client

## 1. Sujet

Échec du pipeline GitHub Actions lors du build Docker : Prisma Client introuvable ou génération échouée.

## 2. Problème traité

Le pipeline CI/CD échoue à l'étape `docker/build-push-action` avec une erreur de type :

```
Error: PrismaClientInitializationError: Prisma Client could not be located
```

ou

```
Error: Generated Prisma Client could not be found.
```

Cela bloque le déploiement de l'application et empêche la mise en production.

## 3. Symptômes

- Le workflow GitHub Actions est marqué en **rouge** sur la branche `main`
- L'étape "Build Docker image" échoue avec le message : `PrismaClientInitializationError`
- Les logs contiennent : `ENOENT: no such file or directory, open '.../node_modules/.prisma/client/index.js'`
- L'image Docker n'est pas poussée sur DockerHub
- Aucun déploiement n'est possible tant que le pipeline est rouge

## 4. Public cible

- Développeur backend (niveau bac+3)
- DevOps junior en charge du pipeline CI/CD
- Toute personne ayant les droits de commit sur la branche `main`

## 5. Quand appliquer ce runbook

**Immédiatement** dès qu'un commit sur `main` produit un pipeline rouge avec l'erreur Prisma Client.

## 6. Quand ne PAS appliquer ce runbook

- Si l'erreur est un problème de base de données (serveur MySQL inaccessible) — voir le runbook dédié
- Si l'erreur est un problème de dépendances npm (package introuvable) — exécuter `npm ci` en local d'abord
- Si le pipeline échoue aux tests unitaires — corriger le code d'abord
- Si l'erreur est un timeout DockerHub — relancer le workflow manuellement

## 7. Étapes à suivre

### Étape 1 : Diagnostiquer

```bash
# Aller dans l'onglet Actions du repo GitHub
# Cliquer sur le run échoué
# Ouvrir les logs de l'étape "Build Docker image"
# Chercher "PrismaClientInitializationError" ou "ENOENT"
```

### Étape 2 : Vérifier le Dockerfile

```dockerfile
# Le Dockerfile doit contenir ces deux étapes clés :
# 1. Builder stage : npx prisma generate
# 2. Final stage : copier node_modules/.prisma depuis le builder

FROM node:22-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY prisma ./prisma
RUN npx prisma generate          # <-- Vérifier que cette ligne existe
COPY tsconfig.json ./
COPY src ./src
RUN npm run build

FROM node:22-alpine AS prod-deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
COPY prisma ./prisma
RUN npx prisma generate          # <-- Vérifier que cette ligne existe aussi

FROM node:22-alpine
WORKDIR /app
COPY --from=prod-deps /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
```

### Étape 3 : Vérifier le schéma Prisma

```bash
# Vérifier que prisma/schema.prisma existe et est valide
cat prisma/schema.prisma

# Vérifier que le fichier schema-test.prisma n'écrase pas le générateur
# Le fichier schema-test.prisma doit avoir un output différent :
generator client {
  provider = "prisma-client-js"
  output   = "../node_modules/.prisma/client-test"  # <-- output différent
}
```

### Étape 4 : Tester la génération en local

```bash
# Nettoyer le cache Prisma
npx prisma generate --force

# Vérifier que le client est bien généré
ls node_modules/.prisma/client/index.js
# Doit afficher : node_modules/.prisma/client/index.js
```

### Étape 5 : Vérifier le .dockerignore

```bash
# Vérifier que .dockerignore n'exclut PAS prisma/
cat .dockerignore
# Doit contenir :
# node_modules
# dist
# coverage
# reports
# .git
# .env
# *.test.ts
# *.test.js
# prisma/test.db
# prisma/schema-test.prisma
#
# NE PAS exclure : prisma/schema.prisma
```

### Étape 6 : Forcer la regénération et repousser

```bash
# En local
npm ci
npx prisma generate
git add -A
git commit -m "fix: regenerate Prisma client"
git push
```

### Étape 7 : Vérifier le pipeline

- Aller sur <https://github.com/LTOssian/cicd-tasklist-backend/actions>
- Vérifier que le nouveau run passe au vert
- Vérifier que l'image est poussée sur DockerHub

## 8. Post-mortem

Après résolution, mettre à jour ce runbook si la cause était différente de celles listées.

## 9. Liens utiles

- Pipeline CI/CD : <https://github.com/LTOssian/cicd-tasklist-backend/actions>
- DockerHub : <https://hub.docker.com/r/LTOssian/cicd-tasklist-backend>
- Documentation Prisma : <https://www.prisma.io/docs/orm/prisma-client/setup-and-configuration/generating-prisma-client>
- SonarQube : <https://sonarqube.cicd.kits.ext.educentre.fr/dashboard?id=louisan-cicd-tasklist-backend>
