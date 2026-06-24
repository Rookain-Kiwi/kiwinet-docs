# ADR-005 — Base de données mutualisée PostgreSQL et interface PgAdmin

- **Date** : juin 2026
- **Statut** : accepté
- **Auteur** : Loïc Kergoat (Rookain)

---

## Contexte

Le déploiement de Synapse (serveur Matrix) nécessite une base de données relationnelle. SQLite est supporté par Synapse mais déconseillé en production : absence de concurrence, performances dégradées sous charge, pas de sauvegardes à chaud fiables.

Par ailleurs, d'autres services de la stack Kiwinet sont susceptibles de nécessiter une base relationnelle à terme (Grafana notamment, actuellement sur SQLite embarqué). Déployer une instance PostgreSQL par service serait inefficace en ressources et en maintenance.

Cette décision formalise le déploiement d'une instance PostgreSQL mutualisée sur la VM Freebox, accompagnée d'une interface d'administration PgAdmin, dans la perspective d'un premier usage par Synapse — et, comme détaillé plus bas, par d'autres services candidats identifiés a posteriori.

---

## Décisions

### 1. Instance PostgreSQL mutualisée

Un seul container PostgreSQL est déployé sur la VM Freebox. Il héberge plusieurs bases de données logiquement isolées, une par service consommateur.

Chaque service dispose de son propre utilisateur PostgreSQL avec des droits limités à sa base :

```sql
CREATE DATABASE synapse;
CREATE USER synapse WITH PASSWORD '...';
GRANT ALL PRIVILEGES ON DATABASE synapse TO synapse;
GRANT ALL ON SCHEMA public TO synapse;
```

> **Note (PostgreSQL 15+)** : `GRANT ALL PRIVILEGES ON DATABASE` ne couvre plus automatiquement les droits sur le schéma `public` — `GRANT ALL ON SCHEMA public` doit systématiquement être exécuté après la création de chaque base, sous peine d'erreur `permission denied for schema public` au premier démarrage du service consommateur.

L'ajout d'un nouveau consommateur ne requiert ni redémarrage ni migration — uniquement la création d'une nouvelle base et d'un nouvel utilisateur.

### 2. Réseau Docker dédié `db`

Un réseau Docker `db` est créé en complément du réseau `proxy` existant. PostgreSQL n'est rattaché qu'au réseau `db` — il n'est jamais exposé via Traefik.

Les services consommateurs (Synapse, Grafana, futurs services) rejoignent le réseau `db` pour accéder à PostgreSQL, tout en restant sur le réseau `proxy` pour leur exposition via Traefik.

PgAdmin est le seul container présent sur les deux réseaux simultanément : `db` pour atteindre PostgreSQL, `proxy` pour être exposé via Traefik.

```
Réseau proxy  ← Traefik, services exposés publiquement
Réseau db     ← PostgreSQL, Synapse, Grafana, PgAdmin
                (PostgreSQL : réseau db + loopback hôte, voir section 6)
                (PgAdmin : db + proxy)
```

### 3. Interface d'administration PgAdmin

PgAdmin est déployé comme interface d'administration SQL. Il est exposé via Traefik sur `pgadmin.kiwinet.me`, protégé par authentification native PgAdmin.

PgAdmin permet l'administration complète des bases (requêtes SQL, gestion des schémas, utilisateurs, exports) sans nécessiter d'accès direct au container PostgreSQL.

**Limitation constatée** : PgAdmin encapsule les requêtes du Query Tool dans une transaction par défaut. `CREATE DATABASE` ne peut pas s'exécuter dans une transaction (`ERROR: CREATE DATABASE cannot run inside a transaction block`). La création de base se fait donc via `psql` en ligne de commande (`docker exec -it postgres psql`), ou via le mode auto-commit de PgAdmin.

### 4. Intégration dans `kiwinet-services`

