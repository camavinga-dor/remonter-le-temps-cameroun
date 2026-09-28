# Remonter le temps – Cameroun

Plateforme web de **comparaison d'époques** pour l'Institut National de Cartographie (INC) du Cameroun :
photographies aériennes, cartes anciennes et images satellites, affichées côte à côte pour observer
l'évolution du territoire (étalement urbain, couvert forestier, infrastructures).

Le projet s'inspire du service [« Remonter le temps »](https://remonterletemps.ign.fr/) de l'IGN (France) et suit
le cahier des charges technique de l'INC.

> Projet réalisé dans le cadre du cycle ingénieur en Génie Informatique et télécommunications (IUGET).

---

## Fonctionnalités

### Site public

- **Comparateur à curseur** : deux époques de part et d'autre d'une ligne que l'on fait glisser, ou superposées
  avec une opacité réglable.
- **Frise chronologique intelligente** : pour chaque côté, les époques disponibles **à l'endroit affiché** ;
  bascule photos / cartes ; jamais la même époque des deux côtés.
- **Recherche** : villes, coordonnées « lat, lng », et limites administratives (régions, départements…) avec
  affichage de leur contour.
- **Mesure** de distances et de surfaces, plusieurs figures à la fois, utilisable à la souris comme au doigt.
- **Export PNG / PDF** de la vue, avec échelle, boussole, source, mesures, points d'intérêt et contours.
- **Partage** par lien : position, époques et mode d'affichage sont conservés dans l'adresse.
- **Points d'intérêt historiques** et vue **« Données et couverture »** (emprise de chaque époque).
- **Téléchargement** des données publiées (GeoTIFF) et de leur fiche de métadonnées (PDF).
- **Bilingue** français / anglais, **thème sombre**, **adapté aux téléphones**.

### Administration (`/admin`)

- **Rôles** : Super Admin, Cartographe (dépose), Valideur (approuve et publie).
- **Couches raster** : dépôt de GeoTIFF, métadonnées, cycle brouillon → soumise → approuvée / refusée,
  publication automatique sur GeoServer, modification et dépublication.
- **Limites administratives** : dépôt d'un Shapefile (ou d'un zip), analyse automatique des colonnes, choix de
  la colonne du nom, publication en WFS.
- **Points d'intérêt**, **utilisateurs**, **journal d'audit** de toutes les actions.
- **Mot de passe oublié** : lien de réinitialisation à usage unique envoyé par e-mail.

### Outil de préparation d'images

`outils/images_yaounde.py` fabrique des images satellites en couleurs naturelles d'une ville, une par année
(Landsat depuis 1986, Sentinel-2 depuis 2016), prêtes à déposer. Seule la zone de la ville est téléchargée.

---

## Architecture

```
Navigateur ──► Frontend React (Vite)
                 ├── /api        ──► Backend Laravel ──► PostgreSQL + PostGIS
                 │                         │
                 │                         └── publie les couches ──┐
                 └── /geoserver  ─────────────────────────────────► GeoServer (tuiles TMS, WFS)
```

Le navigateur ne parle qu'à une seule adresse : le serveur du site relaie `/api` vers Laravel et
`/geoserver/gwc`, `/geoserver/wfs` vers GeoServer (Vite en développement, Nginx en production). L'interface
d'administration de GeoServer n'est jamais exposée.

| Couche | Technologies |
|---|---|
| Frontend | React 18, Vite 5, Leaflet 1.9, jsPDF |
| Backend | Laravel (PHP 8.2+), Laravel Sanctum |
| Base de données | PostgreSQL 16+ avec PostGIS |
| Serveur cartographique | GeoServer 2.28 (image Docker officielle) |
| Traitements géographiques | GDAL / OGR |

---

## Structure du dépôt

```
IGN-cameroun/
├── frontend/   Application React : site public et administration
├── backend/    Fichiers propres au projet Laravel (à copier sur un Laravel neuf, voir ci-dessous)
│   └── docker/ docker-compose de GeoServer
└── outils/     Script de préparation des images satellites
```

