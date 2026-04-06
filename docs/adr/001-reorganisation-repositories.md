# ADR-001 — Réorganisation des repositories et stratégie IaC

- **Date** : avril 2026
- **Statut** : accepté
- **Auteur** : Loïc Kergoat (Rookain)

---

## Contexte

Le projet Kiwinet couvre jusqu'ici deux des trois blocs de compétences du parcours Bachelor Administrateur Système DevOps (RNCP niveau 6) :

- **Bloc 2** : Conteneurisation et CI/CD — couvert par `kiwinet-infra` et `kiwinet-web`
- **Bloc 3** : Supervision — couvert par `kiwinet-status` et `kiwinet-monitoring`

Le **Bloc 1** (automatisation du déploiement d'infrastructure cloud) n'est pas encore couvert par le projet.

Par ailleurs, la structure des quatre repositories existants présente des limites :

- `kiwinet-infra` porte un nom générique qui ne reflète pas précisément son rôle (orchestration de services applicatifs via Docker Compose)
- `kiwinet-status` et `kiwinet-monitoring` partagent la même responsabilité fonctionnelle (supervision), le même cycle de déploiement (git pull manuel) et le même opérateur — leur séparation n'est pas justifiée
- L'introduction de l'IaC nécessite deux nouveaux repositories dont le nommage doit être cohérent avec l'existant et entre eux

Cette décision formalise la réorganisation complète de la structure des repositories en réponse à ces constats.

---

## Décisions

### 1. Renommage de `kiwinet-infra` en `kiwinet-services`

**Justification** : `kiwinet-infra` contient exclusivement la configuration Docker Compose des services applicatifs (Traefik, Plex, Minecraft, Home Assistant, Mosquitto) et les configurations Traefik associées. Son rôle est d'orchestrer des services, pas de provisionner une infrastructure. Le terme `services` reflète précisément ce périmètre.

Le terme `infra` est libéré intentionnellement pour être réutilisé dans le préfixe des nouveaux repositories IaC (voir décision 3).

### 2. Fusion de `kiwinet-status` et `kiwinet-monitoring` en `kiwinet-observability`

**Justification** : Les deux repositories partagent trois caractéristiques qui justifient une fusion :

- **Même responsabilité** : la supervision de l'infrastructure, avec ses trois piliers — disponibilité (Uptime Kuma), métriques (Prometheus, cAdvisor, Node Exporter, Grafana) et logs (Loki, Promtail)
- **Même cycle de déploiement** : déploiement par `git pull` manuel sur la VM, sans artifact CI/CD
- **Même opérateur** : aucune séparation d'équipe ou de périmètre de responsabilité

Le terme `observability` est le terme consacré dans les bonnes pratiques DevOps et cloud native (CNCF) pour désigner l'ensemble métriques + logs + disponibilité. Il est préféré à `monitoring` qui n'évoque que les métriques.

**Structure interne du repo fusionné** :

```
kiwinet-observability/
  uptime/        # Uptime Kuma (ex kiwinet-status)
  monitoring/    # Prometheus, cAdvisor, Node Exporter, Loki, Promtail, Grafana
  README.md
```

**Méthode de migration** : création d'un nouveau repository `kiwinet-observability`, migration du contenu des deux repos via `git subtree` pour préserver l'historique, puis archivage des repos sources sur GitHub.

### 3. Création de deux repositories IaC avec nommage cohérent

Deux nouveaux repositories sont créés pour couvrir le Bloc 1 de la formation et étendre Kiwinet vers l'IaC :

| Repository | Outil | Cible | Rôle |
|---|---|---|---|
| `kiwinet-infra-vm` | Ansible | VM Freebox Delta | Provisioning système : Docker, UFW, utilisateurs, montages, Node.js, certificats |
| `kiwinet-infra-cloud` | Terraform | VPS Scaleway | Infrastructure cloud : compute, réseau, DNS, déclenchement Ansible post-deploy |

**Justification du préfixe `infra-`** : ces deux repositories décrivent et automatisent une infrastructure, par opposition aux repositories qui font tourner des services sur une infrastructure existante. Le préfixe `infra-` marque cette appartenance à la couche IaC. Le suffixe `-vm` / `-cloud` distingue la cible de déploiement.

**Justification de la séparation en deux repos** : bien que partageant le même objectif (IaC), ces deux repositories ont des cycles de vie distincts. `kiwinet-infra-vm` est rejoué lors de chaque modification système de la VM. `kiwinet-infra-cloud` est rejoué lors de modifications de l'infrastructure Scaleway. Les mélanger dans un seul repo créerait une ambiguïté sur le périmètre de chaque opération.

**Justification du besoin cloud** : la VM résidentielle (Freebox Delta) présente des contraintes réelles pour l'expérimentation — connexion résidentielle instable, ressources partagées avec les services de production, IP résidentielle. Un VPS Scaleway dédié répond à un besoin identifié : disposer d'un environnement sandbox isolé pour tester des services sans risquer la production, et délester la VM principale.

### 4. Création de `kiwinet-docs`

Un repository de documentation centrale est créé pour héberger les ADR, le stack technique et la vision globale du projet. Les READMEs de chaque repository sont recentrés sur leur rôle opérationnel (prérequis, commandes, structure des fichiers) et pointent vers `kiwinet-docs` pour le contexte global.

---

## Structure cible

```
kiwinet-docs            # Documentation centrale, ADRs (ce repo)
kiwinet-services        # Docker Compose : Traefik, Plex, MC, HA, Mosquitto
kiwinet-observability   # Uptime Kuma + Prometheus/Grafana/Loki
kiwinet-web             # Site portfolio Astro + Nginx (CI/CD)
kiwinet-infra-vm        # Ansible : provisioning VM Freebox Delta
kiwinet-infra-cloud     # Terraform : infrastructure Scaleway
```

---

## Conséquences

### Immédiate — Zéro régression avant toute nouvelle brique

Avant de créer `kiwinet-infra-vm` et `kiwinet-infra-cloud`, la remise à plat de l'existant doit être complète et validée en production.

### Plan de migration séquencé

**Phase 0 — Base documentaire** *(cette étape)*
- Création de `kiwinet-docs` avec ADR-001 et stack-technique.md
- Mise à jour des READMEs des quatre repos existants (recentrage opérationnel)

**Phase 1 — Remise à plat de l'existant**
- Renommage de `kiwinet-infra` → `kiwinet-services` sur GitHub
- Création de `kiwinet-observability`, migration via `git subtree`, archivage des sources
- Validation : zéro régression sur les services de production
- Mise à jour documentaire au fil de chaque opération

**Phase 2 — `kiwinet-infra-vm`**
- Création du repository Ansible
- Écriture et test des playbooks (dry-run avant application sur VM de production)
- Validation complète avant passage en phase 3

**Phase 3 — `kiwinet-infra-cloud`**
- Création du repository Terraform
- Provisioning du VPS Scaleway
- Déploiement du service sandbox justifié
- Connexion avec `kiwinet-infra-vm` (Ansible déclenché post-deploy)

*Les phases 2 et 3 peuvent être menées en parallèle une fois la phase 1 validée.*

### Conventions de nommage établies

Ce document établit les conventions de nommage suivantes, applicables aux futurs repositories du projet :

- Préfixe `kiwinet-` systématique
- `kiwinet-infra-*` : repositories IaC (Ansible, Terraform)
- `kiwinet-*` sans préfixe secondaire : repositories runtime (services, supervision, site)

---

## Alternatives écartées

**Fusionner `kiwinet-infra-vm` et `kiwinet-infra-cloud` en un seul repo `kiwinet-infra`** : écarté car les cycles de vie sont distincts et les outils différents (Ansible vs Terraform). Un seul repo mélangerait deux périmètres d'opération, créant une ambiguïté sur ce qui est rejoué et quand.

**Nommer les repos IaC `kiwinet-iac-vm` et `kiwinet-iac-cloud`** : écarté au profit de `kiwinet-infra-*`. Le terme `infra` est plus immédiatement lisible pour un recruteur ou un pair, et cohérent avec la libération intentionnelle de l'appellation par le renommage de l'ancien `kiwinet-infra`.

**Garder `kiwinet-status` et `kiwinet-monitoring` séparés** : écarté — aucun des critères justifiant la séparation (cycle de déploiement distinct, équipe distincte, risque distinct) n'est présent.

**Nommer le repo fusionné `kiwinet-monitoring`** : écarté car `monitoring` ne couvre sémantiquement que les métriques, pas les logs ni la disponibilité. `observability` est le terme consacré pour l'ensemble des trois piliers.