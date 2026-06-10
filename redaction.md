# 🗂️ Projet — Serveur BookStack sur Raspberry Pi 4

> **Contexte :** Projet personnel réalisé dans le cadre du BTS SIO option SISR.  
> **Objectif :** Déployer un wiki personnel auto-hébergé, sécurisé et accessible depuis n'importe où dans le monde.

---

## Sommaire

1. [Présentation du projet](#présentation-du-projet)
2. [Matériel utilisé](#matériel-utilisé)
3. [Architecture technique](#architecture-technique)
4. [Étapes de réalisation](#étapes-de-réalisation)
   - [Étape 1 — Installation de l'OS](#étape-1--installation-de-los)
   - [Étape 2 — Connexion SSH et configuration réseau](#étape-2--connexion-ssh-et-configuration-réseau)
   - [Étape 3 — Nom de domaine dynamique (DuckDNS)](#étape-3--nom-de-domaine-dynamique-duckdns)
   - [Étape 4 — Déploiement des services avec Docker](#étape-4--déploiement-des-services-avec-docker)
   - [Étape 5 — Redirection de ports (NAT/PAT)](#étape-5--redirection-de-ports-natpat)
   - [Étape 6 — Sécurisation HTTPS avec Let's Encrypt](#étape-6--sécurisation-https-avec-lets-encrypt)
5. [Résultat final](#résultat-final)
6. [Compétences acquises](#compétences-acquises)
7. [Difficultés rencontrées](#difficultés-rencontrées)
8. [Pistes d'amélioration](#pistes-damélioration)

---

## Présentation du projet

L'objectif de ce projet était de transformer un **Raspberry Pi 4** en serveur web hébergeant une instance **BookStack** — un wiki open source permettant d'organiser de la documentation, des procédures et des notes.

**Pourquoi ce projet ?**

Un PC standard branché en permanence consomme trop d'électricité pour être envisagé comme serveur domestique. Les solutions cloud (VPS) existent, mais le choix a été fait d'héberger les données sur une machine physique personnelle, notamment pour des raisons de maîtrise des données et d'apprentissage concret de la configuration serveur. Un NAS aurait aussi été envisageable, mais plus coûteux et moins formateur qu'une configuration faite de zéro.

Le Raspberry Pi 4 (faible consommation, format compact, architecture ARM) est parfaitement adapté à ce cas d'usage.

---

## Matériel utilisé

| Composant | Rôle | Prix indicatif |
|---|---|---|
| Raspberry Pi 4 (4 Go de RAM) | Serveur principal | ~70 € |
| Carte microSD 32 Go (classe 10) | Stockage système et données | ~12 € |
| Câble alimentation USB-C (5V/3A) | Alimentation du Pi | ~10 € |
| Câble Ethernet | Connexion filaire stable à la box | ~5 € |
| PC sous Windows (machine principale) | Administration à distance via SSH | — |

---

## Architecture technique

```
Internet
    │
    ▼
Box (routeur) — Redirection ports 80/443
    │
    ▼
Raspberry Pi 4 (IP locale fixe)
    │
    └── Docker Compose
          ├── bookstack          (port 80 interne)
          ├── bookstack_db       (MariaDB)
          └── nginx-proxy-manager (ports 80, 443, 81)
                  │
                  └── Certificat SSL Let's Encrypt
                        → https://bookstack-clara.duckdns.org
```

> **→ Accessible en HTTPS depuis n'importe où, 24h/24.**

---

## Étapes de réalisation

### Étape 1 — Installation de l'OS

<!-- 📸 Insérer ici une capture de Raspberry Pi Imager -->

On utilise l'outil officiel **Raspberry Pi Imager** pour flasher la carte microSD depuis le PC. L'image choisie est **Raspberry Pi OS Lite (64-bit)** : sans interface graphique, pour dédier toutes les ressources du Pi au serveur.

Lors de l'écriture, on configure directement dans Raspberry Pi Imager :
- Le nom d'utilisateur et le mot de passe
- L'activation du **protocole SSH** (authentification par mot de passe)

> SSH est indispensable : le Pi étant utilisé sans écran ni clavier, c'est le seul moyen de le piloter à distance.

---

### Étape 2 — Connexion SSH et configuration réseau

<!-- 📸 Insérer ici une capture du terminal SSH connecté -->

Une fois le Pi démarré et branché en Ethernet, on récupère son adresse IP locale depuis l'interface de la box (section « Appareils connectés »).

**Connexion SSH depuis Windows :**

```bash
ssh pi@192.168.1.130
```

**Problème rencontré et solution :** Par défaut, la box attribue les adresses IP dynamiquement (DHCP). Si l'adresse du Pi change après un redémarrage, la connexion SSH et les redirections de ports deviennent invalides. Pour éviter ce problème, on crée un **bail DHCP statique** directement dans l'interface de la box : on associe l'adresse MAC du Pi à une IP fixe. Ainsi, le Pi retrouve toujours la même adresse, quelle que soit la durée de coupure.

| Sans réservation d'IP | Avec réservation d'IP |
|---|---|
| L'IP peut changer après redémarrage → SSH cassé, BookStack inaccessible | L'IP est toujours identique → configuration stable |

**Mise à jour du système :**

```bash
sudo apt update && sudo apt upgrade -y
```

---

### Étape 3 — Nom de domaine dynamique (DuckDNS)

<!-- 📸 Insérer ici une capture de l'interface DuckDNS -->

L'adresse IP publique fournie par l'opérateur change régulièrement. Pour accéder au serveur via une URL stable (ex : `https://bookstack-clara.duckdns.org`), on utilise le service **DuckDNS** — un DNS dynamique gratuit.

DuckDNS fait le lien entre le nom de domaine choisi et l'IP publique actuelle de la connexion internet. Le domaine est créé en quelques clics depuis [duckdns.org](https://www.duckdns.org).

---

### Étape 4 — Déploiement des services avec Docker

<!-- 📸 Insérer ici une capture de la sortie de `docker ps` -->

On installe **Docker** sur le Pi via le script officiel :

```bash
curl -sSL https://get.docker.com | sh
sudo usermod -aG docker $USER
```

Ensuite, on crée un fichier `docker-compose.yml` qui orchestre trois conteneurs :

| Conteneur | Rôle |
|---|---|
| `bookstack` | Application wiki (interface web) |
| `bookstack_db` | Base de données MariaDB |
| `nginx-proxy-manager` | Reverse proxy + gestion des certificats SSL |

**Extrait du `docker-compose.yml` :**

```yaml
services:
  bookstack:
    image: lscr.io/linuxserver/bookstack:latest
    environment:
      - APP_URL=https://bookstack-clara.duckdns.org
      - DB_HOST=bookstack_db
      - DB_USER=bookstack
      - DB_PASS=mot_de_passe_db
    volumes:
      - ./config:/config
    restart: unless-stopped
    depends_on:
      - bookstack_db

  bookstack_db:
    image: lscr.io/linuxserver/mariadb:latest
    environment:
      - MYSQL_DATABASE=bookstackapp
      - MYSQL_USER=bookstack
      - MYSQL_PASS=mot_de_passe_db
    volumes:
      - ./db_config:/config
    restart: unless-stopped

  nginx-proxy-manager:
    image: jc21/nginx-proxy-manager:latest
    ports:
      - '80:80'
      - '443:443'
      - '81:81'
    volumes:
      - ./npm/data:/data
      - ./npm/letsencrypt:/etc/letsencrypt
    restart: unless-stopped
```

**Démarrage de l'ensemble des services :**

```bash
docker compose up -d
```

La directive `restart: unless-stopped` garantit que les services redémarrent automatiquement en cas de coupure de courant.

---

### Étape 5 — Redirection de ports (NAT/PAT)

<!-- 📸 Insérer ici une capture de l'interface de la box (section redirections de ports) -->

Le serveur tourne sur le réseau local, mais reste invisible depuis Internet. On configure la box pour rediriger les connexions entrantes vers le Pi :

| Règle | Port externe | Port interne | Destination |
|---|---|---|---|
| Web-HTTP | 80 | 80 | IP locale du Pi |
| Web-HTTPS | 443 | 443 | IP locale du Pi |

> Le port 80 est nécessaire pour la validation des certificats Let's Encrypt. Le port 443 est le port du trafic HTTPS chiffré.

---

### Étape 6 — Sécurisation HTTPS avec Let's Encrypt

<!-- 📸 Insérer ici une capture de l'interface Nginx Proxy Manager -->

On accède à l'interface de Nginx Proxy Manager via `http://192.168.1.130:81` et on configure un **Proxy Host** :

- **Domain :** `bookstack-clara.duckdns.org`
- **Forward vers :** conteneur `bookstack`, port `80`
- **SSL :** génération d'un certificat via Let's Encrypt, avec redirection forcée HTTP → HTTPS

Le certificat est renouvelé automatiquement tous les 90 jours.

---

## Résultat final

<!-- 📸 Insérer ici une capture de BookStack affiché dans un navigateur avec le cadenas HTTPS -->

Le serveur BookStack est accessible en **HTTPS** depuis n'importe quelle connexion internet (4G, Wi-Fi extérieur, bureau) à l'adresse :

```
https://bookstack-clara.duckdns.org
```

Caractéristiques du déploiement final :

- ✅ **Accessible partout**, 24h/24 — 7j/7
- ✅ **Connexion chiffrée** HTTPS (certificat Let's Encrypt)
- ✅ **Redémarrage automatique** des conteneurs après coupure de courant
- ✅ **Données hébergées localement** sur machine physique personnelle
- ✅ **Coût d'exploitation nul** (hébergement gratuit, pas d'abonnement)

---

## Compétences acquises

### Administration système Linux

- Flash d'une image OS sur carte SD et premier démarrage headless
- Administration en ligne de commande : gestion des paquets (`apt`), navigation dans l'arborescence, édition de fichiers de configuration (`nano`)
- Gestion des permissions (`chmod`, `chown`)

### Conteneurisation avec Docker

- Installation de Docker sur architecture ARM
- Rédaction d'un fichier `docker-compose.yml` multi-services
- Gestion des volumes pour la persistance des données
- Suivi des conteneurs et consultation des logs (`docker ps`, `docker compose logs`)

### Réseau et services web

- Adressage IP et réservation de bail DHCP statique
- Configuration de redirections de ports (NAT/PAT) sur box opérateur
- DNS dynamique avec DuckDNS
- Configuration d'un reverse proxy (Nginx Proxy Manager)

### Sécurité

- Administration à distance via SSH
- Mise en place du chiffrement HTTPS avec certificat SSL/TLS (Let's Encrypt)
- Cloisonnement des services par conteneurs Docker

---

## Difficultés rencontrées

| Difficulté | Solution apportée |
|---|---|
| Adresse IP du Pi changeante après redémarrage | Réservation d'IP fixe (bail DHCP statique) dans l'interface de la box |
| Services inaccessibles depuis l'extérieur | Vérification et correction des règles NAT/PAT + mise à jour de DuckDNS |
| Erreur 500 au premier lancement de BookStack | Attente de l'initialisation complète de la base de données MariaDB |
| Confusion entre port 80 et 443 | Compréhension du rôle de chaque port : HTTP pour Let's Encrypt, HTTPS pour le trafic utilisateur |

---

## Pistes d'amélioration

- **Sauvegardes automatisées** : mise en place d'un script cron pour sauvegarder régulièrement la base de données SQL et les fichiers BookStack
- **Accès VPN (Tailscale)** : rendre le serveur accessible uniquement via VPN pour supprimer l'exposition publique tout en conservant l'accès distant
- **Monitoring** : ajout d'un outil de supervision (Uptime Kuma ou Netdata) pour surveiller la disponibilité du serveur
- **Migration vers SSD** : remplacement de la carte SD par un SSD USB pour améliorer la durabilité et les performances

---

*Projet réalisé dans le cadre du BTS SIO option SISR — Clara B.*