Le dossier `backend/` ne contient **que les fichiers du projet** (modèles, contrôleurs, migrations, services,
routes, configuration), pas une installation Laravel complète.

---

## Installation (développement)

Prérequis : Node.js 20+, PHP 8.2+ avec les extensions `pgsql` et `zip`, Composer, PostgreSQL + PostGIS,
GDAL, Docker. Sous Windows, le backend s'installe dans **WSL 2** (Ubuntu).

### 1. Frontend seul (mode démo, sans backend)

```bash
cd frontend
npm install
npm run dev          # http://localhost:5173
```

Sans configuration, le site affiche des couches de démonstration fabriquées dans le navigateur.

### 2. Base de données

```sql
CREATE USER rlt WITH PASSWORD 'choisir-un-mot-de-passe';
CREATE DATABASE rlt OWNER rlt;
\c rlt
CREATE EXTENSION postgis;
```

### 3. Backend

```bash
composer create-project laravel/laravel rlt-backend
cd rlt-backend
php artisan install:api                # routes d'API et Laravel Sanctum
cp -r ../backend/. .                   # fichiers du projet par-dessus le Laravel neuf
```

Reporter ensuite dans le `.env` de Laravel les variables de `backend/.env.example`, puis :

```bash
php artisan migrate --seed             # crée les tables et le premier compte Super Admin
php artisan serve                      # http://localhost:8000
```

Variables principales :

| Variable | Rôle |
|---|---|
| `DB_*` | Connexion PostgreSQL |
| `SEED_ADMIN_EMAIL`, `SEED_ADMIN_PASSWORD` | Premier compte Super Admin (mot de passe généré si vide) |
| `GEOSERVER_REST_URL`, `GEOSERVER_PASSWORD` | Accès de Laravel à l'API REST de GeoServer |
| `GEOSERVER_PUBLIC_URL=/geoserver` | Adresse de GeoServer vue par le navigateur (relative) |
| `FRONTEND_URL`, `MAIL_*` | Lien et envoi des e-mails « mot de passe oublié » |

### 4. GeoServer

```bash
cd backend/docker
GEOSERVER_PASSWORD=choisir-un-mot-de-passe docker compose up -d geoserver   # http://localhost:8080/geoserver
```

### 5. Relier le frontend au backend

```bash
echo "VITE_API_URL=/api" > frontend/.env
cd frontend && npm run dev
```

---

## Démonstration publique sans serveur

Pour montrer le site depuis Internet pendant qu'il tourne sur un poste :

```bash
cd frontend
npm run build && npm run preview                     # port 4173
cloudflared tunnel --url http://localhost:4173       # affiche une adresse https://….trycloudflare.com
```

Le site n'est accessible que tant que le poste, les services et le tunnel restent actifs.

---

## Données

Les images satellites proviennent de programmes publics, via le catalogue du Microsoft Planetary Computer :

- **Landsat** : « Landsat — courtesy of the U.S. Geological Survey » ;
- **Sentinel-2** : « Contient des données Copernicus Sentinel modifiées [année] ».

Les limites administratives utilisées pour les tests sont celles publiées par l'INC via OCHA sur
[Humanitarian Data Exchange](https://data.humdata.org/dataset/cod-ab-cmr) (licence CC BY-IGO).

Aucune donnée n'est versionnée dans ce dépôt : images, Shapefiles et fichiers `.env` sont exclus par
`.gitignore`.

---

## État du projet

| Élément | État |
|---|---|
| Fonctions du cahier des charges (F-01 à F-06) | Réalisées |
| Ordinateur et téléphone Android | Vérifiés |
| iPhone (Safari) | À vérifier |
| Tests automatisés | À faire |
| Déploiement de production (Nginx, HTTPS, sauvegardes) | À faire |

---

## Licence

À définir avec l'INC avant toute diffusion publique du code.
