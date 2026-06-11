# 🖥️ Serveur BookStack sur Raspberry Pi 4

> Wiki personnel auto-hébergé, accessible depuis n'importe où dans le monde — 24h/24, 7j/7.

## 📌 Présentation

Ce projet documente la mise en place d'un serveur **BookStack** sur un **Raspberry Pi 4**,
accessible publiquement via un nom de domaine sécurisé en HTTPS.

BookStack est un wiki open source qui permet d'organiser et de partager des procédures
techniques, des notes et de la documentation de manière structurée.

## 🛠️ Stack technique

| Composant | Rôle |
|---|---|
| Raspberry Pi 4 | Serveur physique |
| Docker | Gestionnaire de conteneurs |
| BookStack | Application wiki |
| MariaDB | Base de données |
| Nginx Proxy Manager | Reverse proxy + certificat SSL |
| DuckDNS | Nom de domaine dynamique (DDNS) |

## ✅ Prérequis

- Raspberry Pi 4 (2 Go RAM minimum)
- Carte microSD (32 Go minimum) avec Raspberry Pi OS Lite
- Connexion Internet avec accès à la box (redirection de ports)
- Un compte DuckDNS (gratuit)

## 📄 Documentation

👉 [Procédure d'installation complète](./procédure.md)

