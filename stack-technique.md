# Stack technique — Kiwinet

> Document de référence des choix technologiques validés.
> Dernière mise à jour : 8 avril 2026 — Phase 2 validée : kiwinet-infra-vm (Ansible, Bookworm ARM64)

---

## Philosophie

Chaque choix technologique est motivé par trois critères :

- **Open source** : outils communautaires, auditables, sans dépendance propriétaire
- **Représentatif** : standards reconnus en entreprise, lisibles par un recruteur technique
- **Cohérent** : les outils s'intègrent naturellement entre eux, l'ensemble forme un tout logique

---

## Infrastructure physique

| Composant      | Détail                              |
|----------------|-------------------------------------|
| OS             | Debian GNU/Linux 13.3 (Trixie)      |
| Architecture   | ARM Cortex-A72 — AArch64 (2 vCPU)  |
| RAM            | 12 Go                               |
| Virtualisation | QEMU / VirtIO (Freebox Delta)       |
| Stockage       | VirtIO Disk                         |
| Connexion      | Fibre résidentielle, IP fixe        |
| Domaine        | `kiwinet.me` — DNS géré chez Bluehost |

> L'IP publique et les détails réseau sensibles sont documentés localement, hors repository.

---

## Architecture des repositories

Six repositories, organisés en deux familles de responsabilité.

| Repository | Rôle | Cycle de déploiement |
|---|---|---|
| `kiwinet-infra-vm` | Provisioning VM via Ansible | Init machine + évolutions système |
| `kiwinet-infra-cloud` | Infrastructure cloud via Terraform (Scaleway) | Init infra cloud + évolutions |
| `kiwinet-services` | Services applicatifs Docker Compose | À chaque ajout ou modification de service |
| `kiwinet-observability` | Supervision : disponibilité, métriques, logs | Rarement (configuration stable) |
| `kiwinet-web` | Site portfolio Astro + Nginx | À chaque commit (CI/CD automatique) |
| `kiwinet-docs` | Documentation centrale, ADRs, stack technique | Au fil des évolutions |

La justification de cette organisation est détaillée dans [ADR-001](./docs/adr/001-reorganisation-repositories.md).

**Workflow Git général :**
```
Laptop Debian (VS Code) → git push → GitHub → git pull → VM Debian
```

La VM est uniquement un serveur d'exécution. Aucun IDE n'y est installé.

---

## Architecture réseau et domaines

```
Internet
    │
    ▼
IP publique fixe (Freebox Delta)
    │
    ├── :80   → Freebox → VM → Traefik  (HTTP Challenge + redirection HTTPS)
    ├── :443  → Freebox → VM → Traefik
    │               │
    │               ├── kiwinet.me / www.kiwinet.me  → kiwinet-web (CI/CD)
    │               ├── traefik.kiwinet.me           → Dashboard Traefik (auth-basic)
    │               ├── plex.kiwinet.me              → Plex (container Docker)
    │               ├── hub.kiwinet.me               → Home Assistant (network_mode: host)
    │               ├── status.kiwinet.me            → Uptime Kuma
    │               └── grafana.kiwinet.me           → Grafana
    │
    ├── :22    → SSH (hors Docker)
    ├── :25565 → Minecraft (TCP passthrough Traefik)
    ├── :1883  → Mosquitto MQTT (LAN uniquement)
    └── UDP    → WireGuard VPN (accès réseau local, natif Freebox)
```

`freebox.kiwinet.me` : DNS actif, aucun port ouvert — leurre passif. L'accès à Freebox OS se fait exclusivement via le VPN WireGuard.

---

## Réseaux Docker

| Réseau | Type | Rôle |
|---|---|---|
| `proxy` | bridge externe | Créé par `kiwinet-services/traefik/`, partagé par tous les services exposés publiquement |
| `monitoring` | bridge interne | Isolé entre Prometheus, cAdvisor, Node Exporter, Loki, Promtail. Grafana est sur les deux réseaux. |

Traefik doit toujours démarrer en premier — il crée le réseau `proxy`.

