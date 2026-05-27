# kore-watchtower

Mises à jour automatiques des conteneurs Docker sur le VPS KORE — **opt-in par label**.

## Rôle dans l'écosystème

| Brique | Responsabilité |
|---|---|
| [kore-traefik](https://github.com/alak8ba/kore-traefik) | Reverse proxy + TLS + sécurité transverse |
| **kore-watchtower** (ici) | Mise à jour auto des conteneurs opt-in |
| [kore-monitoring](https://github.com/alak8ba/kore-monitoring) | Prometheus + Grafana + Loki |
| [kore-backup](https://github.com/alak8ba/kore-backup) | Snapshots restic des volumes |

Chaque brique vit dans son propre dossier sur le VPS (`/opt/watchtower`, `/opt/traefik`, …) et se branche sur le réseau partagé `traefik-public` quand nécessaire.

## Principe

Watchtower scrute périodiquement les images Docker des conteneurs **explicitement opt-in** via le label :

```yaml
labels:
  - "com.centurylinklabs.watchtower.enable=true"
```

Les conteneurs sans ce label sont **ignorés**. C'est volontaire : on ne veut pas qu'un cache Redis ou une DB se fasse `pull` + `restart` sans contrôle.

## Démarrage

```shell
cp .env.prod.example .env.prod
# editer .env.prod (interval, notif Slack/email optionnelle)
docker compose --env-file .env.prod -f docker-compose.prod.yml up -d
```

## Configuration

Voir [.env.prod.example](.env.prod.example) pour les variables disponibles.

## Sécurité

- Le conteneur est en `read_only` + `no-new-privileges`.
- Il monte `/var/run/docker.sock` en **lecture seule**.
- Aucun port exposé sur l'hôte.
