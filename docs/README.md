# Documentation - Kore Watchtower

Mises à jour automatiques des conteneurs Docker opt-in sur le VPS KORE.

Structure alignée sur la convention KORE.

| Dossier | Contenu | État |
|---|---|---|
| [01-context](01-context/) | Vision, glossaire | ébauche |
| [02-architecture](02-architecture/) | Flux pull → restart | à compléter |
| [03-design](03-design/) | ADRs | ✓ ADR-001 |
| [04-devops](04-devops/) | Install, ajout d'un conteneur opt-in | à compléter |
| [05-quality](05-quality/) | Surveillance, rollback | à compléter |
| [99-decisions](99-decisions/) | Roadmap | à compléter |

## Conventions

- VPS de référence : **Debian 12**.
- Chemins : `/opt/watchtower` (code).
- Pas de volume persistant nécessaire (état dans le socket Docker).
- Le fichier d'exemple d'env est [`/.env.prod.example`](../.env.prod.example).
