# Vision

## Pourquoi kore-watchtower

Sur un VPS qui héberge plusieurs apps :
- les images Docker reçoivent des correctifs de sécurité régulièrement ;
- les versions patch (`1.4.x`) corrigent souvent des CVE sans breaking change ;
- l'opérateur n'a pas le temps de surveiller chaque registry à la main.

**kore-watchtower** automatise les `pull` + `restart` des conteneurs **explicitement marqués** comme candidats, et **ignore les autres**.

## Principe directeur

> **Opt-in par label, jamais par défaut.**

Un conteneur sans label `com.centurylinklabs.watchtower.enable=true` n'est jamais touché. Les bases de données, le reverse proxy, les composants infra restent sous contrôle manuel — c'est explicite, audité, prévisible.

## Hors scope

- **Mises à jour de l'OS hôte** (`unattended-upgrades`, `kured`) : couche complémentaire, traitée séparément.
- **Migrations de version majeure** (Postgres 16 → 17) : jamais automatisées — toujours manuelles avec dump préalable via kore-backup.
- **CI/CD push-based** : Watchtower est pull-based. Si un pipeline build + push directement les nouvelles images, Watchtower devient redondant.

## Voir aussi

- [ADR-001 — Opt-in par label](../03-design/adr-001-opt-in-par-label.md) — justification du mode et liste des services à NE jamais auto-update.
