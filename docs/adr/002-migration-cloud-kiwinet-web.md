# ADR-002 — Migration de kiwinet-web vers VPS Scaleway

**Date :** 14 avril 2026  
**Statut :** Accepté  
**Contexte :** Phase 3 — kiwinet-infra-cloud

---

## Contexte

Le site kiwinet.me était hébergé sur la VM résidentielle Freebox Delta (ARM64).
Des testeurs ont signalé des lenteurs dues à la contention de la connexion résidentielle
et à la latence variable d'une IP résidentielle.

Par ailleurs, la Phase 3 du projet vise à démontrer une compétence IaC cloud
(Terraform + provider Scaleway) en complément du provisioning VM Ansible (Phase 2).

---

## Décision

Migrer `kiwinet-web` (site Astro + Nginx Alpine) vers un VPS Scaleway STARDUST1-S
en `fr-par-1`, provisionné via Terraform et configuré via Ansible.

---

## Justification

- **Performance** : bande passante datacenter stable, latence réduite pour les visiteurs
- **Isolation** : le site ne partage plus les ressources avec Plex, Minecraft, Home Assistant
- **Portfolio** : démontre une chaîne IaC complète — Terraform (infra) + Ansible (config) + GitHub Actions (CI/CD)
- **Coût maîtrisé** : ~5 €/mois (STARDUST1-S + IP flexible)

---

## Périmètre de la migration

| Élément | Avant | Après |
|---|---|---|
| Hébergement | VM Freebox Delta (ARM64) | VPS Scaleway STARDUST1-S (x86_64) |
| Reverse proxy | Traefik (kiwinet-services) | Nginx direct port 80 |
| SSL | Let's Encrypt via Traefik | Non géré (HTTP uniquement — Phase 4) |
| CI/CD build | linux/arm64 + QEMU | linux/amd64 (natif) |
| DNS | `82.67.126.108` (Freebox) | `163.172.134.30` (Scaleway) |

---

## Conséquences

### Immédiates
- `kiwinet-web` retiré du périmètre Traefik sur la Freebox (Phase 4)
- CI/CD GitHub Actions mis à jour : suppression QEMU, architecture amd64
- `docker-compose.yml` devient la config Scaleway — `docker-compose.vm.yml` conservé comme fallback Freebox

### À traiter en Phase 4
- Déploiement Traefik sur le VPS pour gérer HTTPS (Let's Encrypt)
- Nettoyage `kiwinet-services` : suppression labels et config Traefik pour kiwinet-web
- Mise à jour monitors Uptime Kuma et dashboards Grafana

---

## Alternatives écartées

**Garder kiwinet-web sur la Freebox** : écarté — ne couvre pas le bloc IaC cloud du portfolio,
ne résout pas les problèmes de performance signalés.

**Héberger sur AWS/GCP/Azure** : écarté — Scaleway est suffisant pour ce périmètre,
français, RGPD, et cohérent avec l'orientation du projet.

**Wildcard DNS + Traefik dès la Phase 3** : écarté — Bluehost ne propose pas d'API DNS,
le DNS Challenge est impossible. HTTP uniquement en Phase 3, HTTPS délégué à la Phase 4.