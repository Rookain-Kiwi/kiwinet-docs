# Kiwinet — Documentation centrale

Kiwinet est une infrastructure DevOps auto-hébergée, conçue comme portfolio professionnel et homelab fonctionnel.
Ce repository est le point d'entrée documentaire de l'ensemble du projet.

## Contenu

| Fichier | Rôle |
|---|---|
| [`stack-technique.md`](./stack-technique.md) | État technique complet de l'infrastructure |
| [`docs/adr/`](./docs/adr/) | Décisions architecturales (ADR) |

## Repositories du projet

| Repository | Rôle | Type |
|---|---|---|
| [`kiwinet-services`](https://github.com/Rookain-Kiwi/kiwinet-services) | Services applicatifs Docker Compose | Runtime |
| [`kiwinet-observability`](https://github.com/Rookain-Kiwi/kiwinet-observability) | Supervision : disponibilité, métriques, logs | Runtime |
| [`kiwinet-web`](https://github.com/Rookain-Kiwi/kiwinet-web) | Site portfolio Astro + Nginx | CI/CD |
| [`kiwinet-infra-vm`](https://github.com/Rookain-Kiwi/kiwinet-infra-vm) | Provisioning VM via Ansible | IaC |
| [`kiwinet-infra-cloud`](https://github.com/Rookain-Kiwi/kiwinet-infra-cloud) | Infrastructure cloud via Terraform (Scaleway) | IaC |

> Le repo `kiwinet-infra-cloud` est en cours de création (voir [ADR-001](./docs/adr/001-reorganisation-repositories.md)).

## Liens

- Site : [kiwinet.me](https://kiwinet.me)
- Status : [status.kiwinet.me](https://status.kiwinet.me)
- GitHub : [Rookain-Kiwi](https://github.com/Rookain-Kiwi)
- LinkedIn : [loïc-kergoat-51150131a](https://www.linkedin.com/in/loïc-kergoat-51150131a)