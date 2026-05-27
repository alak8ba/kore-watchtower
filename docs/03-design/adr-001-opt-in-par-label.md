# ADR-001 — Watchtower opt-in par label, pas all-containers

- **Statut** : Accepté
- **Date** : 2026-05-27
- **Décideur** : opérateur VPS

## Contexte

Watchtower peut fonctionner selon deux modes :

1. **All-in** : surveille **tous** les conteneurs en marche, pull + restart automatique dès qu'une nouvelle image est disponible.
2. **Opt-in par label** : ne surveille **que** les conteneurs portant `com.centurylinklabs.watchtower.enable=true`. Les autres sont ignorés.

Le VPS KORE héberge un mix hétérogène :
- Apps applicatives (frontends, backends) → mises à jour fréquentes souhaitables.
- Bases de données (Postgres, Redis) → mises à jour **manuelles**, jamais auto (risque de breaking change majeur en migration).
- Outils infra (Traefik, Prometheus, Grafana) → mises à jour à étaler, jamais en lot.

## Décision

**Mode opt-in par label.** Variable `WATCHTOWER_LABEL_ENABLE=true` activée dans le compose.

Un conteneur n'est mis à jour automatiquement que si **explicitement** marqué par son mainteneur.

## Conséquences

### Positives

- **Sécurité par défaut** : un nouveau conteneur déployé sans label réfléchi n'est jamais touché. Pas de mauvaise surprise après un `git pull` d'un compose tiers.
- **Granularité fine** : on peut activer Watchtower sur le frontend d'une app sans toucher à sa DB.
- **Contrôle explicite des dépendances critiques** : DBs, message brokers, Traefik lui-même restent sous contrôle humain pour les versions majeures.
- **Auditabilité** : `grep -r "watchtower.enable" /opt/*/docker-compose*.yml` liste toutes les cibles d'auto-update. Pas d'effet de bord caché.

### Négatives

- **Charge de pensée** : chaque mainteneur d'app doit décider explicitement quoi activer.
- **Oublis possibles** : un nouveau service applicatif peut rester sans label et accumuler de la dette de mise à jour. Mitigation : audit régulier + checklist d'ajout d'app.

## Alternatives écartées

### Mode all-in

Écarté parce que :
- Le risque qu'un Postgres se fasse `pull postgres:16 → postgres:17` automatiquement (si tag `latest` mal pinné côté compose) est **inacceptable** : migrations destructrices possibles.
- Un cache Redis qui redémarre perd ses données chaudes → impact perf imprévisible.
- Traefik lui-même : une mise à jour avec breaking change (renommage de flags CLI) casse tout le reverse proxy d'un coup.

### Désactiver Watchtower entièrement

Écarté : on veut quand même **automatiser** les mises à jour des conteneurs explicitement candidats (frontends statiques, sidecars de monitoring, etc.) pour ne pas accumuler de retards de sécurité.

### kured / unattended-upgrades côté OS

Complémentaire, pas alternatif : ces outils mettent à jour **l'OS hôte**, pas les images Docker. Garder les deux niveaux.

## Convention d'utilisation

Dans le compose d'une app candidate :

```yaml
services:
  web:
    image: monorg/monapp:latest
    labels:
      - "com.centurylinklabs.watchtower.enable=true"
```

Bonnes pratiques :
- **Pinner la version** dans le tag (`monapp:1.4.x` plutôt que `latest`) pour borner le saut.
- **Activer uniquement sur les services stateless** (frontends, API stateless, sidecars).
- **Ne JAMAIS activer** sur : Postgres, MySQL, Redis persistant, MongoDB, Traefik, Vault, Consul.

## Quand reconsidérer

- Si on adopte un orchestrateur cluster (K8s) avec son propre cycle de mise à jour (Argo CD, Flux) → Watchtower devient redondant.
- Si on introduit un pipeline CI/CD qui push directement les nouvelles images via un hook → Watchtower devient moins pertinent (pull-based vs push-based).