---

## Ports UFW ouverts

| Port  | Protocole | Usage |
|-------|-----------|-------|
| 22    | TCP | SSH |
| 80    | TCP | HTTP Challenge + redirection HTTPS |
| 443   | TCP | HTTPS |
| 25565 | TCP | Minecraft (passthrough Traefik) |
| 32400 | TCP | Plex (accès direct clients distants) |
| 1883  | TCP | Mosquitto MQTT (LAN uniquement) |

---

## Gestion SSL

### Stratégie : HTTP Challenge (Let's Encrypt)

Un certificat wildcard (`*.kiwinet.me`) nécessiterait un DNS Challenge, qui requiert une API DNS chez le registrar. Bluehost n'en propose pas. Le HTTP Challenge (port 80 ouvert) implique un certificat par domaine — c'est le compromis retenu.

| Certificat | Domaine | Gestionnaire | Renouvellement |
|---|---|---|---|
| #1 | `kiwinet.me` + `www` | Traefik | Automatique |
| #2 | `traefik.kiwinet.me` | Traefik | Automatique |
| #3 | `plex.kiwinet.me` | Traefik | Automatique |
| #4 | `hub.kiwinet.me` | Traefik | Automatique |
| #5 | `status.kiwinet.me` | Traefik | Automatique |
| #6 | `grafana.kiwinet.me` | Traefik | Automatique |
| #7 | `freebox.kiwinet.me` | Certbot standalone | Manuel — 15/06/2026 |

Le certificat `freebox.kiwinet.me` est un cas particulier : la Freebox bloque les connexions en loopback, Traefik ne peut pas lui faire de proxy. Le certificat est généré par Certbot standalone (port 80 libéré temporairement) et importé manuellement dans l'interface Freebox.

---

## Stack par couche

### Reverse proxy — Traefik v3

Point d'entrée unique de la VM. Aucun service n'est exposé sans passer par Traefik.

**Pourquoi Traefik plutôt que Nginx ou Caddy ?**
- Découverte automatique des containers via labels Docker
- Gestion SSL Let's Encrypt intégrée
- TCP passthrough natif (Minecraft)
- Standard de facto dans les stacks Docker conteneurisées

**Points critiques :**

`dynamic.yml` : toutes les définitions (middlewares, routers, services) sous une unique section `http:`. Plusieurs sections provoquent des erreurs silencieuses.

Rechargement à chaud non fiable malgré `watch: true` — toujours redémarrer après modification :
```bash
docker restart traefik
```

Nettoyage `acme.json` après échec ACME (rate limit : 5 tentatives/heure/domaine) :
```bash
cd /opt/kiwinet-services/traefik && docker compose down
python3 -c "
import json
with open('acme.json', 'r') as f:
    data = json.load(f)
for resolver in data:
    data[resolver]['Certificates'] = [
        c for c in data[resolver].get('Certificates', [])
        if c.get('domain', {}).get('main') not in ['domaine-a-supprimer.kiwinet.me']
    ]
with open('acme.json', 'w') as f:
    json.dump(data, f, indent=2)
print('OK')
"
chmod 600 acme.json && docker compose up -d
```

Adressage depuis un container Docker vers un service en `network_mode: host` :
```yaml
# Ne fonctionne pas
url: "http://127.0.0.1:8123"

# Correct — gateway réseau proxy (vérifier : docker network inspect proxy | grep Gateway)
url: "http://172.18.0.1:8123"
```

**Middlewares disponibles (`dynamic.yml`) :**

| Middleware | Usage |
|---|---|
| `auth-basic@file` | Dashboard Traefik |
| `secure-headers@file` | Services publics |
| `rate-limit@file` | Endpoints publics |
| `ha-forwardproto@file` | Home Assistant (X-Forwarded-Proto) |

---

### Site principal — Astro + Nginx Alpine

