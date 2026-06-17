# ADR-006 — Déploiement de SearXNG comme méta-moteur de recherche

**Date :** 2026-06-17
**Statut :** Accepté
**Auteur :** Loïc Kergoat

---

## Contexte

Dans une démarche de souveraineté numérique progressive, l'utilisation de moteurs de recherche tiers (Google, Bing, DuckDuckGo) expose l'utilisateur à du profilage publicitaire et à la collecte de données comportementales.

L'objectif est de disposer d'un moteur de recherche personnel, accessible depuis n'importe quel appareil, qui interroge les sources existantes sans leur transmettre d'identité.

---

## Décision

Déploiement de **SearXNG** sur la VM Freebox Delta, exposé via Traefik sur `searxng.kiwinet.me`.

SearXNG est un méta-moteur open source (AGPL-3.0) qui agrège les résultats de plusieurs moteurs sans stocker de requêtes ni créer de profil utilisateur.

---

## Architecture retenue

```
Utilisateur → searxng.kiwinet.me (Traefik HTTPS)
                    ↓
              SearXNG (port 8080)
                    ↓
         Google / DuckDuckGo / Wikipedia / GitHub...
                    ↓
              Redis (cache, rate limiting)
```

**Réseaux Docker :**
- `proxy` (externe) : exposition Traefik
- `searxng` (interne) : communication SearXNG ↔ Redis isolée

---

## Paramètres clés

| Paramètre | Valeur | Justification |
|---|---|---|
| `limiter: true` | Activé | Protection contre l'abus et les blocages sources |
| `image_proxy: true` | Activé | Les images transitent par l'instance, pas depuis les sources |
| `enable_metrics: false` | Désactivé | Aucune collecte de métriques d'usage |
| `safe_search: 0` | Désactivé | Contrôle utilisateur |
| Bing | Désactivé | Éviter la dépendance Microsoft |
| `UWSGI_WORKERS=2` | 2 workers | Adapté aux ressources VM (3 vCPU) |

---

## Alternatives écartées

| Alternative | Raison du rejet |
|---|---|
| DuckDuckGo (direct) | Hébergé aux USA, soumis au Cloud Act |
| Brave Search | Indépendant mais tiers — données transitent ailleurs |
| Whoogle | Agrège Google uniquement — source unique |
| Startpage | Propriétaire, racheté par System1 (publicité) |

---

## Conséquences

**Positives :**
- Aucune requête de recherche ne quitte l'infrastructure avec une identité associée
- Accessible depuis tous les appareils (desktop, Android) via HTTPS
- Moteur configurable librement (sources, langue, thème)
- Cohérent avec la démarche de souveraineté numérique de Kiwinet

**Négatives / points de vigilance :**
- Les sources (Google, DDG) peuvent bloquer l'IP de la VM si les requêtes sont trop fréquentes — le rate limiting mitigue ce risque
- Instance publique : n'importe qui connaissant l'URL peut l'utiliser — acceptable en usage familial, à restreindre si nécessaire via middleware Traefik

---

## Évolutions possibles

- Ajout d'un middleware Traefik BasicAuth si l'instance devient trop exposée
- Activation de sources supplémentaires (Brave Search, Qwant, ArXiv...)
- Intégration comme moteur par défaut dans Firefox hardened (desktop) et IronFox (Android)

---

## Références

- [SearXNG GitHub](https://github.com/searxng/searxng)
- [Documentation officielle](https://docs.searxng.org/)
- ADR-001 à ADR-005 — architecture Kiwinet
