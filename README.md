# Juice Shop – Pipeline DevSecOps (TP)

Ce dépôt est un fork d'[OWASP Juice Shop](https://github.com/juice-shop/juice-shop) utilisé comme support de TP pour mettre en place une chaîne CI/CD DevSecOps complète avec GitHub Actions, exécutée sur un runner self-hosted et déployée sur un serveur OVH.

Le TP répond à deux volets :

1. **Juice Shop** — chaîne CI/CD complète avec contrôles de sécurité automatisés (SAST, SCA, secrets, image, DAST).
2. **WrongSecrets** — gestion centralisée et rotation des secrets de l'application (clé JWT) avec OpenBao.

---

## 1. Chaîne CI/CD — `.github/workflows/pipeline.yml`

Déclenchée sur chaque `push` vers `main`/`master`, ou manuellement (`workflow_dispatch`). Une seule exécution à la fois par branche (les runs précédents sont annulés).

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

| # | Job          | Outil                                               | Rôle                                                                             |
| - | ------------ | --------------------------------------------------- | -------------------------------------------------------------------------------- |
| 1 | `secrets`    | [Gitleaks](https://github.com/gitleaks/gitleaks)    | Détection de secrets commités dans le code                                       |
| 2 | `sast`       | [Semgrep](https://semgrep.dev/)                     | Analyse statique du code (SAST)                                                  |
| 3 | `sca`        | [Trivy](https://aquasecurity.github.io/trivy/) (fs) | Analyse des dépendances (SCA), vulnérabilités HIGH/CRITICAL                      |
| 4 | `build`      | Docker Buildx                                       | Build de l'image et push vers GHCR (`ghcr.io`)                                   |
| 5 | `image-scan` | Trivy (image)                                       | Scan de vulnérabilités de l'image construite                                     |
| 6 | `deploy`     | Docker + OpenBao                                    | Déploiement du conteneur sur OVH, injection de la clé JWT depuis OpenBao (Vault) |
| 7 | `dast`       | [OWASP ZAP](https://www.zaproxy.org/) (baseline)    | Analyse dynamique (DAST) de l'application déployée                               |

### Réponse aux critères du sujet (Juice Shop)

**Chaîne CI/CD build → test → release → déploiement**
Le pipeline couvre le build (job `build`, image Docker), la release (push vers GHCR avec tag `sha-<commit>` + `latest`) et le déploiement (job `deploy` sur OVH). Il n'y a en revanche **pas de job de test applicatif** (unit/API/e2e) avant le build — voir [pistes d'amélioration](#pistes-damélioration--juice-shop).

**Analyse du code source (SAST)**
Job `sast` : Semgrep (`--config=auto`) scanne l'intégralité du dépôt à chaque run.

**Analyse des dépendances (SCA)**
Job `sca` : Trivy en mode `fs` scanne `package.json`/lockfiles à la recherche de CVE connues (sévérités HIGH/CRITICAL, `--ignore-unfixed`).

**Détection de secrets**
Job `secrets` : Gitleaks scanne tout l'historique du dépôt (`dir /repo`) avec redaction des correspondances (`--redact`).

**Analyse de l'image conteneurisée**
Job `image-scan` : Trivy en mode `image`, exécuté après le build, sur l'image réellement poussée vers GHCR (même tag que celui déployé).

**Déploiement automatique en environnement de test**
Job `deploy` : déploie le conteneur sur le serveur OVH (port `127.0.0.1:3003`), avec vérification de disponibilité (`Wait for readiness`, polling HTTP jusqu'à 40 tentatives).

**Analyse dynamique (DAST)**
Job `dast` : OWASP ZAP en mode `baseline` (`zap-baseline.py`) contre l'instance déployée à l'étape précédente.

**Génération et conservation des rapports**
Chaque job de sécurité (secrets, sast, sca, image-scan, dast) produit un rapport (SARIF pour les 4 premiers, HTML+JSON pour ZAP), qui est :

- archivé comme artefact GitHub Actions (`actions/upload-artifact`, téléchargeable depuis l'onglet *Actions* de chaque run) ;
- remonté dans l'onglet **Security** du dépôt (`github/codeql-action/upload-sarif`, en *best-effort* — `continue-on-error: true`).

### Gates configurables (`strict` / `observe`)

Chaque scanner est piloté par une variable d'environnement dédiée, qui détermine si un échec du scan **bloque** le pipeline (`strict`, `exit $RC`) ou se contente de **remonter un rapport** sans bloquer (`observe`, `exit 0`) :

```yaml
SECRETS_GATE: "observe"
SAST_GATE:    "observe"
SCA_GATE:     "observe"
IMAGE_GATE:   "observe"
DAST_GATE:    "observe"
```

Tous les gates sont actuellement en mode `observe` (audit sans blocage), un choix assumé le temps de stabiliser les scans sur Juice Shop (qui contient volontairement de nombreuses vulnérabilités et remonterait énormément de faux positifs/vrais positifs si les gates étaient `strict` dès maintenant). Le gate `secrets` a d'ailleurs été testé en `strict` puis repassé en `observe` (voir historique Git : `test(secrets): change gitleak to strict mode` / `test(secrets): change gitleak to observe mode`).

### Vulnérabilités identifiées / limites actuelles (Juice Shop)

- Juice Shop est **intentionnellement vulnérable** : les scans SAST/SCA/DAST remontent un grand nombre de findings by design (c'est l'objet même de l'application). Les gates en `observe` reflètent cet état : le pipeline documente les vulnérabilités sans encore les bloquer.
- Absence de job de **tests fonctionnels** (`npm test`, Cypress) dans la CI avant le build : rien ne garantit qu'une régression fonctionnelle soit détectée avant déploiement.
- Le déploiement se fait directement sur l'hôte Docker de production/démo (pas d'environnement de staging isolé) : le DAST s'exécute donc contre l'environnement réellement exposé.

### Pistes d'amélioration — Juice Shop

- Ajouter un job `test` (unit/API/e2e) entre `sast`/`sca` et `build`, bloquant en cas d'échec.
- Passer progressivement les gates en `strict`, en commençant par `SECRETS_GATE` (aucun secret ne devrait légitimement être commité) puis `IMAGE_GATE`/`SCA_GATE` une fois une baseline de vulnérabilités triée/acceptée.
- Isoler le déploiement `deploy`/`dast` dans un environnement de test dédié (staging) distinct de la démo publique.
- Ajouter une politique de rétention/format centralisé des rapports (ex. DefectDojo) plutôt que des artefacts épars par run.

---

## 2. Gestion des secrets — OpenBao (volet WrongSecrets)

### Secret identifié dans l'application

Juice Shop contient, **par construction**, une clé privée JWT RSA codée en dur dans [lib/insecurity.ts](lib/insecurity.ts) (challenge historique de l'application). Cette clé étant publique (visible dans le code source open-source), n'importe qui peut forger un jeton JWT admin valide : c'est le secret exposé que ce TP corrige.

```ts
// avant : clé privée en clair dans le code, servant de secret de signature JWT
const privateKey = '-----BEGIN RSA PRIVATE KEY-----\r\n...'
```

### Solution centralisée de gestion des secrets : OpenBao

[OpenBao](https://openbao.org/) (fork open-source de HashiCorp Vault) est déployé en dehors de ce dépôt, directement sur le serveur OVH (conteneur nommé `openbao`), et sert de coffre-fort centralisé. Le secret est stocké dans un moteur KV, sous le chemin `secret/juice-shop`, avec le champ `jwt_private_key_b64`.

Ce dépôt ne fait que **consommer** ce coffre via l'API `bao kv get/put`, avec deux tokens distincts et à portée restreinte (jamais le token root) :

| Token                        | Droit                    | Utilisé par                   |
| ---------------------------- | ------------------------ | ----------------------------- |
| `secrets.VAULT_TOKEN`        | lecture seule (`kv get`) | job `deploy` (`pipeline.yml`) |
| `secrets.VAULT_ROTATE_TOKEN` | écriture (`kv put`)      | job `rotate` (`rotate.yml`)   |

### Remplacement du secret en dur par un secret externalisé

Le code de [lib/insecurity.ts](lib/insecurity.ts) a été modifié pour lire la clé JWT depuis la variable d'environnement `JWT_PRIVATE_KEY` (injectée au déploiement depuis OpenBao), avec repli sur l'ancienne clé codée en dur uniquement si la variable est absente (usage local/dev) :

```ts
const hardcodedPrivateKey = '-----BEGIN RSA PRIVATE KEY-----\r\n...' // conservé comme fallback dev uniquement
const privateKey = process.env.JWT_PRIVATE_KEY
  ? Buffer.from(process.env.JWT_PRIVATE_KEY, 'base64').toString('utf8')
  : hardcodedPrivateKey
export const publicKey = fs
  ? crypto.createPublicKey(privateKey).export({ type: 'spki', format: 'pem' }).toString()
  : 'placeholder-public-key'
```

Point notable : la clé publique n'est plus lue depuis un fichier statique (`encryptionkeys/jwt.pub`) mais **dérivée cryptographiquement** de la clé privée courante (`crypto.createPublicKey`). Cela permet à la clé publique de suivre automatiquement chaque rotation de la clé privée, sans fichier à régénérer/redéployer séparément.

Flux au déploiement (job `deploy`, `pipeline.yml`) :

1. lecture du secret depuis OpenBao avec le token restreint : `bao kv get -field=jwt_private_key_b64 secret/juice-shop` ;
2. échec explicite si le secret est vide (`Secret vide -> abort`) plutôt qu'un déploiement silencieux avec une clé absente ;
3. masquage de la valeur dans les logs GitHub Actions (`::add-mask::`) ;
4. injection dans le conteneur via la variable d'environnement `JWT_PRIVATE_KEY` (jamais écrite sur disque ni dans un fichier de config versionné).

### Rotation automatique — `.github/workflows/rotate.yml`

Workflow planifié (`cron: '0 3 * * 1'`, tous les lundis à 3h) ou déclenchable manuellement (`workflow_dispatch`) :

1. génère une nouvelle paire de clés RSA (2048 bits) via `openssl genrsa` ;
2. stocke la nouvelle clé dans OpenBao (nouvelle version versionnée du secret, `bao kv put`) avec le token dédié à la rotation (`VAULT_ROTATE_TOKEN`) ;
3. redéploie le conteneur `juice-shop-app` avec la nouvelle clé injectée ;
4. attend que l'application réponde (`HTTP 200`) avant de terminer.

### Rapports générés et conservés (volet secrets)

- Le job `secrets` (Gitleaks) de `pipeline.yml` s'exécute à **chaque push**, produit un rapport SARIF archivé en artefact et remonté dans l'onglet Security — c'est le contrôle continu qui garantit qu'aucun nouveau secret ne revient dans le code (y compris la clé JWT elle-même, désormais détectable si elle était réintroduite).
- Les logs des jobs `deploy`/`rotate` ne contiennent jamais la valeur du secret en clair (masquage `::add-mask::` appliqué dès la lecture depuis OpenBao).

### Bonnes pratiques appliquées

- Séparation stricte lecture (déploiement) / écriture (rotation) des tokens OpenBao.
- Aucun secret présent en clair dans le dépôt Git ni dans les fichiers de configuration versionnés (`config/*.yml`).
- Secret transmis uniquement via variable d'environnement au runtime du conteneur, jamais écrit sur disque.
- Masquage systématique des valeurs sensibles dans les logs CI (`::add-mask::`).
- Rotation planifiée réduisant la fenêtre d'exposition en cas de fuite non détectée.