PostgreSQL et PgAdmin sont intégrés dans le repository `kiwinet-services`, conformément à la convention un-compose-par-service. Deux fichiers distincts sont créés :

- `postgres/docker-compose.yml`
- `pgadmin/docker-compose.yml`

Le réseau `db` est déclaré comme `external: true` dans tous les composes consommateurs, à l'identique du réseau `proxy`.

### 5. Impact sur `kiwinet-infra-ansible`

Un nouveau rôle Ansible `db` a été ajouté au playbook VM pour provisionner :

- Le réseau Docker `db`
- Les répertoires de volumes PostgreSQL (`/var/lib/postgresql/data`, owner `999:999` — UID postgres dans l'image Alpine) et PgAdmin
- Vérification de présence des fichiers `.env` (warning si absent, pas de blocage)

### 6. Exposition loopback pour les services en `network_mode: host`

**Constat opérationnel** : certains services (Home Assistant) tournent en `network_mode: host` pour des raisons fonctionnelles propres (découverte mDNS pour Google Cast/Chromecast/Nest). Ces services ne sont rattachés à aucun réseau Docker bridge et ne peuvent donc pas résoudre le nom DNS `postgres` via le réseau `db`.

**Décision** : exposer PostgreSQL sur la loopback de l'hôte, en plus du réseau `db` :

```yaml
ports:
  - "127.0.0.1:5432:5432"
```

Le binding strict sur `127.0.0.1` exclut toute exposition LAN ou externe — seuls les processus tournant directement sur la VM (dont les containers en `network_mode: host`) peuvent y accéder. Les services sur réseau Docker bridge classique continuent d'utiliser le nom `postgres` via le réseau `db`, sans changement.

---

## Services migrés vers PostgreSQL mutualisé

| Service | Ancienne base | Statut migration | Méthode |
|---|---|---|---|
| **Synapse** | — (nouveau service) | Base créée, en attente de déploiement | Base + user dédiés |
| **Grafana** | SQLite embarqué (volume `grafana_data`) | **Migré** | Repart à zéro — aucune donnée historique conservée (choix délibéré, aucun attachement aux anciens dashboards) |
| **Komga** | SQLite embarqué | **Non migré — incompatible** | Komga ne supporte que SQLite nativement (vérifié via issues GitHub gotson/komga #1269 et #1327, demandes de fonctionnalité PostgreSQL non implémentées à ce jour). Reste sur SQLite, aucune action possible. |
| **Home Assistant** | SQLite (`home-assistant_v2.db`) | **Migré** | Intégration `recorder` native, `db_url` via `secrets.yaml`. Historique repart à zéro (choix délibéré) ; configuration, automatisations et entités non affectées — seul l'historique des états/capteurs est concerné par le changement de backend. Connexion via loopback hôte (voir section 6), HA tournant en `network_mode: host`. |

### Détail migration Grafana

```yaml
environment:
  GF_DATABASE_TYPE: postgres
  GF_DATABASE_HOST: postgres:5432
  GF_DATABASE_NAME: grafana
  GF_DATABASE_USER: grafana
  GF_DATABASE_PASSWORD: ${GRAFANA_DB_PASSWORD}
  GF_DATABASE_SSL_MODE: disable
networks:
  - proxy
  - monitoring
  - db
```

Dashboards restaurés post-migration via les ID communautaires officiels Grafana.com : Node Exporter Full (`1860`), cAdvisor (`19908`). Le dashboard Loki communautaire (`13639`) s'est révélé inadapté — conçu pour un contexte Kubernetes (labels `namespace`/`pod`/`app` absents d'une stack Docker Compose pure), il a été abandonné en faveur de l'exploration ad hoc via **Grafana Explore → Loki**.

### Détail migration Home Assistant

```yaml
# configuration.yaml
recorder:
  db_url: !secret recorder_db_url
  purge_keep_days: 14
```

```yaml
# secrets.yaml (gitignored)
recorder_db_url: "postgresql://homeassistant:PASSWORD@127.0.0.1:5432/homeassistant"
```

**Point d'attention credentials** : un mot de passe contenant des caractères réservés à la syntaxe URI (`@`, `:`, `/`, `?`, `#`) doit être percent-encodé dans `db_url` (ex. `@` → `%40`), sous peine d'ambiguïté dans le parsing de l'URL par SQLAlchemy. Les caractères hors de cette liste (`!`, `*`, etc.) n'ont pas besoin d'encodage.

Validation effectuée par observation de l'incrément du compteur de la table `states` sur deux lectures successives (confirmant une écriture active du recorder, et non une simple création de schéma au démarrage).

---

## Alternatives écartées

| Alternative | Raison du rejet |
|---|---|
| **SQLite par service** | Non viable en production pour Synapse ; pas mutualisable |
| **Une instance PostgreSQL par service** | Surcoût en RAM et maintenance sans bénéfice |
| **Adminer** | Interface minimaliste suffisante pour la consultation, mais PgAdmin offre des capacités d'administration avancées (plans d'exécution, gestion fine des rôles, exports) justifiant le surcoût |
| **Instance PostgreSQL sur VPS Scaleway** | Synapse est déployé sur la VM dans un premier temps ; la base suit le serveur applicatif |
| **Migration de l'historique Grafana/HA (pgloader ou export/import)** | Complexité et risque jugés disproportionnés par rapport à la valeur des données historiques concernées (dashboards par défaut, historique capteurs court terme) ; choix délibéré de repartir à zéro |
| **IP du bridge Docker pour HA → PostgreSQL** | Fragile (IP non garantie stable si le réseau est recréé) ; préféré l'exposition loopback, stable et documentée |

