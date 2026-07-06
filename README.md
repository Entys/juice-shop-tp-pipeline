# Juice Shop – Pipeline DevSecOps (TP)

Ce dépôt est un fork d'[OWASP Juice Shop](https://github.com/juice-shop/juice-shop) utilisé comme support de TP pour mettre en place une chaîne CI/CD DevSecOps complète avec GitHub Actions, exécutée sur un runner self-hosted et déployée sur un serveur OVH.

## Pipeline principal — `.github/workflows/pipeline.yml`

Déclenchée sur chaque `push` vers `main`/`master`, ou manuellement (`workflow_dispatch`). Une seule exécution à la fois par branche (les runs précédents sont annulés).

Le pipeline enchaîne 7 jobs sur un runner `[self-hosted, juice-pipeline]` :

| # | Job          | Outil                                               | Rôle                                                                             |
| - | ------------ | --------------------------------------------------- | -------------------------------------------------------------------------------- |
| 1 | `secrets`    | [Gitleaks](https://github.com/gitleaks/gitleaks)    | Détection de secrets commités dans le code                                       |
| 2 | `sast`       | [Semgrep](https://semgrep.dev/)                     | Analyse statique du code (SAST)                                                  |
| 3 | `sca`        | [Trivy](https://aquasecurity.github.io/trivy/) (fs) | Analyse des dépendances (SCA), vulnérabilités HIGH/CRITICAL                      |
| 4 | `build`      | Docker Buildx                                       | Build de l'image et push vers GHCR (`ghcr.io`)                                   |
| 5 | `image-scan` | Trivy (image)                                       | Scan de vulnérabilités de l'image construite                                     |
| 6 | `deploy`     | Docker + OpenBao                                    | Déploiement du conteneur sur OVH, injection de la clé JWT depuis OpenBao (Vault) |
| 7 | `dast`       | [OWASP ZAP](https://www.zaproxy.org/) (baseline)    | Analyse dynamique (DAST) de l'application déployée                               |

Chaque job de sécurité (1, 2, 3, 5, 7) génère un rapport **SARIF**, qui est :

- archivé comme artefact GitHub Actions (`actions/upload-artifact`) ;
- remonté dans l'onglet **Security** du dépôt (`github/codeql-action/upload-sarif`, best-effort).

### Gates configurables (`strict` / `observe`)

Chaque scanner est piloté par une variable d'environnement dédiée, qui détermine si un échec du scan **bloque** le pipeline (`strict`) ou se contente de **remonter un rapport** sans bloquer (`observe`) :

```yaml
SECRETS_GATE: "observe"
SAST_GATE:    "observe"
SCA_GATE:     "observe"
IMAGE_GATE:   "observe"
DAST_GATE:    "observe"
```

Actuellement tous les gates sont en mode `observe` (audit sans blocage) le temps de stabiliser les scans ; ils peuvent être passés en `strict` job par job pour les rendre bloquants.

### Gestion du secret JWT

Le job `deploy` ne stocke aucun secret en clair dans le repo : la clé privée JWT est lue depuis **OpenBao** (fork open-source de Vault) via un token CI à droits restreints (`secrets.VAULT_TOKEN`), jamais le token root, puis injectée dans le conteneur au runtime via une variable d'environnement.

## Rotation de la clé JWT — `.github/workflows/rotate.yml`

Workflow planifié (`cron: '0 3 * * 1'`, tous les lundis à 3h) ou déclenchable manuellement. Il :

1. génère une nouvelle paire de clés RSA (2048 bits) ;
2. la stocke dans OpenBao (nouvelle version du secret, via `VAULT_ROTATE_TOKEN`) ;
3. redéploie le conteneur `juice-shop-app` avec la nouvelle clé ;
4. attend que l'application réponde (`HTTP 200`) avant de terminer.

## Compression d'images — `.github/workflows/image_actions.yml`

Workflow hérité du projet Juice Shop upstream : compresse automatiquement les images (`.png`, `.jpg`, `.webp`, `.avif`) ajoutées via PR ou push sur `master`/`develop`, et ouvre une PR de compression si nécessaire. Sans lien avec la partie sécurité du TP.

## Schéma global

```text
push main/master
        │
        ▼
  ┌─────────────┬─────────────┬─────────────┐
  │  secrets     │   sast      │    sca      │   (parallèle)
  │ (Gitleaks)   │ (Semgrep)   │  (Trivy fs) │
  └─────────────┴─────────────┴─────────────┘
                       │
                       ▼
                  build & push (GHCR)
                       │
                       ▼
                image-scan (Trivy image)
                       │
                       ▼
              deploy (OVH + OpenBao)
                       │
                       ▼
              dast (OWASP ZAP baseline)
```
