# Architecture

## Vue d'ensemble

```
┌────────────────────────────────────────────────────────────────────┐
│  VPS Debian (single host)                                          │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Watchtower                                                 │   │
│  │  - poll interval : ${WATCHTOWER_POLL_INTERVAL}              │   │
│  │  - label-only    : WATCHTOWER_LABEL_ENABLE=true             │   │
│  └─────────────┬───────────────────────────────────────────────┘   │
│                │ Docker API (socket RO)                            │
│                ▼                                                   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Docker daemon  (unix:///var/run/docker.sock)               │   │
│  └───────┬─────────────────────────────────────────────┬───────┘   │
│          │ inspect / pull / restart                    │           │
│          ▼                                             ▼           │
│  ┌──────────────────────┐                  ┌──────────────────┐   │
│  │  Conteneurs OPT-IN   │                  │ Conteneurs SANS  │   │
│  │  com.centurylinklabs │                  │ label opt-in     │   │
│  │  .watchtower.enable= │                  │  ─► IGNORÉS      │   │
│  │  "true"              │                  │                  │   │
│  │                      │                  │  Ex. Postgres,    │   │
│  │  Ex. frontends,      │                  │  Redis, Traefik,  │   │
│  │  APIs stateless,     │                  │  bases de données │   │
│  │  sidecars            │                  │                  │   │
│  └──────────┬───────────┘                  └──────────────────┘   │
│             │ pull image / restart si nouvelle digest             │
│             ▼                                                     │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  Notification (optionnelle)                              │    │
│  │  ${WATCHTOWER_NOTIFICATION_URL} → Slack / Discord / SMTP │    │
│  └──────────────────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────────────────────┘
```

## Composants et responsabilités

| Composant | Image | Port | Rôle |
|---|---|---|---|
| **Watchtower** | `containrrr/watchtower` | aucun | Poll Docker, pull images nouvelles, restart des conteneurs opt-in |

Une seule image, un seul conteneur. C'est la stack la plus simple de l'écosystème KORE.

## Flux d'une mise à jour

1. **Tick** : Watchtower réveille toutes les `WATCHTOWER_POLL_INTERVAL` secondes (défaut 3600 = 1h).
2. **Liste** : interroge Docker pour énumérer tous les conteneurs en marche.
3. **Filtre** : ne garde que ceux portant `com.centurylinklabs.watchtower.enable=true` (mode `WATCHTOWER_LABEL_ENABLE=true`).
4. **Inspect** : pour chaque conteneur retenu, récupère le digest de l'image courante.
5. **Pull** : tente un `docker pull` du tag (ex. `monapp:1.4`). Si le digest distant diffère du local → mise à jour disponible.
6. **Stop + Recreate** : si nouvelle image, arrête le conteneur, le supprime, et le recrée avec **les mêmes paramètres** (volumes, env, labels, network).
7. **Cleanup** : si `WATCHTOWER_CLEANUP=true`, supprime l'ancienne image dangling.
8. **Notify** : push le résumé (conteneurs mis à jour, erreurs) sur `WATCHTOWER_NOTIFICATION_URL` si configuré.

## Modes de fonctionnement (et notre choix)

| Mode | Description | Notre choix |
|---|---|---|
| **Label opt-in** | Seuls les conteneurs avec le label sont touchés | ✅ retenu — voir [ADR-001](../03-design/adr-001-opt-in-par-label.md) |
| All-containers | Tous les conteneurs en marche sont candidats | ❌ trop risqué (DBs, infra) |
| Monitor-only | Notifie sans agir | Non utilisé — viser kore-monitoring + Alertmanager pour ça |
| One-shot | Une seule passe puis quitte | Utile pour debug : `docker run --rm ... watchtower --run-once` |

## Sécurité

| Aspect | Choix |
|---|---|
| Socket Docker | Monté en **lecture seule** (`:ro`). Watchtower peut interroger + déclencher pull/restart via API, mais ne peut pas `chmod` le socket lui-même |
| `read_only` rootfs | Activé. Pas d'écriture sur le système de fichiers du conteneur |
| `no-new-privileges` | Activé. Empêche l'escalade de privilèges (setuid, capabilities) |
| Ressources | `mem_limit: 128m`, `cpus: 0.5` — Watchtower est très léger |
| Ports exposés | **Aucun**. Le service n'écoute sur rien |
| Réseau | Pas de réseau Docker custom nécessaire — interaction uniquement via socket |

Le risque résiduel principal : un attaquant qui compromet l'image `containrrr/watchtower` (chaîne d'approvisionnement) aurait accès à l'API Docker. Mitigations :
- Pinner l'image en production (`containrrr/watchtower:1.7.1` plutôt que `latest`).
- Audit du contrôle des labels : un attaquant qui peut écrire dans un `docker-compose.yml` peut ajouter le label opt-in à un conteneur sensible. C'est un vecteur **infra**, pas un vecteur Watchtower.

## Volumes persistants

**Aucun.** Watchtower est entièrement stateless. Toute la configuration vient des variables d'environnement et du socket Docker. Si on supprime le conteneur Watchtower et qu'on le recrée, tout repart sans perte.

## Réseau

Pas de réseau Docker requis. Watchtower communique avec le démon via `unix:///var/run/docker.sock` (monté en volume), pas via TCP/IP. Conséquence : pas besoin d'être sur `traefik-public` ni sur aucun réseau partagé.

## Dépendances entre briques KORE

| Brique | Rôle vis-à-vis de kore-watchtower |
|---|---|
| [kore-traefik](https://github.com/alak8ba/kore-traefik) | Pas de dépendance directe. Watchtower peut auto-update Traefik si label opt-in — **déconseillé** sur le reverse proxy critique |
| [kore-monitoring](https://github.com/alak8ba/kore-monitoring) | Watchtower n'expose pas de métriques Prometheus. Si surveillance souhaitée, parser ses logs via Loki + alerte sur `level=error` |
| [kore-backup](https://github.com/alak8ba/kore-backup) | Pas de volume à snapshotter |

## Liste recommandée des conteneurs opt-in

À **activer** sur :
- Frontends statiques (Next.js, Nuxt en build mode, sites Hugo/Astro).
- APIs stateless avec versions patch fréquentes.
- Sidecars infra (Promtail, node-exporter, cAdvisor — généralement stables).
- Grafana (les patchs sont sûrs ; les dashboards sont dans le volume persistant).

À **NE PAS activer** sur :
- Postgres, MySQL, MariaDB, MongoDB, CouchDB, autres DB persistantes.
- Redis persistant (si AOF/RDB activé).
- Traefik (changements de flags CLI possibles entre minor versions).
- Vault, Consul, etcd.
- Toute app dont la migration de version majeure nécessite des étapes manuelles.

## Quand cette architecture cesse de tenir

- **Adoption d'un orchestrateur cluster** (K8s + Argo CD/Flux) : Watchtower devient redondant — le contrôleur GitOps gère les updates.
- **Pipeline CI/CD push-based** : si chaque build pousse l'image puis déclenche un webhook de redéploiement, le polling devient inutile.
- **Multi-host** : Watchtower est mono-host. Pour plusieurs VPS, soit déployer Watchtower sur chaque, soit migrer vers un orchestrateur.
- **Conformité réglementaire** demandant un changelog signé par humain avant tout déploiement : désactiver Watchtower, passer en pull manuel.
