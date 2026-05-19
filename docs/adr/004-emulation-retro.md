# ADR-004 — Exploration émulation rétro sur VM ARM64

**Date :** Mai 2026
**Statut :** Abandonné
**Décideurs :** Loïc Kergoat

---

## Contexte

Exploration d'une solution d'émulation rétro auto-hébergée, accessible depuis un navigateur, pour jouer à des jeux Amstrad CPC, Oric Atmos et Amiga 500/1200. L'objectif était de faire tourner RetroArch sur la VM Freebox Delta (ARM64, 3 vCPU, 12 Go RAM, sans GPU) avec accès web via streaming vidéo.

---

## Solutions évaluées

### 1. `linuxserver/retroarch` + Selkies/Wayland

Image officielle LinuxServer, support ARM64 natif, streaming via Selkies (WebSocket + H264 software).

**Résultat :** Échec. L'autostart labwc/Wayland ne lance pas RetroArch au démarrage du container sur cette architecture sans GPU. Cause probable : bug de l'image sur ARM64 sans `/dev/dri`. Temps passé à diagnostiquer sans résolution.

### 2. `kasmweb/retroarch:aarch64-1.14.0` + KasmVNC

Image Kasm Technologies, ARM64 natif, streaming via KasmVNC (X11 + Xfce4 + encodage H264 software).

**Résultat :** Fonctionnel mais non viable en production.

- RetroArch démarre ✅
- Core cap32 (Amstrad CPC) opérationnel ✅
- ROMs depuis NAS CIFS fonctionnelles ✅
- Émulation CPC correcte (Cybernoid, 5ème Axe testés) ✅
- **Charge CPU : 180-240% en permanence** (encodage x264 software sur ARM64 sans SIMD GPU) ❌
- **Sans connexion navigateur active : charge identique** — KasmVNC encode en continu ❌
- Incompatible avec la cohabitation Plex + Minecraft sur 3 vCPU ❌

---

## Cause racine

L'encodage vidéo H264 software sur ARM64 sans GPU est incompressible. Sur x86_64, l'encodeur x264 bénéficie d'instructions SIMD (SSE4, AVX2) qui réduisent drastiquement la charge. Sur ARM Cortex-A72, ces optimisations sont absentes ou limitées, rendant le streaming vidéo continu non viable sur une VM mutualisée.

Le problème n'est pas l'émulation elle-même — le CPC, l'Oric et l'Amiga tournent parfaitement en temps réel sur ce CPU. C'est exclusivement le streaming vidéo qui est incompatible avec les ressources disponibles.

---

## Décision

**Abandon de l'émulation via streaming vidéo sur cette infrastructure.**

Les ROMs sont conservées sur le NAS (`/mnt/Libraries/ROMs/`) pour une utilisation future éventuelle.

---

## Alternatives non explorées

| Alternative | Principe | Pourquoi non retenue |
|---|---|---|
| **EmulatorJS** | Émulation WebAssembly dans le navigateur — zéro charge serveur | Non explorée faute de temps — reste une piste viable |
| **Machine dédiée** | Raspberry Pi 4 + RetroArch natif, pas de streaming | Hors périmètre du projet |
| **GPU dédié** | Encodage matériel (VA-API, NVENC) | Non disponible sur VM Freebox Delta |

EmulatorJS reste la piste la plus prometteuse pour un usage futur : l'émulation tourne dans le navigateur du client, la VM ne sert que les fichiers statiques et les ROMs.

---

## Leçons retenues

- Le streaming vidéo H264 software sur ARM64 sans GPU est incompatible avec une VM mutualisée à 3 vCPU.
- Distinguer la charge d'émulation (négligeable pour systèmes 8/16 bits) de la charge de streaming (incompressible sans GPU).
- `docker restart` échoue sur ce kernel ARM sans AppArmor — toujours utiliser `docker compose up -d --force-recreate`.
- Les images KasmVNC encodent en continu même sans client connecté — prévoir un mécanisme d'arrêt si le service est déployé à nouveau.
- Le montage CIFS fonctionne correctement pour les ROMs (lecture séquentielle au chargement).
