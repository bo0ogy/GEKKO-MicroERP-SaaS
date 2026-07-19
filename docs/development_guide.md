# Guide de Développement et de Production - GEKKO Micro-ERP

Ce guide documente les standards, les procédures et l'architecture à suivre par l'équipe de développement pour mener à bien le projet GEKKO Micro-ERP, en parfaite adéquation avec le `scrum_backlog.md`.

---

## 1. Prérequis & Environnement de Base

Avant de commencer le développement (Sprint 0), chaque développeur doit configurer son environnement local :

- **Node.js** : Version LTS recommandée (ex: v20.x).
- **Gestionnaire de paquets** : `npm` ou `pnpm` (à standardiser dans l'équipe).
- **Docker & Docker Compose** : Obligatoire pour faire tourner la base de données PostgreSQL en local.
- **Git** : Pour le contrôle de version.
- **IDE** : VS Code recommandé avec les extensions ESLint, Prettier, et Prisma.

---

## 2. Exécution du Sprint 0 (Initialisation)

### 2.1. Backend (NestJS)
Le backend sert de fondation solide et sécurisée.
**Actions à réaliser :**
```bash
# Installation de la CLI NestJS globale
npm i -g @nestjs/cli

# Création du projet backend
nest new backend
```
* **Standards :** Strict TypeScript, configuration de `ESLint` et `Prettier` imposée dès le premier jour. 
* **Validation :** Utilisation de `class-validator` et `class-transformer` pour blinder les DTO (Data Transfer Objects).

### 2.2. Base de données (PostgreSQL & Prisma)
**Actions à réaliser :**
1. Création d'un `docker-compose.yml` à la racine pour PostgreSQL.
2. Initialisation de Prisma dans le dossier `backend` :
```bash
cd backend
npm install prisma --save-dev
npx prisma init
```
* **Standards :** Toutes les migrations de base de données doivent passer par Prisma (`npx prisma migrate dev`). Aucune modification manuelle de la BDD.

### 2.3. Frontend (Framework au choix, ex: Next.js)
**Actions à réaliser :**
```bash
npx create-next-app@latest frontend
```
* **Standards :** Intégration des templates statiques existants de GEKKO. Mise en place de TailwindCSS (ou du CSS pur existant) et paramétrage du routeur.

---

## 3. Lignes Directrices pour le Code (Sprints 1 à 5)

Pour garantir la pérennité du code tout au long des Sprints, les règles suivantes s'appliquent :

### Architecture Backend (NestJS)
* **Modularité :** Chaque fonctionnalité majeure doit avoir son propre module (ex: `AuthModule`, `CrmModule`, `InvoiceModule`).
* **Séparation des responsabilités :**
  * `Controllers` : Uniquement pour gérer les requêtes/réponses HTTP.
  * `Services` : Contient toute la logique métier complexe (ex: Moteur de calcul des devis - US 3.1).
  * `Guards` : Pour sécuriser les routes (JWT Auth & RBAC - US 1.2 & US 1.3).

### Sécurité & Multi-Tenancy (CRITIQUE)
Étant donné qu'il s'agit d'un SaaS, l'isolation des données est absolue.
* **Prisma Middleware (Client Extension) :** Implémenter une extension Prisma pour injecter automatiquement une clause `where: { organizationId: currentTenantId }` sur toutes les requêtes afin de garantir qu'une organisation ne verra jamais les données d'une autre (US 1.2).

### Transactions & Intégrité (Sprints 4)
* **Verrous (Locks) :** Pour la génération des factures (US 4.1 et 4.2), utiliser `prisma.$transaction` et des verrous au niveau de la ligne (Row-Level Locking) pour garantir que la numérotation légale des factures se fait de manière incrémentale et sans conflits, même si deux collaborateurs créent une facture à la même milliseconde.

---

## 4. Préparation pour la Production

La mise en production doit être pensée dès le Sprint 1.

### 4.1. Variables d'Environnement
* Ne **jamais** commiter de fichiers `.env` contenant des secrets.
* Utiliser `@nestjs/config` dans le backend avec une validation stricte via `Joi` ou `zod` pour s'assurer que le serveur ne démarre pas s'il manque une variable de production (ex: `DATABASE_URL`, `JWT_SECRET`).

### 4.2. Dockerisation
Créer des `Dockerfile` Multi-Stage pour le Backend et le Frontend :
* **Stage de Build :** Installe toutes les dépendances (y compris devDependencies) et compile le code (TypeScript vers JavaScript).
* **Stage de Run :** Ne conserve que le code compilé (`dist`), le dossier `node_modules` de production, et utilise une image légère (ex: `node:20-alpine`) pour réduire la surface d'attaque et le poids.

### 4.3. Pipeline CI/CD (Intégration et Déploiement Continus)
Mettre en place des GitHub Actions ou GitLab CI pour :
1. Linter tout le code (Frontend & Backend).
2. Vérifier que la compilation TypeScript réussit.
3. Construire les images Docker.
4. Les pousser sur un registre (Docker Hub ou AWS ECR).

### 4.4. Base de données en Production
* Exécuter `npx prisma migrate deploy` dans le pipeline de déploiement (jamais `migrate dev` en prod).
* Assurer des sauvegardes automatisées (Backups) régulières de la base PostgreSQL.
