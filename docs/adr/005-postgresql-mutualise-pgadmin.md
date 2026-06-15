# ADR-005 — Base de données mutualisée PostgreSQL et interface PgAdmin

- **Date** : juin 2026
- **Statut** : accepté
- **Auteur** : Loïc Kergoat (Rookain)

---

## Contexte

Le déploiement de Synapse (serveur Matrix) nécessite une base de données relationnelle. SQLite est supporté par Synapse mais déconseillé en production : absence de concurrence, performances dégradées sous charge, pas de sauvegardes à chaud fiables.

Par ailleurs, d'autres services de la stack Kiwinet sont susceptibles de nécessiter une base relationnelle à terme (Grafana notamment, actuellement sur SQLite embarqué). Déployer une instance PostgreSQL par service serait inefficace en ressources et en maintenance.

Cette décision formalise le déploiement d'une instance PostgreSQL mutualisée sur la VM Freebox, accompagnée d'une interface d'administration PgAdmin, dans la perspective d'un premier usage par Synapse.

---

## Décisions

### 1. Instance PostgreSQL mutualisée

Un seul container PostgreSQL est déployé sur la VM Freebox. Il héberge plusieurs bases de données logiquement isolées, une par service consommateur.

Chaque service dispose de son propre utilisateur PostgreSQL avec des droits limités à sa base :

```sql
CREATE DATABASE synapse;
CREATE USER synapse WITH PASSWORD '...';
GRANT ALL PRIVILEGES ON DATABASE synapse TO synapse;
```

L'ajout d'un nouveau consommateur ne requiert ni redémarrage ni migration — uniquement la création d'une nouvelle base et d'un nouvel utilisateur.

### 2. Réseau Docker dédié `db`

Un réseau Docker `db` est créé en complément du réseau `proxy` existant. PostgreSQL n'est rattaché qu'au réseau `db` — il n'est jamais exposé via Traefik et aucun port n'est ouvert sur l'hôte.

Les services consommateurs (Synapse, futurs services) rejoignent le réseau `db` pour accéder à PostgreSQL, tout en restant sur le réseau `proxy` pour leur exposition via Traefik.

PgAdmin est le seul container présent sur les deux réseaux simultanément : `db` pour atteindre PostgreSQL, `proxy` pour être exposé via Traefik.

```
Réseau proxy  ← Traefik, services exposés publiquement
Réseau db     ← PostgreSQL, Synapse, PgAdmin
                (PostgreSQL : réseau db uniquement)
                (PgAdmin : db + proxy)
```

### 3. Interface d'administration PgAdmin

PgAdmin est déployé comme interface d'administration SQL. Il est exposé via Traefik sur `pgadmin.kiwinet.me`, protégé par authentification native PgAdmin.

PgAdmin permet l'administration complète des bases (requêtes SQL, gestion des schémas, utilisateurs, exports) sans nécessiter d'accès direct au container PostgreSQL.

### 4. Intégration dans `kiwinet-services`

PostgreSQL et PgAdmin sont intégrés dans le repository `kiwinet-services`, conformément à la convention un-compose-par-service. Deux fichiers distincts sont créés :

- `postgres/docker-compose.yml`
- `pgadmin/docker-compose.yml`

Le réseau `db` est déclaré comme `external: true` dans tous les composes consommateurs, à l'identique du réseau `proxy`.

### 5. Impact sur `kiwinet-infra-ansible`

Un nouveau rôle Ansible `db` est ajouté au playbook VM pour provisionner :

- Le réseau Docker `db`
- Les répertoires de volumes PostgreSQL (`/var/lib/postgresql/data`) et PgAdmin avec les permissions appropriées
- Les variables sensibles (credentials PostgreSQL) gérées via Ansible Vault

---

## Alternatives écartées

| Alternative | Raison du rejet |
|---|---|
| **SQLite par service** | Non viable en production pour Synapse ; pas mutualisable |
| **Une instance PostgreSQL par service** | Surcoût en RAM et maintenance sans bénéfice |
| **Adminer** | Interface minimaliste suffisante pour la consultation, mais PgAdmin offre des capacités d'administration avancées (plans d'exécution, gestion fine des rôles, exports) justifiant le surcoût |
| **Instance PostgreSQL sur VPS Scaleway** | Synapse est déployé sur la VM dans un premier temps ; la base suit le serveur applicatif |

---

## Axes d'évolution identifiés

- **Monitoring PostgreSQL via Grafana** : ajout de `postgres_exporter` dans `kiwinet-observability` pour exposer les métriques PostgreSQL (connexions actives, taille des bases, requêtes lentes) vers Prometheus, puis dashboards Grafana dédiés.
- **Migration vers le VPS Scaleway** : lors du passage de Synapse en fédération ouverte sur le VPS, PostgreSQL migre avec lui (export `pg_dump` + import, mise à jour DNS).
- **Sauvegardes externalisées** : intégration future dans la stratégie de backup Rclone identifiée comme axe d'amélioration global de l'infrastructure.

---

## Leçons retenues

- Le réseau Docker `db` applique le principe de moindre privilège : PostgreSQL n'est jamais joignable depuis l'extérieur, ni depuis des services qui n'en ont pas besoin.
- La mutualisation dès le premier service évite une dette technique (migration ultérieure d'une instance SQLite isolée vers du partagé).
- Les permissions des volumes PostgreSQL doivent être provisionnées par Ansible avant tout `docker compose up` — PostgreSQL refuse de démarrer si le répertoire de données appartient à root.