---

## Axes d'évolution identifiés

- **Monitoring PostgreSQL via Grafana** : ajout de `postgres_exporter` dans `kiwinet-observability` pour exposer les métriques PostgreSQL (connexions actives, taille des bases, requêtes lentes) vers Prometheus, puis dashboard Grafana dédié (ID `9628` ou équivalent). Reporté — pas de blocage fonctionnel actuel.
- **Migration vers le VPS Scaleway** : lors du passage de Synapse en fédération ouverte sur le VPS, PostgreSQL migre avec lui (export `pg_dump` + import, mise à jour DNS).
- **Sauvegardes externalisées** : intégration future dans la stratégie de backup Rclone identifiée comme axe d'amélioration global de l'infrastructure.

---

## Leçons retenues

- Le réseau Docker `db` applique le principe de moindre privilège : PostgreSQL n'est jamais joignable depuis l'extérieur, ni depuis des services qui n'en ont pas besoin. L'exception loopback (section 6) reste strictement bornée à l'hôte local.
- La mutualisation dès le premier service évite une dette technique (migration ultérieure d'une instance SQLite isolée vers du partagé).
- Les permissions des volumes PostgreSQL doivent être provisionnées par Ansible avant tout `docker compose up` — PostgreSQL refuse de démarrer si le répertoire de données appartient à root.
- **PostgreSQL 15+ change le comportement des droits par défaut sur le schéma `public`** — un `GRANT ALL ON SCHEMA public` explicite est désormais systématique après chaque `CREATE DATABASE`, sans quoi le service consommateur échoue ses migrations internes avec une erreur de permission peu explicite au premier abord.
- Tous les services ne sont pas migrables vers PostgreSQL : la compatibilité dépend du logiciel lui-même (Komga ne le supporte pas, malgré des demandes communautaires ouvertes depuis plusieurs années) — une vérification préalable de la documentation officielle évite un travail de migration entrepris à tort.
- Les services en `network_mode: host` cassent l'hypothèse implicite de résolution DNS via réseau Docker bridge — toute intégration future de ce type de service avec PostgreSQL mutualisé devra prévoir l'exposition loopback dès la conception, plutôt que la découvrir en cours de déploiement.
