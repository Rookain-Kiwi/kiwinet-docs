# ADR-003 — HTTPS sur le VPS Scaleway et consolidation de l'architecture

**Date :** avril 2026
**Statut :** Accepté
**Contexte :** Phase 4 — kiwinet-infra-ansible / kiwinet-services / kiwinet-web

---

## Contexte

L'ADR-002 documentait la migration de `kiwinet-web` vers le VPS Scaleway en HTTP uniquement,
en identifiant deux chantiers à traiter en Phase 4 :

1. **Nettoyage de la Freebox** : retrait de `kiwinet-web` du périmètre Traefik sur `kiwinet-services`
2. **HTTPS sur le VPS** : déploiement de Traefik + Let's Encrypt pour rétablir le certificat SSL

L'analyse des repos en Phase 4 a révélé que le point 1 ne nécessitait aucune action :
`kiwinet-web` n'a jamais été intégré dans `kiwinet-services` — les labels Traefik pour ce service
sont déclarés dans `kiwinet-web/docker-compose.vm.yml` (config Freebox) et n'ont jamais été
portés dans `kiwinet-services/traefik/`. L'architecture était déjà propre.

Le chantier Phase 4 se concentre donc entièrement sur le déploiement de Traefik sur le VPS.

---

## Décision

Déployer Traefik v3 sur le VPS Scaleway comme reverse proxy frontal, en remplacement de
l'exposition directe Nginx sur le port 80. Traefik prend en charge la terminaison TLS
via Let's Encrypt (HTTP Challenge) pour `kiwinet.me` et `www.kiwinet.me`.

Traefik est ajouté dans `kiwinet-services` sous la forme d'un Compose dédié VPS
(`traefik/docker-compose.vps.yml`), en parallèle du Compose Freebox existant
(`traefik/docker-compose.yml`). Le déploiement est automatisé via un nouveau rôle Ansible
`kiwinet-vps` dans `kiwinet-infra-ansible`.

---

## Justification

### Cohérence architecturale
Traefik est le point d'entrée de tous les services Kiwinet. Maintenir cette responsabilité
dans `kiwinet-services`, quel que soit l'hôte cible, préserve la séparation des couches :
`kiwinet-services` orchestre les services, `kiwinet-infra-ansible` configure les machines.
Intégrer Traefik dans `kiwinet-web` aurait couplé l'infrastructure réseau au code du site.

### Contrainte DNS Challenge
Bluehost ne propose pas d'API DNS. Le DNS Challenge Let's Encrypt est impossible.
Le HTTP Challenge (port 80 → validation ACME → certificat) est la seule option viable,
et le port 80 est déjà ouvert dans `firewall.tf` et UFW.

### Dual-target dans `kiwinet-services`
La coexistence de deux fichiers Compose pour Traefik (`docker-compose.yml` pour la Freebox,
`docker-compose.vps.yml` pour le VPS) suit le même pattern que `kiwinet-web`
(`docker-compose.yml` pour Scaleway, `docker-compose.vm.yml` pour la Freebox).
Ce pattern est déjà établi et documenté dans le projet.

---

## Périmètre des changements

| Repo | Fichier | Changement |
|---|---|---|
| `kiwinet-services` | `traefik/docker-compose.vps.yml` | Nouveau — Traefik VPS (réseau `proxy`, ports 80/443) |
| `kiwinet-services` | `traefik/traefik.vps.yml` | Nouveau — config statique Traefik VPS |
| `kiwinet-services` | `traefik/dynamic.vps.yml` | Nouveau — config dynamique VPS (router kiwinet.me) |
| `kiwinet-web` | `docker-compose.yml` | Mise à jour — Nginx rejoint le réseau `proxy`, suppression port 80 direct, ajout labels Traefik |
| `kiwinet-infra-ansible` | `roles/kiwinet-vps/` | Nouveau rôle — clone kiwinet-services, démarrage Traefik VPS |
| `kiwinet-infra-ansible` | `playbook-cloud.yml` | Ajout du rôle `kiwinet-vps` après `kiwinet-web` |
| `kiwinet-observability` | Uptime Kuma | Mise à jour monitor `kiwinet.me` : HTTP → HTTPS |
| `kiwinet-observability` | Grafana | Vérification dashboard (métriques container Traefik VPS) |

---

## Architecture cible (VPS Scaleway)

```
Internet
    │
    ▼ :80 / :443
┌─────────────────────┐
│   Traefik v3        │  HTTP Challenge Let's Encrypt
│   kiwinet-services  │  Certificat : kiwinet.me + www.kiwinet.me
└────────┬────────────┘
         │ réseau Docker proxy
         ▼ :80 (interne)
┌─────────────────────┐
│   Nginx Alpine      │  ghcr.io/rookain-kiwi/kiwinet-web
│   kiwinet-web       │  Aucune exposition de port directe
└─────────────────────┘
```

---

## Conséquences

### Immédiates
- Le `docker-compose.yml` de `kiwinet-web` (config VPS) perd son `ports: 80:80` direct
  et rejoint le réseau `proxy` avec les labels Traefik appropriés
- `acme.json` doit être créé manuellement sur le VPS (`chmod 600`) avant le premier démarrage
- La CI/CD `kiwinet-web` n'est pas impactée (elle déploie uniquement l'image, pas la config réseau)

### Monitoring
- Monitor Uptime Kuma `kiwinet.me` : passer de HTTP à HTTPS après validation du certificat
- Grafana : ajouter cAdvisor/Node Exporter du VPS si souhaité (hors périmètre Phase 4)

---

## Alternatives écartées

**Intégrer Traefik dans `kiwinet-web`** : écarté — couplage infra/app, contraire à la séparation
des responsabilités. Un jury DevOps identifierait ce couplage comme une dette technique.

**Certbot standalone** : écarté — Traefik gère le renouvellement automatique. Certbot nécessiterait
une gestion manuelle ou un cron, et n'est pas cohérent avec l'architecture reverse proxy déjà établie.

**Caddy** : écarté — Traefik est déjà en production sur la Freebox. Introduire un second outil
de reverse proxy sans justification technique serait contra-productif pour le portfolio.

**Wildcard Let's Encrypt** : écarté — requiert un DNS Challenge, impossible avec Bluehost.
