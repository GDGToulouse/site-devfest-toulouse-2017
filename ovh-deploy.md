# Déploiement sur Hébergement Mutualisé OVH

Ce guide explique comment déployer le site DevFest Toulouse 2017 sur un hébergement web mutualisé OVH, avec ou sans Firebase.

## Table des matières

- [Prérequis](#prérequis)
- [1. Build du projet](#1-build-du-projet)
- [2. Upload des fichiers](#2-upload-des-fichiers)
- [3. Configuration Apache (.htaccess)](#3-configuration-apache-htaccess)
- [4. Configuration DNS](#4-configuration-dns)
- [5. Configuration Multisite OVH](#5-configuration-multisite-ovh)
- [6. Vérifications post-déploiement](#6-vérifications-post-déploiement)
- [7. Se passer de Firebase (mode statique)](#7-se-passer-de-firebase-mode-statique)

---

## Prérequis

- Un hébergement web mutualisé OVH (Pro ou Performance recommandé)
- Accès FTP/SFTP à l'hébergement
- Node.js et npm installés localement
- Le nom de domaine `devfesttoulouse.fr` configuré chez OVH

---

## 1. Build du projet

En local, générer le build de production :

```bash
# Installer les dépendances
npm install
bower install

# Générer le build de production
npm run build
```

Le dossier `build/` contient les fichiers prêts à être déployés.

---

## 2. Upload des fichiers

### Connexion FTP

- **Hôte** : `ftp.cluster0XX.hosting.ovh.net` (voir espace client OVH)
- **Utilisateur** : ton identifiant FTP OVH
- **Port** : 21 (FTP) ou 22 (SFTP)

### Structure sur le serveur

Uploader le **contenu** du dossier `build/` vers :

```
/home/ton_login/2017.devfesttoulouse.fr/
```

Structure finale :

```
/home/ton_login/
└── 2017.devfesttoulouse.fr/
    ├── .htaccess              # À créer (voir section suivante)
    ├── index.html
    ├── manifest.json
    ├── service-worker.js
    ├── src/
    ├── data/
    ├── images/
    ├── scripts/
    └── bower_components/
```

---

## 3. Configuration Apache (.htaccess)

Créer un fichier `.htaccess` à la racine du site (`/home/ton_login/2017.devfesttoulouse.fr/.htaccess`) :

```apache
# ============================================
# Configuration Apache pour DevFest Toulouse
# Site Polymer SPA (Single Page Application)
# ============================================

# Activer le moteur de réécriture
RewriteEngine On

# -----------------------------------------
# Forcer HTTPS (recommandé pour PWA/Service Worker)
# -----------------------------------------
RewriteCond %{HTTPS} off
RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]

# -----------------------------------------
# Routage SPA (Single Page Application)
# Redirige toutes les URLs vers index.html
# sauf si le fichier ou dossier existe
# -----------------------------------------
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule ^(.*)$ /index.html [L,QSA]

# -----------------------------------------
# Clean URLs - Supprimer .html des URLs
# -----------------------------------------
RewriteCond %{REQUEST_FILENAME} !-d
RewriteCond %{REQUEST_FILENAME}\.html -f
RewriteRule ^(.*)$ $1.html [L]

# -----------------------------------------
# Cache des assets statiques
# -----------------------------------------
<IfModule mod_expires.c>
    ExpiresActive On

    # HTML - cache court (1 heure)
    ExpiresByType text/html "access plus 1 hour"

    # CSS et JavaScript - cache long (1 an)
    ExpiresByType text/css "access plus 1 year"
    ExpiresByType application/javascript "access plus 1 year"

    # Images - cache long (1 an)
    ExpiresByType image/png "access plus 1 year"
    ExpiresByType image/jpeg "access plus 1 year"
    ExpiresByType image/gif "access plus 1 year"
    ExpiresByType image/svg+xml "access plus 1 year"
    ExpiresByType image/webp "access plus 1 year"
    ExpiresByType image/x-icon "access plus 1 year"

    # Fonts - cache long (1 an)
    ExpiresByType application/font-woff "access plus 1 year"
    ExpiresByType application/font-woff2 "access plus 1 year"
    ExpiresByType font/woff "access plus 1 year"
    ExpiresByType font/woff2 "access plus 1 year"

    # JSON - cache moyen (1 jour)
    ExpiresByType application/json "access plus 1 day"
</IfModule>

# -----------------------------------------
# Compression GZIP
# -----------------------------------------
<IfModule mod_deflate.c>
    AddOutputFilterByType DEFLATE text/html
    AddOutputFilterByType DEFLATE text/plain
    AddOutputFilterByType DEFLATE text/xml
    AddOutputFilterByType DEFLATE text/css
    AddOutputFilterByType DEFLATE application/javascript
    AddOutputFilterByType DEFLATE application/json
    AddOutputFilterByType DEFLATE image/svg+xml
</IfModule>

# -----------------------------------------
# Types MIME corrects
# -----------------------------------------
<IfModule mod_mime.c>
    AddType application/javascript .js
    AddType text/css .css
    AddType application/json .json
    AddType application/manifest+json .webmanifest
    AddType image/svg+xml .svg
    AddType image/webp .webp
    AddType font/woff .woff
    AddType font/woff2 .woff2
</IfModule>

# -----------------------------------------
# Headers de sécurité
# -----------------------------------------
<IfModule mod_headers.c>
    # Empêcher le sniffing MIME
    Header set X-Content-Type-Options "nosniff"

    # Protection contre le clickjacking
    Header set X-Frame-Options "SAMEORIGIN"

    # Protection XSS
    Header set X-XSS-Protection "1; mode=block"

    # Referrer policy
    Header set Referrer-Policy "strict-origin-when-cross-origin"
</IfModule>

# -----------------------------------------
# Interdire l'accès aux fichiers sensibles
# -----------------------------------------
<FilesMatch "(^\.htaccess|\.git|\.env|bower\.json|package\.json|firebase\.json)$">
    Order Allow,Deny
    Deny from all
</FilesMatch>
```

---

## 4. Configuration DNS

Dans l'espace client OVH : **Domaines** > `devfesttoulouse.fr` > **Zone DNS**

### Option A : Enregistrements A (recommandé)

| Type | Sous-domaine | Cible | TTL |
|------|--------------|-------|-----|
| A | 2017 | `IP_de_ton_hebergement` | 3600 |
| A | test.2017 | `IP_de_ton_hebergement` | 3600 |

> L'IP de l'hébergement se trouve dans : **Hébergement** > **Informations générales** > **IPv4**

### Option B : Enregistrements CNAME

| Type | Sous-domaine | Cible | TTL |
|------|--------------|-------|-----|
| CNAME | 2017 | `ton_hebergement.hosting.ovh.net.` | 3600 |
| CNAME | test.2017 | `ton_hebergement.hosting.ovh.net.` | 3600 |

> **Note** : Le point final après `.net.` est obligatoire pour les CNAME.

---

## 5. Configuration Multisite OVH

Dans l'espace client OVH : **Hébergement** > **Multisite**

### Ajouter le domaine principal

1. Cliquer sur **Ajouter un domaine ou sous-domaine**
2. Sélectionner `devfesttoulouse.fr`
3. Entrer le sous-domaine : `2017`
4. Dossier racine : `2017.devfesttoulouse.fr`
5. Cocher **SSL** (Let's Encrypt gratuit)
6. Valider

### Ajouter le domaine de test

1. Cliquer sur **Ajouter un domaine ou sous-domaine**
2. Sélectionner `devfesttoulouse.fr`
3. Entrer le sous-domaine : `test.2017`
4. Dossier racine : `2017.devfesttoulouse.fr` (même dossier)
5. Cocher **SSL**
6. Valider

> **Astuce** : Les deux domaines pointent vers le même dossier. Utiliser `test.2017` pour tester avant de mettre à jour `2017`.

---

## 6. Vérifications post-déploiement

### Attendre la propagation DNS

La propagation DNS peut prendre de 4 à 24 heures.

### Tester les URLs

```bash
# Test basique
curl -I https://2017.devfesttoulouse.fr
curl -I https://test.2017.devfesttoulouse.fr

# Test du routage SPA (doit retourner 200, pas 404)
curl -I https://2017.devfesttoulouse.fr/speakers/
curl -I https://2017.devfesttoulouse.fr/schedule/
curl -I https://2017.devfesttoulouse.fr/blog/

# Test HTTPS
curl -I http://2017.devfesttoulouse.fr
# Doit retourner 301 redirect vers https://
```

### Vérifier dans le navigateur

1. Ouvrir https://2017.devfesttoulouse.fr
2. Vérifier la console développeur (F12) pour les erreurs
3. Tester la navigation entre les pages
4. Vérifier que le Service Worker s'installe (onglet Application)

### Points d'attention

- **Service Worker** : Ne fonctionne qu'en HTTPS
- **Firebase** : Si conservé, ajouter les domaines autorisés dans la console Firebase
- **Cache navigateur** : Vider le cache ou utiliser le mode incognito pour tester

---

## 7. Se passer de Firebase (mode statique)

Pour un site d'archive d'un événement passé, Firebase peut être supprimé. Le site devient 100% statique.

### Utilisation actuelle de Firebase

| Fonctionnalité | Fichier | Peut être remplacé |
|----------------|---------|-------------------|
| Données (speakers, sessions, schedule, partners, team, videos, tickets) | `src/elements/app-data.html` | ✅ Oui - JSON statiques |
| Authentification Google | `src/hoverboard-app.html` | ✅ Oui - Supprimer |
| Ratings des sessions | `src/elements/session-details.html` | ✅ Oui - Désactiver |
| Sessions favorites utilisateur | `src/pages/schedule-page.html` | ✅ Oui - Supprimer |

### Données disponibles

Le fichier `data/export-data-firebase.json` contient toutes les données nécessaires.

### Étapes pour supprimer Firebase

#### Étape 1 : Créer les fichiers JSON statiques

Extraire les données de `data/export-data-firebase.json` vers des fichiers séparés :

```
data/
├── speakers.json      # Données des speakers
├── sessions.json      # Données des sessions
├── schedule.json      # Planning horaire
├── partners.json      # Sponsors/partenaires
├── team.json          # Équipe organisateurs
├── videos.json        # Vidéos des sessions
└── tickets.json       # Infos billetterie
```

#### Étape 2 : Modifier `src/elements/app-data.html`

Remplacer les composants Firebase par des requêtes AJAX :

```html
<!-- AVANT (Firebase) -->
<firebase-document
  app-name="{$ firebase.name $}"
  path="/speakers"
  data="{{_firebaseSpeakersRaw}}">
</firebase-document>

<app-indexeddb-mirror
  session="{$ firebase.indexedDbSession $}"
  key="speakers"
  data="{{_firebaseSpeakersRaw}}"
  persisted-data="{{_speakersRaw}}">
</app-indexeddb-mirror>

<!-- APRÈS (JSON statique) -->
<iron-ajax
  auto
  url="data/speakers.json"
  handle-as="json"
  last-response="{{_speakersRaw}}">
</iron-ajax>
```

Répéter pour : sessions, schedule, partners, team, videos, tickets.

#### Étape 3 : Modifier `src/hoverboard-app.html`

Supprimer les imports Firebase :

```html
<!-- SUPPRIMER ces lignes -->
<link rel="import" href="../bower_components/polymerfire/firebase-app.html">
<link rel="import" href="../bower_components/polymerfire/firebase-auth.html">
```

Supprimer le composant `<firebase-app>` :

```html
<!-- SUPPRIMER ce bloc -->
<firebase-app
  name="{$ firebase.name $}"
  auth-domain="{$ firebase.domain $}"
  database-url="{$ firebase.database $}"
  api-key="{$ firebase.api $}">
</firebase-app>
```

Supprimer `<firebase-auth>` et la logique d'authentification associée.

#### Étape 4 : Désactiver les ratings

Dans `data/hoverboard.config.json` :

```json
{
  "enableSessionsRating": false
}
```

#### Étape 5 : Nettoyer les dépendances (optionnel)

Dans `bower.json`, supprimer `polymerfire` si plus utilisé.

### Ce qu'on perd

| Fonctionnalité | Impact |
|----------------|--------|
| Connexion Google | Plus de login utilisateur |
| "Mon agenda" personnel | Plus de sauvegarde des sessions favorites |
| Notation des sessions | Plus de ratings |
| Mise à jour temps réel | Données figées |

### Ce qu'on garde

- ✅ Toutes les pages (accueil, speakers, schedule, blog, team, FAQ)
- ✅ Affichage complet du programme
- ✅ Profils des speakers
- ✅ Vidéos des sessions
- ✅ Blog et articles
- ✅ Mode offline (Service Worker)
- ✅ PWA (Add to homescreen)

### Avantages du mode statique

- **Pas de dépendance externe** : Le site fonctionne sans service tiers
- **Pas de coûts Firebase** : Aucun quota à gérer
- **Meilleure performance** : Pas de requêtes vers Firebase
- **Simplicité** : Hébergement statique simple
- **Pérennité** : Le site fonctionnera indéfiniment

---

## Résumé des commandes

```bash
# Build
npm install && bower install && npm run build

# Test local
npm run serve

# Upload (via FTP/SFTP)
# Contenu de build/ → /home/login/2017.devfesttoulouse.fr/
```

---

## Support

- Documentation OVH : https://docs.ovh.com/fr/hosting/
- Documentation Polymer : https://polymer-library.polymer-project.org/1.0/docs/devguide/feature-overview