- Astro : générateur statique, zéro JavaScript inutile, image Docker finale ~15 Mo
- Nginx Alpine : serveur de fichiers minimaliste dans le container
- Build multi-stage : Astro → Nginx Alpine
- Architecture ARM64 : build cross-compilé via QEMU + Buildx sur runner GitHub Actions x86_64

---

### CI/CD — GitHub Actions + SSH

Déclenché à chaque push sur `main` de `kiwinet-web` :

```
git push origin main
    ↓
GitHub Actions :
  ├── docker build (linux/arm64 via QEMU/Buildx)
  ├── docker push → ghcr.io/rookain-kiwi/kiwinet-web:latest + :<sha>
  └── SSH → VM → docker compose pull + up -d
```

Double tag systématique : `latest` (commodité) + `<sha>` (rollback possible vers n'importe quelle version).

**Convention commentaires GitHub Actions :**
- Aucun commentaire inline dans les valeurs scalaires (`tags:`, `image:`, `run:`, `name:`)
- Aucun commentaire dans les blocs multilignes (`|` ou `>`) — le `#` y est transmis comme donnée brute
- Commentaires sur la ligne précédant la clé concernée

---

### Services applicatifs

**Minecraft Java Edition**
- Image : `itzg/minecraft-server` (ARM64 natif)
- VANILLA 26.1, 4 Go RAM, 6 joueurs max, whitelist activée
- TCP passthrough Traefik sur port 25565
- Volume `minecraft-data` en `external: true` — données persistantes entre recreations
- RCON activé (port 25575, interne) : `docker exec -i minecraft rcon-cli --password <RCON_PASSWORD>`

**Plex Media Server**
- Image officielle `plexinc/pms-docker` (ARM64)
- Réseau `proxy`, labels Traefik directs
- Médias montés depuis le NAS Freebox via CIFS (`/etc/fstab`)
- Données persistées via bind mount sur `/var/lib/plexmediaserver`

**Home Assistant + Mosquitto**
- HA en `network_mode: host` — obligatoire pour la découverte mDNS (Google Cast, Chromecast, Nest)
- Route Traefik déclarée dans `dynamic.yml` (labels Docker impossibles en mode host)
- Mosquitto : broker MQTT local, port 1883 LAN uniquement

---

### Observabilité

Deux niveaux complémentaires, regroupés dans `kiwinet-observability` :

| | Uptime Kuma | Grafana (Prometheus + Loki) |
|---|---|---|
| Audience | Externe, public | Interne, ops |
| Question | "Mon service est-il en ligne ?" | "Pourquoi ça a planté ?" |
| Accès | `status.kiwinet.me` | `grafana.kiwinet.me` |

**Dashboards Grafana importés :**

| Dashboard | ID | Data source |
|---|---|---|
| Node Exporter Full | 1860 | Prometheus |
| cAdvisor métriques containers | 19792 | Prometheus |

Logs : consultés via **Explore → Loki**. Les dashboards communautaires Loki (13639, 15141) ont des variables orphelines (`DS_LOKI`) — Explore est l'interface recommandée.

**Points critiques monitoring :**
- Permissions Loki : `user: "0"`, volume sur `/loki` (pas `/tmp/loki`)
- cAdvisor ARM64 : flag `--disable_metrics` n'accepte pas `accelerator` — liste valide : `cpu_topology,cpuset,memory_numa,process,referenced_memory,resctrl,sched,tcp,udp`
- cAdvisor : `--housekeeping_interval=30s` (défaut 1s — source principale de charge CPU)

---

### Accès sécurisé — WireGuard VPN

WireGuard natif Freebox Delta — zéro dépendance externe.

```
Avant : Internet → freebox.kiwinet.me:<port> → Freebox OS  (exposé)
Après : Internet → port fermé → rien
        Internet → WireGuard UDP → tunnel VPN → réseau local → Freebox OS
```

Un peer par appareil client, clé révocable individuellement.

---

## Estimation RAM

| Service | RAM estimée |
|---|---|
| Traefik | ~40 Mo |
| Site principal (Nginx) | ~20 Mo |
| Uptime Kuma | ~200 Mo |
| Prometheus + Node Exporter | ~330 Mo |
| cAdvisor | ~50 Mo |
| Loki + Promtail | ~330 Mo |
| Grafana | ~330 Mo |
| Home Assistant | ~630 Mo |
| Mosquitto | ~3 Mo |
| Minecraft (4 Go alloués) | ~500 Mo idle |
| **Total stack** | **~3 Go** |

Marge confortable sur 12 Go avec la stack complète active.

---

## Tableau récapitulatif

| Rôle | Outil | Justification clé |
|---|---|---|
| Reverse proxy | Traefik v3 | Natif Docker, SSL auto, TCP passthrough |
| SSL (services) | Let's Encrypt via Traefik | Automatique, gratuit |
| SSL (Freebox) | Let's Encrypt via Certbot | Standalone, import manuel |
| Framework front | Astro | Statique, léger, image ~15 Mo |
| Serveur fichiers | Nginx Alpine | Minimaliste, multi-stage build |
| Registry Docker | GHCR | Cohérence écosystème GitHub |
| CI/CD | GitHub Actions + SSH | Auditable, secrets maîtrisés |
| Provisioning VM | Ansible | Idempotent, standard DevOps |
| Infrastructure cloud | Terraform + Scaleway | IaC, sandbox isolée |
| Serveur Minecraft | itzg/minecraft-server | ARM64, RCON intégré |
| Média | Plex (Docker) | Labels Traefik, ARM64 officiel |
| Domotique | Home Assistant | Open source, Google Cast/MQTT |
| Broker IoT | Mosquitto | MQTT léger, local |
| Disponibilité | Uptime Kuma | Page publique, alertes Discord |
| Métriques containers | cAdvisor | Métriques Docker temps réel |
| Métriques système | Node Exporter | CPU, RAM, disk, réseau |
| Agrégation métriques | Prometheus | Standard industrie |
| Logs | Loki + Promtail | Cohérence Grafana Labs |
| Dashboards | Grafana | Visualisation unifiée |
| Accès distant | WireGuard (natif Freebox) | Zéro exposition publique |
| Versioning | Git + GitHub | Traçabilité, IaC |

---

## Maintenance récurrente

| Action | Fréquence | Prochaine échéance |
|---|---|---|
| Renouvellement certificats Traefik | Automatique | — |
| Renouvellement certificat Freebox | 90 jours | 15/06/2026 |
| Rotation PAT GitHub (`kiwinet-ghcr-push`) | 90 jours | À noter à création |
| Mises à jour système VM | Mensuel | — |
| Rotation clés WireGuard | Si compromis | — |
| Passage Minecraft VANILLA → PAPER | Dès support 26.1 | À surveiller |

---

## Avancement

| Étape | Statut |
|---|---|
| Infrastructure VM opérationnelle | ✅ |
| Traefik + SSL | ✅ |
| CI/CD GitHub Actions | ✅ |
| Site portfolio FR | ✅ |
| Plex Media Server | ✅ |
| Uptime Kuma + alertes Discord | ✅ |
| Stack Prometheus + Grafana + Loki | ✅ |
| Home Assistant + Mosquitto | ✅ |
| Minecraft dockerisé | ✅ |
| WireGuard VPN | ✅ |
| Base documentaire `kiwinet-docs` (ADR-001) | ✅ |
| `kiwinet-infra` → `kiwinet-services` | ✅ |
| `kiwinet-observability` (fusion status + monitoring) | ✅ |
| `kiwinet-status` archivé | ✅ |
| `kiwinet-monitoring` archivé | ✅ |
| `kiwinet-infra-vm` (Ansible) | ✅ Validé (Bookworm ARM64, 8 avril 2026) |
| `kiwinet-infra-cloud` (Terraform + Scaleway) | À venir — Phase 3 |
| Renouvellement certificat Freebox | 15/06/2026 |

---

*Document maintenu à jour au fil des phases. Voir [ADR-001](./docs/adr/001-reorganisation-repositories.md) pour le détail des décisions de réorganisation.*
