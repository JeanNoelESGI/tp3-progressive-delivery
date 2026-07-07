# TP 3 — Progressive Delivery & SRE
## ESGI 5 SRC — Promotion 2025-2026

> **Auteur :** _Prénom NOM — votre matricule_
> **Date :** Juillet 2026
> **Cluster :** Kind 2 nœuds (`kind-pulse`) — Windows / Docker Desktop

---

## Table des matières

1. [Étape 0 — Outillage complémentaire](#étape-0--outillage-complémentaire)
2. [Étape 1 — Matrice SLI/SLO/Error Budget](#étape-1--matrice-slisloerror-budget)
3. [Étape 2 — Instrumentation & Buckets PromQL](#étape-2--instrumentation--buckets-promql)
4. [Étape 3 — Déploiement GitOps kube-prometheus-stack](#étape-3--déploiement-gitops-kube-prometheus-stack)
5. [Étape 4 — ServiceMonitor & Dashboard Grafana](#étape-4--servicemonitor--dashboard-grafana)
6. [Étape 5 — Migration vers Rollout Canary](#étape-5--migration-vers-rollout-canary)
7. [Étape 6 — Guide opérationnel des scénarios manuels](#étape-6--guide-opérationnel-des-scénarios-manuels)
8. [Étape 7 — AnalysisTemplate automatique](#étape-7--analysistemplate-automatique)
9. [Étape 8 — Stratégie Blue/Green pour Planning](#étape-8--stratégie-bluegreen-pour-planning)
10. [Étape 9 — Routage avancé par headers HTTP](#étape-9--routage-avancé-par-headers-http)
11. [Étape 10 — Alerting & Notifications](#étape-10--alerting--notifications)
12. [Étape 11 — Matrice comparative Argo Rollouts vs Flagger vs Native](#étape-11--matrice-comparative-argo-rollouts-vs-flagger-vs-native)
13. [Étape 12 — Synthèse d'Architecture & Rétrospective](#étape-12--synthèse-darchitecture--rétrospective)

---

## Étape 0 — Outillage complémentaire

### Objectif

Vérifier et compléter l'outillage CLI nécessaire au TP 3. En plus des outils du TP 2 (docker, kubectl, helm, kind, argocd), trois composants supplémentaires sont requis :

| Outil | Rôle dans le TP | Installation |
|---|---|---|
| `kubectl-argo-rollouts` | Plugin kubectl pour piloter les Rollouts (promote, abort, status) | Binaire GitHub releases |
| `promtool` | Valider syntaxiquement les règles PromQL/alerting avant push Git | Bundlé avec Prometheus |
| `amtool` | Tester la configuration Alertmanager (routes, receivers) | Bundlé avec Alertmanager |
| `jq` | Manipuler les sorties JSON de kubectl/curl (AnalysisRun, dashboards) | Package natif OS |

### Versions figées pour reproductibilité

```text
Outil                    Version minimale   Version utilisée (relevée)
─────────────────────────────────────────────────────────────────────
docker                   24.x               voir sortie `make tools-check-tp3`
kubectl                  1.29.x             voir sortie `make tools-check-tp3`
helm                     3.14.x             voir sortie `make tools-check-tp3`
kind                     0.23.x             voir sortie `make tools-check-tp3`
argocd CLI               2.11.x             voir sortie `make tools-check-tp3`
kubectl-argo-rollouts    1.7.x              voir sortie `make tools-check-tp3`
promtool                 2.52.x             voir sortie `make tools-check-tp3`
amtool                   0.27.x             voir sortie `make tools-check-tp3`
jq                       1.7.x              voir sortie `make tools-check-tp3`
```

> **Note Windows :** Les outils doivent être disponibles dans le PATH de Git Bash / WSL2 utilisé
> pour exécuter le Makefile. `amtool` est distribué dans l'archive Alertmanager officielle.
> `promtool` est distribué dans l'archive Prometheus officielle.

### Commandes d'installation rapide

#### kubectl-argo-rollouts (plugin kubectl)

```bash
# Linux / macOS
curl -fsSL \
  https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64 \
  -o /usr/local/bin/kubectl-argo-rollouts
chmod +x /usr/local/bin/kubectl-argo-rollouts

# Windows (PowerShell — placer dans un répertoire du PATH)
$url = "https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-windows-amd64"
Invoke-WebRequest -Uri $url -OutFile "$env:LOCALAPPDATA\Microsoft\WindowsApps\kubectl-argo-rollouts.exe"

# Vérification
kubectl argo rollouts version
```

#### promtool & amtool

```bash
# Télécharger l'archive Prometheus (contient promtool)
PROM_VER=2.52.0
curl -fsSL https://github.com/prometheus/prometheus/releases/download/v${PROM_VER}/prometheus-${PROM_VER}.linux-amd64.tar.gz \
  | tar xz --strip-components=1 -C /usr/local/bin \
      prometheus-${PROM_VER}.linux-amd64/promtool

# Télécharger l'archive Alertmanager (contient amtool)
AM_VER=0.27.0
curl -fsSL https://github.com/prometheus/alertmanager/releases/download/v${AM_VER}/alertmanager-${AM_VER}.linux-amd64.tar.gz \
  | tar xz --strip-components=1 -C /usr/local/bin \
      alertmanager-${AM_VER}.linux-amd64/amtool

# Vérifications
promtool --version
amtool --version
```

#### jq

```bash
# Ubuntu/Debian
sudo apt-get install -y jq

# macOS
brew install jq

# Windows (winget)
winget install jqlang.jq

# Vérification
jq --version
```

### Vérification complète : `make tools-check-tp3`

Lancer depuis le répertoire `pulse-campus/` :

```bash
cd pulse-campus
make tools-check-tp3
```

**Sortie attendue (exemple) :**

```text
[OK] docker                  : 27.3.1
[OK] kubectl                 : v1.30.2
[OK] helm                    : v3.15.2+g1a500d5
[OK] kind                    : kind v0.24.0 go1.22.6 linux/amd64
[OK] argocd                  : argocd: v2.11.3+3f344d5
[OK] kubectl-argo-rollouts   : argo-rollouts: v1.7.2+2b13ef2
[OK] promtool                : promtool, version 2.52.0
[OK] amtool                  : amtool, version 0.27.0
[OK] jq                      : jq-1.7.1
──────────────────────────────────────────────────────
Tous les outils requis sont présents. TP 3 prêt.
```

Si un outil est absent, la cible retourne `exit 1` et affiche `[KO]` avec un lien d'installation.

### Analyse des choix

**Pourquoi `promtool` avant de pousser en Git ?**
Les erreurs de syntaxe PromQL dans un `PrometheusRule` ne sont pas détectées au niveau Kubernetes
(le CRD accepte le YAML syntaxiquement valide) — elles n'apparaissent qu'à l'évaluation dans Prometheus.
`promtool check rules <fichier>` détecte ces erreurs localement, sans cluster.

**Pourquoi `amtool` ?**
`amtool config routes test` permet de simuler quelles routes et quels receivers une alerte déclencherait,
sans redémarrer Alertmanager. Essentiel pour valider les matrices de routage gradées (page/ticket) de l'étape 10.

**Pourquoi `jq` ?**
Plusieurs commandes du TP parsent la sortie JSON des `AnalysisRun` et des webhooks de notification.
`jq` est utilisé dans les scripts de vérification du Makefile.

---

<!-- ================================================================== -->
<!-- Les sections suivantes seront complétées au fil des étapes          -->
<!-- ================================================================== -->

## Étape 1 — Matrice SLI/SLO/Error Budget

### 1.1 Définitions préalables

| Terme | Définition opérationnelle dans le TP |
|---|---|
| **SLI** (Service Level Indicator) | Métrique quantitative mesurée par Prometheus — ex. taux de succès HTTP |
| **SLO** (Service Level Objective) | Cible de fiabilité que l'équipe s'engage à tenir sur une fenêtre de 30 jours |
| **Error Budget** | Temps/volume d'erreurs tolérées = `(1 − SLO_target) × window` |
| **Burn Rate** | Vitesse à laquelle le budget est consommé ; BR = 1 → budget épuisé exactement en 30 j |

> **Choix de la fenêtre :** 30 jours glissants (`[30d]` en PromQL) sont utilisés pour aligner l'error budget sur un cycle mensuel, ce qui correspond aux sprints et revues de production. Une fenêtre trop courte (7 j) augmente les faux positifs ; trop longue (90 j) amortit les incidents.

---

### 1.2 Matrice SLI/SLO/Error Budget par service

#### Contexte des services

| Service | Langage | Fonction métier | Criticité |
|---|---|---|---|
| **annuaire** | Node.js / Express | CRUD étudiants — consulté en continu par l'UI | 🔴 HAUTE |
| **planning** | Python / FastAPI | Gestion créneaux salles — usage intensif agenda | 🔴 HAUTE |
| **notif** | Go / net/http | Notifications événements — asynchrone, tolérant | 🟡 MOYENNE |

---

#### Service `annuaire` — SLO Disponibilité 99,9 %

| Dimension | Valeur |
|---|---|
| **SLI — Disponibilité** | Proportion de requêtes HTTP retournant un code non-5xx (hors probes `/healthz`, `/readyz`, `/metrics`) |
| **SLO cible** | **99,9 %** sur fenêtre glissante 30 jours |
| **Error Budget mensuel** | `30 × 24 × 60 × (1 − 0,999)` = **43,2 minutes** |
| **Seuil alerte fast burn** | Burn rate ≥ 14,4 — consomme 2 % du budget en 1 h → **PAGE** (réveil d'astreinte) |
| **Seuil alerte slow burn** | Burn rate ≥ 6,0 — consomme 5 % du budget en 6 h → **TICKET** (P2) |
| **SLI — Latence p95** | 95 % des requêtes répondues en < **300 ms** |
| **Justification seuil** | Expérience UX : au-delà de 300 ms, l'utilisateur perçoit une lenteur sur une liste d'étudiants |

#### Service `planning` — SLO Disponibilité 99,5 %

| Dimension | Valeur |
|---|---|
| **SLI — Disponibilité** | Proportion de requêtes HTTP retournant un code non-5xx |
| **SLO cible** | **99,5 %** sur fenêtre glissante 30 jours |
| **Error Budget mensuel** | `30 × 24 × 60 × (1 − 0,995)` = **216 minutes** (3 h 36) |
| **Seuil alerte fast burn** | Burn rate ≥ 14,4 → **PAGE** |
| **Seuil alerte slow burn** | Burn rate ≥ 6,0 → **TICKET** |
| **SLI — Latence p95** | 95 % des requêtes répondues en < **500 ms** |
| **Justification seuil** | L'overhead ASGI de FastAPI + éventuelle logique de slot-check justifient un seuil 500 ms vs 300 ms pour annuaire |

#### Service `notif` — SLO Disponibilité 99,5 %

| Dimension | Valeur |
|---|---|
| **SLI — Disponibilité** | Proportion de requêtes HTTP retournant un code non-5xx |
| **SLO cible** | **99,5 %** sur fenêtre glissante 30 jours |
| **Error Budget mensuel** | **216 minutes** (3 h 36) |
| **Seuil alerte fast burn** | Burn rate ≥ 14,4 → **PAGE** |
| **Seuil alerte slow burn** | Burn rate ≥ 6,0 → **TICKET** |
| **SLI — Latence p95** | 95 % des requêtes répondues en < **200 ms** |
| **Justification seuil** | Go `net/http` est le runtime le plus léger des 3 — le SLO latence peut être plus strict |

---

### 1.3 Calcul de l'Error Budget — Formule de référence

```
Error Budget (secondes) = window_secondes × (1 - slo_fraction)

annuaire  : 2 592 000 × (1 - 0.999) =  2 592 s  ≈  43 min 12 s
planning  : 2 592 000 × (1 - 0.995) = 12 960 s  ≈ 3 h 36 min
notif     : 2 592 000 × (1 - 0.995) = 12 960 s  ≈ 3 h 36 min
```

**Burn rate et alerting multi-fenêtre (méthode Google SRE)** :

```
burn_rate = taux_erreur_observé / (1 - slo_fraction)

Exemple annuaire (SLO 99,9 % → error_rate_budget = 0,001) :
  Si taux d'erreur = 1 %  → burn_rate = 0,01 / 0,001 = 10
  Si taux d'erreur = 2 %  → burn_rate = 0,02 / 0,001 = 20  ← CRITIQUE
```

---

### 1.4 Requêtes PromQL RED — pseudo-code et formules finales

> **Convention de cardinalité** : tous les services exposent le label `status_class`
> (`2xx`, `3xx`, `4xx`, `5xx`) au lieu du code HTTP brut. Cela réduit la cardinalité
> de ~500 séries à ~5 séries par route. Les routes sont normalisées (pattern Express /
> FastAPI / Go littéral) — jamais l'URL brute.
>
> **Convention de filtrage Canary** : dès l'étape 5, on ajoute le filtre
> `pod_template_hash` ou le label `rollouts-pod-template-hash` pour isoler
> le trafic Stable vs Canary dans les AnalysisTemplate.

---

#### 1.4.1 RED — Rate (débit en requêtes/seconde)

```promql
# ── Débit total — tous services ──────────────────────────────────────────────
# Fenêtre 5 min — adapté au scrape interval 30 s (min. 10 points pour stabilité)
sum(rate(http_requests_total[5m])) by (job)

# ── Débit par service, par route, par méthode ────────────────────────────────
sum(
  rate(http_requests_total{
    job=~"annuaire|planning|notif",
    route!~"/healthz|/readyz|/metrics"
  }[5m])
) by (job, route, method)

# ── Débit Canary uniquement (étape 5 — filtre anti-dilution) ─────────────────
# Le label rollouts-pod-template-hash est injecté par Argo Rollouts sur les pods Canary.
# Remplacer <HASH_CANARY> par la valeur réelle au moment du déploiement.
sum(
  rate(http_requests_total{
    job="annuaire",
    rollouts_pod_template_hash="<HASH_CANARY>"
  }[5m])
) by (rollouts_pod_template_hash)
```

---

#### 1.4.2 RED — Errors (taux d'erreurs 5xx)

```promql
# ── Taux d'erreur 5xx brut (par service) ─────────────────────────────────────
# Utilisé dans les AnalysisTemplate (étape 7) pour décider Abort/Promote.
sum(rate(http_requests_total{status_class="5xx", job="annuaire"}[5m]))
/
sum(rate(http_requests_total{job="annuaire"}[5m]))

# ── Taux d'erreur en % (plus lisible en dashboard Grafana) ───────────────────
100 * (
  sum(rate(http_requests_total{status_class="5xx", job="annuaire"}[5m]))
  /
  sum(rate(http_requests_total{job="annuaire"}[5m]))
)

# ── Taux d'erreur Canary vs Stable — agrégation spatiale correcte ─────────────
# ATTENTION : ne jamais agréger Canary+Stable puis diviser — cela dilue le signal.
# Calculer séparément pour chaque pod_template_hash, puis comparer.
sum by (rollouts_pod_template_hash) (
  rate(http_requests_total{status_class="5xx", job="annuaire"}[5m])
)
/
sum by (rollouts_pod_template_hash) (
  rate(http_requests_total{job="annuaire"}[5m])
)

# ── SLI Disponibilité — proportion de bonnes requêtes (pour SLO burn rate) ───
# good_ratio sur 30 jours glissants — base du calcul Error Budget
(
  sum(rate(http_requests_total{job="annuaire", status_class!="5xx"}[30d]))
  /
  sum(rate(http_requests_total{job="annuaire"}[30d]))
)

# ── Burn rate — court terme 1 h ───────────────────────────────────────────────
# (1 - good_ratio_1h) / (1 - SLO_target)
# annuaire SLO = 0.999  → denominateur = 0.001
(1 - (
  sum(rate(http_requests_total{job="annuaire", status_class!="5xx"}[1h]))
  /
  sum(rate(http_requests_total{job="annuaire"}[1h]))
)) / 0.001

# ── Burn rate — moyen terme 6 h ───────────────────────────────────────────────
(1 - (
  sum(rate(http_requests_total{job="annuaire", status_class!="5xx"}[6h]))
  /
  sum(rate(http_requests_total{job="annuaire"}[6h]))
)) / 0.001
```

---

#### 1.4.3 RED — Duration (latence p50 / p95 / p99)

```promql
# ── Latence p50 (médiane) — annuaire ─────────────────────────────────────────
histogram_quantile(
  0.50,
  sum(
    rate(http_request_duration_seconds_bucket{
      job="annuaire",
      route!~"/healthz|/readyz|/metrics"
    }[5m])
  ) by (le)
)

# ── Latence p95 — annuaire ────────────────────────────────────────────────────
# C'est le SLI principal de latence (SLO : p95 < 300 ms = 0.3 s)
histogram_quantile(
  0.95,
  sum(
    rate(http_request_duration_seconds_bucket{
      job="annuaire",
      route!~"/healthz|/readyz|/metrics"
    }[5m])
  ) by (le)
)

# ── Latence p99 — annuaire ────────────────────────────────────────────────────
histogram_quantile(
  0.99,
  sum(
    rate(http_request_duration_seconds_bucket{
      job="annuaire",
      route!~"/healthz|/readyz|/metrics"
    }[5m])
  ) by (le)
)

# ── Latence p95 — tous services (vue consolidée dashboard) ───────────────────
histogram_quantile(
  0.95,
  sum(
    rate(http_request_duration_seconds_bucket{
      job=~"annuaire|planning|notif",
      route!~"/healthz|/readyz|/metrics"
    }[5m])
  ) by (job, le)
)

# ── Latence p95 Canary — filtrage strict anti-dilution ───────────────────────
# Agrégation by (le, rollouts_pod_template_hash) obligatoire :
# les buckets Stable et Canary NE DOIVENT PAS être sommés ensemble avant
# l'appel histogram_quantile (résultat mathématiquement invalide).
histogram_quantile(
  0.95,
  sum(
    rate(http_request_duration_seconds_bucket{
      job="annuaire",
      rollouts_pod_template_hash="<HASH_CANARY>"
    }[5m])
  ) by (le)
)

# ── SLI Latence — proportion de requêtes sous le seuil SLO ───────────────────
# Méthode : lire le bucket correspondant au seuil SLO (le="0.3" pour annuaire)
# Plus précis que histogram_quantile pour comparer à un seuil fixe.
sum(
  rate(http_request_duration_seconds_bucket{
    job="annuaire",
    le="0.3",
    route!~"/healthz|/readyz|/metrics"
  }[5m])
)
/
sum(
  rate(http_request_duration_seconds_count{
    job="annuaire",
    route!~"/healthz|/readyz|/metrics"
  }[5m])
)
```

---

### 1.5 Synthèse des SLO — tableau opérationnel

| Service | SLO Dispo | Budget/mois | SLO Latence | Seuil p95 | Fast Burn | Slow Burn |
|---|---|---|---|---|---|---|
| **annuaire** | 99,9 % | 43 min | p95 | < 300 ms | BR ≥ 14,4 → PAGE | BR ≥ 6 → TICKET |
| **planning** | 99,5 % | 3 h 36 | p95 | < 500 ms | BR ≥ 14,4 → PAGE | BR ≥ 6 → TICKET |
| **notif** | 99,5 % | 3 h 36 | p95 | < 200 ms | BR ≥ 14,4 → PAGE | BR ≥ 6 → TICKET |

> **Fichier de référence :** [`platform-sre/slo-definitions.yaml`](file:///c:/Users/jammi/OneDrive%20-%20SUNELIS/Bureau/ESGI/TP%20Youssef/tp3-progressive-delivery/pulse-campus/platform-sre/slo-definitions.yaml)

---

### 1.6 Décisions d'architecture et justifications

**Pourquoi `status_class` et non `http_status_code` ?**
Avec 3 services × N routes × ~50 codes HTTP possibles, la cardinalité exploseraient à plusieurs milliers de séries. Le label `status_class` (`2xx`/`4xx`/`5xx`) réduit ce multiplicateur à 5, sans perdre d'information pour les SLO (qui distinguent uniquement succès vs erreur serveur).

**Pourquoi exclure `/healthz`, `/readyz`, `/metrics` des SLI ?**
Ces routes sont générées par Kubernetes (liveness/readiness probes) et par Prometheus lui-même. Les inclure gonflerait artificiellement le numérateur de bonnes requêtes et masquerait de vraies erreurs utilisateur.

**Pourquoi agréger `by (le)` avant `histogram_quantile` ?**
`histogram_quantile` requiert des buckets compatibles (même valeurs de `le`). En agrégeant d'abord par `le` avec `sum`, on obtient un histogramme multi-pod correct. Agréger par pod puis appeler `histogram_quantile` produirait une médiane de médianes (mathématiquement invalide).

**Pourquoi filtrer par `rollouts_pod_template_hash` en phase Canary ?**
Sans ce filtre, la métrique Stable (80 % du trafic) dilue la métrique Canary (20 %). Une erreur à 10 % sur le Canary apparaît comme ~2 % agrégé — sous le seuil d'alerte. Le filtre garantit que l'AnalysisTemplate mesure **uniquement** le pod Canary.



---

## Étape 2 — Instrumentation & Buckets PromQL

### 2.1 Pourquoi les buckets par défaut sont insuffisants

Les buckets de départ (`0.05, 0.1, 0.2, 0.3, 0.5, 1, 2, 5`) posent deux problèmes :

1. **Absence des seuils fins inférieurs à 50 ms** : pour un service Go comme `notif`, la majorité des requêtes répond en 5–30 ms. Avec `le="0.05"` comme premier bucket, on ne distingue pas les requêtes à 5 ms de celles à 48 ms — perte de précision sur le p50.

2. **Pas de couverture fine au démarrage de la plage utile** : l'histogramme de Prometheus utilise une interpolation linéaire entre deux bornes. Si le SLO est à 200 ms et que les buckets sont `..., 0.1, 0.2, 0.3, ...`, `histogram_quantile(0.95)` sera précis uniquement si 0.2 est une borne exacte.

**Règle fondamentale** : le seuil SLO (ex. 300 ms pour `annuaire`) **doit être une frontière exacte** (`le="0.3"`) pour que la formule SLI bucket-based (`bucket_le_SLO / count_total`) soit exacte et que `histogram_quantile(0.95)` ne soit pas biaisé par interpolation.

---

### 2.2 Design des buckets logarithmiques — méthode

Une progression logarithmique (facteur ≈ 2 à 2,5) couvre plusieurs ordres de grandeur avec un minimum de buckets. On vise :

- **Couverture** : 5 ms → 5 000 ms (couvrir les timeouts réseau)
- **Précision** : résolution fine (< 50 ms) là où se concentrent les requêtes rapides
- **Ancres SLO** : le seuil SLO de chaque service est une borne exacte
- **Cardinalité** : ≤ 12 buckets pour limiter le stockage TSDB

```
Bucket commun aux 3 services (valeur en secondes) :
  0.005 │  5 ms
  0.010 │ 10 ms
  0.025 │ 25 ms   ← ratio ×2.5
  0.050 │ 50 ms   ← ratio ×2
  0.100 │100 ms   ← ratio ×2
  0.200 │200 ms   ← SLO notif  ⚓
  0.300 │300 ms   ← SLO annuaire ⚓
  0.500 │500 ms   ← SLO planning ⚓
  1.000 │  1 s
  2.000 │  2 s
  5.000 │  5 s
         ──────── 11 buckets
```

> **Pourquoi un jeu commun pour les 3 services ?** Les 3 services étant instrumentés avec les mêmes noms de métriques (`http_request_duration_seconds`), un jeu commun de buckets permet de les comparer dans un même panel Grafana sans incohérence d'axe. Les 3 ancres SLO (`0.2`, `0.3`, `0.5`) sont toutes présentes.

---

### 2.3 Modifications des `values.yaml`

#### `services/annuaire/chart/values.yaml` — SLO 300 ms

```yaml
# Ancre SLO annuaire : le="0.3"
metrics:
  buckets: "0.005,0.01,0.025,0.05,0.1,0.2,0.3,0.5,1,2,5"
  businessEnabled: false
```

#### `services/planning/chart/values.yaml` — SLO 500 ms

```yaml
# Ancre SLO planning : le="0.5"
metrics:
  buckets: "0.005,0.01,0.025,0.05,0.1,0.2,0.3,0.5,1,2,5"
  businessEnabled: false
```

#### `services/notif/chart/values.yaml` — SLO 200 ms

```yaml
# Ancre SLO notif : le="0.2"
metrics:
  buckets: "0.005,0.01,0.025,0.05,0.1,0.2,0.3,0.5,1,2,5"
  businessEnabled: false
```

Ces valeurs sont injectées dans les pods via la variable d'environnement `METRICS_BUCKETS`, lue par `parseBuckets()` dans chaque service (Node.js, Python, Go).

---

### 2.4 Création du `PrometheusRule` — Recording Rules SLI

Fichier : [`platform-sre/recording-rules/recording-rules.yaml`](file:///c:/Users/jammi/OneDrive%20-%20SUNELIS/Bureau/ESGI/TP%20Youssef/tp3-progressive-delivery/pulse-campus/platform-sre/recording-rules/recording-rules.yaml)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: pulse-campus-sli-rules
  namespace: monitoring
  labels:
    release: kps         # ← obligatoire pour le ruleSelector du chart kps
```

**4 groupes de règles** produits :

| Groupe | Règles | Usage |
|---|---|---|
| `pulse_campus_rate_5m` | `sli:http_requests_total:rate5m`, `sli:http_requests_errors5xx:rate5m` | Dashboard RPS en temps réel |
| `pulse_campus_error_ratio` | ratio 5m / 1h / 6h par service | Base alertes burn rate (étape 10) |
| `pulse_campus_burn_rate` | burn rate 1h+6h pour chaque service, avec SLO encodé | Alertes `for` directes (étape 10) |
| `pulse_campus_latency_sli` | `sli:http_latency_under_slo:{annuaire,planning,notif}` | SLI latence bucket-based (p95 SLO check) |

**Avantage des recording rules** : les fenêtres `[1h]` et `[6h]` seraient trop lentes à évaluer à chaque chargement de dashboard. Les recording rules pré-calculent toutes les 30 s et stockent le résultat — les dashboards lisent une simple série temporelle.

---

### 2.5 Commandes de validation `promtool`

```bash
# ── Validation syntaxique de toutes les PrometheusRule (sans cluster) ─────────
cd pulse-campus
make promtool-check
# Sortie attendue :
#   ══════════════════════════════════════════════════════
#     promtool check rules — Pulse Campus SRE
#   ══════════════════════════════════════════════════════
#     >>> checking platform-sre/recording-rules/recording-rules.yaml
#     [OK] platform-sre/recording-rules/recording-rules.yaml
#   ──────────────────────────────────────────────────────
#   Toutes les PrometheusRule sont syntaxiquement valides.

# ── Validation directe (sans make) ───────────────────────────────────────────
promtool check rules \
  platform-sre/recording-rules/recording-rules.yaml

# ── Validation syntaxique d'une expression PromQL isolée ─────────────────────
# Utile pour tester une requête avant de la mettre dans un PrometheusRule.
promtool query instant \
  http://prometheus.devhub.local \
  'sli:http_requests_total:rate5m'

# ── Helm lint après modification des buckets ──────────────────────────────────
make helm-lint-all
# Sortie attendue :
#   >>> helm lint annuaire (values-dev.yaml)
#   ==> Linting services/annuaire/chart
#   [INFO] Chart.yaml: icon is recommended
#   1 chart(s) linted, 0 chart(s) failed
#   (idem pour planning et notif)

# ── Vérifier que les variables d'env sont bien passées aux pods ───────────────
# (après déploiement, cluster démarré)
kubectl -n devhub-dev exec deploy/annuaire-annuaire \
  -- env | grep METRICS_BUCKETS
# Attendu : METRICS_BUCKETS=0.005,0.01,0.025,0.05,0.1,0.2,0.3,0.5,1,2,5

# ── Vérifier que les buckets sont bien exposés dans /metrics ─────────────────
kubectl -n devhub-dev port-forward svc/annuaire-annuaire 9090:8080 &
curl -s http://localhost:9090/metrics \
  | grep 'http_request_duration_seconds_bucket' \
  | grep 'le="0.3"'
# Attendu : http_request_duration_seconds_bucket{...,le="0.3"} <valeur>
```

---

### 2.6 Tableau récapitulatif — Buckets par service

| Service | SLO Latence | Buckets (en secondes) | Ancre SLO |
|---|---|---|---|
| **annuaire** | 300 ms | `0.005, 0.01, 0.025, 0.05, 0.1, 0.2, **0.3**, 0.5, 1, 2, 5` | `le="0.3"` ✅ |
| **planning** | 500 ms | `0.005, 0.01, 0.025, 0.05, 0.1, 0.2, 0.3, **0.5**, 1, 2, 5` | `le="0.5"` ✅ |
| **notif** | 200 ms | `0.005, 0.01, 0.025, 0.05, 0.1, **0.2**, 0.3, 0.5, 1, 2, 5` | `le="0.2"` ✅ |

---

### 2.7 Décisions et justifications

**Pourquoi 11 buckets et pas plus ?**
Chaque bucket ajoute une série temporelle dans Prometheus TSDB. Avec 3 services × N routes × 3 status_class × 3 méthodes × 11 buckets ≈ plusieurs centaines de séries. Rester à 11 évite l'explosion de cardinalité tout en couvrant 3 ordres de grandeur.

**Pourquoi `sum without (pod, instance, node)` dans les recording rules ?**
On retire les labels haute-cardinalité (`pod`, `instance`, `node`) qui ne sont pas utiles pour les SLI mais doubleraient le nombre de séries. L'opérateur `without` (plutôt que `by`) est défensif : si un nouveau label est ajouté à l'avenir, il sera automatiquement retiré de l'agrégation.

**Pourquoi `interval: 30s` dans les groupes de règles ?**
Le scrape interval Prometheus est configuré à 30 s dans le chart kube-prometheus-stack. Les recording rules doivent avoir un interval d'évaluation ≥ scrape interval. Mettre 30 s garantit que les données disponibles sont toujours fraîches sans sur-évaluation inutile.



---

## Étape 3 — Déploiement GitOps kube-prometheus-stack

### 3.1 Architecture GitOps — Pattern App of Apps

```
ArgoCD
└── root (Application — root-app.yaml)
    ├── platform-sre/apps/observability/
    │   ├── kube-prometheus-stack   → chart Helm v65.5.0 (namespace: monitoring)
    │   ├── argo-rollouts           → chart Helm v2.37.7 (namespace: argo-rollouts)
    │   └── recording-rules         → PrometheusRule YAML   (namespace: monitoring)
    └── platform-sre/apps/dev/
        ├── annuaire-dev            → chart Helm local
        ├── planning-dev            → chart Helm local
        └── notif-dev               → chart Helm local
```

La root Application scanne récursivement `platform-sre/apps/` et crée toutes les Applications enfants. On ne touche qu'à **un seul `kubectl apply`** pour bootstrapper la plateforme entière.

---

### 3.2 Prérequis — Ordre impératif d'application

> **Point critique** : ArgoCD doit être installé et opérationnel **avant** d'appliquer la root Application. Le Secret Grafana doit exister **avant** la première sync de kube-prometheus-stack.

```bash
# ── 1. Créer le cluster Kind ──────────────────────────────────────────────────
cd pulse-campus
make cluster-up
# >>> cluster prêt — contexte courant : kind-pulse

# ── 2. Installer ingress-nginx + ArgoCD ───────────────────────────────────────
make argocd-install
# Attendre que le pod ArgoCD server soit Ready (~60-90 s)
kubectl wait --namespace argocd \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/name=argocd-server \
  --timeout=180s

# ── 3. Récupérer le mot de passe ArgoCD ───────────────────────────────────────
make argocd-password
# Copier le mot de passe affiché

# ── 4. Créer le namespace monitoring et le Secret Grafana ─────────────────────
# AVANT la sync ArgoCD (le pod Grafana échouerait sinon avec ErrImagePull
# sur le secret manquant)
kubectl create namespace monitoring
kubectl create secret generic grafana-admin-secret \
  --namespace monitoring \
  --from-literal=admin-user=admin \
  --from-literal=admin-password='Esgi@2026SRE!'
# ⚠️  Changez ce mot de passe ! N'utilisez jamais le défaut en production.

# ── 5. Appliquer l'AppProject (RBAC ArgoCD) ───────────────────────────────────
# L'AppProject doit exister avant les Applications qui en dépendent.
kubectl apply -f platform-sre/projects/devhub.yaml

# ── 6. Appliquer la root Application (point de départ unique) ─────────────────
# Modifiez d'abord root-app.yaml : remplacez <votre-handle> par votre handle GitHub
kubectl apply -f platform-sre/bootstrap/root-app.yaml

# ── 7. Synchroniser manuellement la première fois (ou attendre l'auto-sync) ────
argocd app sync root
argocd app sync kube-prometheus-stack
argocd app sync recording-rules
argocd app sync argo-rollouts

# ── 8. Ajouter les entrées /etc/hosts ─────────────────────────────────────────
make hosts-print
# Ajoutez les lignes affichées dans C:\Windows\System32\drivers\etc\hosts (Windows)
```

---

### 3.3 Sécurisation du mot de passe Grafana

#### Problème : `adminPassword` en clair dans Git

```yaml
# ❌ NE PAS FAIRE — mot de passe lisible par quiconque a accès au repo
grafana:
  adminPassword: "changeme"
```

#### Solution : `existingSecret` + Secret Kubernetes hors Git

```yaml
# ✅ À FAIRE — référence à un Secret pré-créé (jamais dans Git en clair)
grafana:
  admin:
    existingSecret: grafana-admin-secret
    userKey: admin-user
    passwordKey: admin-password
```

| Approche | Convient | Description |
|---|---|---|
| `kubectl create secret` (TP) | Cluster local | Secret créé manuellement, non commité |
| **Sealed Secrets** | Staging/Prod | Secret chiffré avec clé publique cluster, commitable |
| **External Secrets Operator** | Prod | Secret tiré depuis Vault / AWS SM à chaque sync |
| **ArgoCD Vault Plugin** | Prod | Injection à la sync via template `<path:secret/grafana#password>` |

---

### 3.4 Compatibilité Kind — Composants désactivés

Kind ne fournit pas les endpoints de monitoring de la control plane Kubernetes standard. Sans désactivation, Prometheus aurait des cibles **DOWN** permanentes et des alertes `KubeControllerManagerDown` se déclencheraient en faux positif.

| Composant | Port standard | Pourquoi désactivé sur Kind |
|---|---|---|
| `kubeControllerManager` | 10257 | Non exposé sur Kind (endpoint absent) |
| `kubeScheduler` | 10259 | Non exposé sur Kind |
| `kubeEtcd` | 2381 | Etcd Kind non configurable pour scrape externe |
| `kubeProxy` | 10249 | Kind utilise iptables-legacy, pas de kube-proxy dédié |
| `prometheusOperator.admissionWebhooks` | — | Requiert cert-manager (absent du TP) |

```yaml
# platform-sre/values/kube-prometheus-stack-values.yaml
kubeControllerManager: { enabled: false }
kubeScheduler:         { enabled: false }
kubeEtcd:              { enabled: false }
kubeProxy:             { enabled: false }
prometheusOperator:
  admissionWebhooks:
    enabled: false
    patch:   { enabled: false }
  tls:       { enabled: false }
```

---

### 3.5 Configuration des Ingress

```yaml
# Grafana — http://grafana.devhub.local
grafana:
  ingress:
    enabled: true
    ingressClassName: nginx
    hosts: [grafana.devhub.local]
    path: /
    pathType: Prefix

# Prometheus — http://prometheus.devhub.local
prometheus:
  ingress:
    enabled: true
    ingressClassName: nginx
    hosts: [prometheus.devhub.local]
    paths: [/]
    pathType: Prefix
```

Les noms de domaine `*.devhub.local` doivent être résolus localement (fichier `hosts`). Le contrôleur ingress-nginx est installé sur le nœud `control-plane` de Kind avec `hostPort: 80/443`.

---

### 3.6 Sélecteurs — Comment Prometheus découvre les métriques

```yaml
prometheusSpec:
  # Désactive le comportement par défaut (découvrir uniquement les SM du même chart)
  serviceMonitorSelectorNilUsesHelmValues: false
  serviceMonitorSelector:
    matchLabels:
      release: kps          # ← chaque ServiceMonitor DOIT porter ce label
  serviceMonitorNamespaceSelector: {}   # tous les namespaces

  ruleSelectorNilUsesHelmValues: false
  ruleSelector:
    matchLabels:
      release: kps          # ← chaque PrometheusRule DOIT porter ce label
  ruleNamespaceSelector: {}
```

> **Piège fréquent** : si un ServiceMonitor ne porte pas `release: kps`, Prometheus ne le découvre pas et aucune erreur n'est visible dans ArgoCD (le CRD est valide). Vérifiez avec :
> ```bash
> kubectl -n monitoring get servicemonitor -A --show-labels | grep release
> ```

---

### 3.7 Vérifications post-déploiement

```bash
# ── Vérifier que les pods monitoring sont Running ─────────────────────────────
kubectl -n monitoring get pods
# Attendu : prometheus-kps-0, alertmanager-kps-0, kps-grafana-*, kps-kube-state-metrics-*

# ── Vérifier que le Secret Grafana est monté ──────────────────────────────────
kubectl -n monitoring get secret grafana-admin-secret -o jsonpath='{.data.admin-user}' \
  | base64 -d ; echo
# Attendu : admin

# ── Tester l'accès Grafana ────────────────────────────────────────────────────
curl -s -o /dev/null -w "%{http_code}" \
  http://grafana.devhub.local
# Attendu : 200 ou 302 (redirect login)

# ── Vérifier que les recording rules sont chargées par Prometheus ─────────────
curl -s http://prometheus.devhub.local/api/v1/rules \
  | jq '.data.groups[].name' \
  | grep pulse_campus
# Attendu : "pulse_campus_rate_5m", "pulse_campus_error_ratio", etc.

# ── Vérifier les targets Prometheus (tous UP sauf control-plane désactivé) ────
curl -s http://prometheus.devhub.local/api/v1/targets \
  | jq '.data.activeTargets[] | select(.health != "up") | .labels.job'
# Attendu : vide (aucun target DOWN hors ceux explicitement désactivés)

# ── Vérifier l'état ArgoCD ────────────────────────────────────────────────────
argocd app list
# Attendu : kube-prometheus-stack → Synced/Healthy
#           recording-rules       → Synced/Healthy
#           argo-rollouts         → Synced/Healthy
```

---

### 3.8 Structure finale de `platform-sre/`

```
platform-sre/
├── apps/
│   ├── dev/
│   │   ├── annuaire.yaml          # Application ArgoCD dev
│   │   ├── planning.yaml
│   │   └── notif.yaml
│   └── observability/
│       ├── kube-prometheus-stack.yaml   # ← chart Helm KPS
│       ├── argo-rollouts.yaml           # ← chart Helm Rollouts
│       └── recording-rules.yaml         # ← PrometheusRule SLI  [NEW]
├── bootstrap/
│   └── root-app.yaml              # Point d'entrée unique App of Apps
├── projects/
│   └── devhub.yaml                # AppProject avec CRD whitelist complète [UPDATED]
├── recording-rules/
│   └── recording-rules.yaml       # PrometheusRule (validé promtool)
├── secrets/
│   └── grafana-admin-secret.yaml  # Template Secret (CHANGEME_BASE64 — ne pas committer rempli)
├── slo-definitions.yaml           # Référentiel SLO (documentaire)
└── values/
    ├── kube-prometheus-stack-values.yaml   # Values Helm complètes [UPDATED]
    └── argo-rollouts-values.yaml
```



---

## Étape 4 — ServiceMonitor & Dashboard Grafana

### 4.1 Architecture du flux de scrape

```
Pod annuaire (:8080/metrics)
    ↑ scrape toutes les 30 s
ServiceMonitor annuaire (namespace devhub-dev)
    label release: kps   ←── OBLIGATOIRE
    ↑ découverte par
Prometheus Operator (namespace monitoring)
    ruleSelector: {release: kps}
    serviceMonitorSelector: {release: kps}
    ↑ gère
Prometheus StatefulSet (namespace monitoring)
    ↓ expose
Grafana DataSource (prometheus.devhub.local)
    ↓ requêtes PromQL
Dashboard "Pulse Campus — RED & SLO"
```

---

### 4.2 ServiceMonitor — Template Helm

Un fichier `servicemonitor.yaml` a été ajouté dans le répertoire `templates/` de chaque chart. Il est conditionnel (`{{- if .Values.monitoring.enabled }}`).

```yaml
# services/annuaire/chart/templates/servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: {{ include "annuaire.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "annuaire.labels" . | nindent 4 }}
    # OBLIGATOIRE : doit correspondre à prometheusSpec.serviceMonitorSelector.matchLabels
    release: {{ .Values.monitoring.serviceMonitor.release | quote }}
spec:
  selector:
    matchLabels:
      {{- include "annuaire.selectorLabels" . | nindent 6 }}
  namespaceSelector:
    matchNames:
      - {{ .Release.Namespace }}
  endpoints:
    - port: http
      path: /metrics
      interval: {{ .Values.monitoring.serviceMonitor.interval }}
      scrapeTimeout: 10s
      honorLabels: false
```

#### Valeurs par défaut dans `values.yaml` (commun aux 3 services)

```yaml
monitoring:
  enabled: false          # true dans values-dev.yaml
  serviceMonitor:
    interval: 30s         # aligné sur le scrapeInterval global de Prometheus
    release: kps          # DOIT correspondre au releaseName du chart kps
```

#### Activation dans `values-dev.yaml`

```yaml
# Étape 4 : ServiceMonitor activé.
monitoring:
  enabled: true
```

> **Piège** : Le label `release: kps` est sur le **ServiceMonitor** (objet Kubernetes), pas sur le pod ni sur le Service. Prometheus Operator filtre par ce label lors de la découverte. Si vous oubliez ce label, Prometheus ne scrape pas le service et aucune erreur n'est visible dans ArgoCD.

---

### 4.3 Vérification de la découverte ServiceMonitor

```bash
# ── Vérifier que les 3 ServiceMonitor sont créés dans devhub-dev ──────────────
kubectl -n devhub-dev get servicemonitor -o wide
# Attendu : annuaire-annuaire, planning-planning, notif-notif

# ── Vérifier qu'ils portent le label release: kps ─────────────────────────────
kubectl -n devhub-dev get servicemonitor --show-labels | grep release
# Attendu : release=kps sur chaque ligne

# ── Vérifier que Prometheus a bien découvert les 3 services ───────────────────
# Via l'UI Prometheus : http://prometheus.devhub.local/targets
# Ou via l'API :
curl -s http://prometheus.devhub.local/api/v1/targets \
  | jq '.data.activeTargets[] | select(.labels.job | test("annuaire|planning|notif")) | {job: .labels.job, health: .health}'
# Attendu : {"job": "annuaire", "health": "up"}  ×3

# ── Vérifier que les métriques remontent ──────────────────────────────────────
curl -s 'http://prometheus.devhub.local/api/v1/query?query=up{job="annuaire"}' \
  | jq '.data.result[].value[1]'
# Attendu : "1"
```

---

### 4.4 Dashboard Grafana — Structure

Fichier : [`platform-sre/dashboards/pulse-campus-dashboard.yaml`](file:///c:/Users/jammi/OneDrive%20-%20SUNELIS/Bureau/ESGI/TP%20Youssef/tp3-progressive-delivery/pulse-campus/platform-sre/dashboards/pulse-campus-dashboard.yaml)

Le dashboard est packagé dans un **ConfigMap** avec le label `grafana_dashboard: "1"`. Le sidecar Grafana le détecte et l'importe automatiquement dans le folder "Pulse Campus".

#### Panels — vue d'ensemble

| Panel | Type | PromQL (source) | Justification technique |
|---|---|---|---|
| **RPS Total** | Stat | `sum(sli:http_requests_total:rate5m{job=~"$job"})` | Recording rule pré-calculée — chargement instantané |
| **Taux d'erreur** | Stat | `sum(sli:http_request_error_ratio:rate5m)` | Couleur background rouge si > 1 % |
| **Burn Rate annuaire 1h** | Stat | `slo:http_burn_rate_1h:annuaire` | Seuil rouge 14.4 (PAGE) |
| **Build Info** | Table | `{__name__=~".*_build_info"}` | Métadonnées version, commit, language |
| **RPS par service** | Timeseries | `sum by (job) (sli:http_requests_total:rate5m)` | Agrégation `by (job)` — 1 série par service |
| **Error Rate %** | Timeseries | `sum by (job) (sli:http_request_error_ratio:rate5m)` | Avec ligne pointillée SLO budget 0.1 % |
| **Burn Rate multi-services** | Timeseries | `slo:http_burn_rate_{1h,6h}:{annuaire,planning,notif}` | 6 séries + 2 seuils (PAGE/TICKET) |
| **Latence p50/p95/p99 — annuaire** | Timeseries | `histogram_quantile(0.95, sum by (le) (rate(...[5m])))` | `sum by (le)` AVANT `histogram_quantile` |
| **Latence p50/p95/p99 — planning** | Timeseries | idem, SLO 500 ms | Ligne rouge SLO à 0.5 s |
| **Latence p50/p95/p99 — notif** | Timeseries | idem, SLO 200 ms | Ligne rouge SLO à 0.2 s |
| **SLI Latence annuaire** | Gauge | `sli:http_latency_under_slo:annuaire` | Bucket-based — exact pour seuil fixe |
| **SLI Latence planning** | Gauge | `sli:http_latency_under_slo:planning` | Vert si > 99.5 % |
| **SLI Latence notif** | Gauge | `sli:http_latency_under_slo:notif` | Vert si > 99.5 % |

---

### 4.5 Points techniques — Agrégation histogram correcte

> **Règle critique** : toujours agréger `by (le)` **avant** d'appeler `histogram_quantile`.

```promql
-- ❌ INCORRECT : histogram_quantile(0.95, rate(...)) by (job)
-- Cela calcule un quantile par série individuelle (pod), puis agrège — mathématiquement invalide.

-- ✅ CORRECT : sum by (le) PUIS histogram_quantile
histogram_quantile(
  0.95,
  sum by (le) (
    rate(http_request_duration_seconds_bucket{
      job="annuaire",
      route!~"/healthz|/readyz|/metrics"
    }[5m])
  )
)
```

**Pourquoi `honorLabels: false` dans le ServiceMonitor ?**
Si le pod expose un label `job` personnalisé (certains exporters le font), avec `honorLabels: true`, ce label écraserait le label `job` injecté par Prometheus (qui vaut le nom du ServiceMonitor). On force `false` pour garantir que `job="annuaire"` dans toutes les requêtes PromQL.

---

### 4.6 Auto-discovery des dashboards — ConfigMap label flow

```
ConfigMap (namespace monitoring)
  label: grafana_dashboard: "1"
      ↓ détecté par
Grafana sidecar (container dans le pod Grafana)
      ↓ lit le JSON
  key: pulse-campus-red-slo.json
      ↓ importe dans
Grafana folder: "Pulse Campus"
  Dashboard uid: "pulse-campus-red-slo-v1"
```

```bash
# ── Vérifier que le ConfigMap est créé ────────────────────────────────────────
kubectl -n monitoring get configmap pulse-campus-red-slo-dashboard

# ── Vérifier l'import par le sidecar (logs) ───────────────────────────────────
kubectl -n monitoring logs -l app.kubernetes.io/name=grafana \
  -c grafana-sc-dashboard --tail=20
# Attendu : INFO :: ConfigMap updated: pulse-campus-red-slo-dashboard

# ── Vérifier via l'API Grafana ────────────────────────────────────────────────
curl -s -u admin:'Esgi@2026SRE!' \
  http://grafana.devhub.local/api/dashboards/uid/pulse-campus-red-slo-v1 \
  | jq '.dashboard.title'
# Attendu : "Pulse Campus — RED & SLO"
```

---

### 4.7 Variables de template du dashboard

| Variable | Type | Query | Usage |
|---|---|---|---|
| `datasource` | datasource | Prometheus | Bascule entre datasources (multi-cluster) |
| `job` | query | `label_values(http_requests_total, job)` | Filtre par service — multi-select + All |

La variable `job` avec `allValue: "annuaire|planning|notif"` permet d'utiliser `job=~"$job"` dans les requêtes. Quand "All" est sélectionné, la regex filtre les 3 services.



## Étape 5 — Migration vers Rollout Canary

### 5.1 Architecture du Rollout Canary pour `annuaire`

Pour le service `annuaire`, nous abandonnons le déploiement natif Kubernetes (`Deployment`) pour un contrôleur de déploiement progressif avancé : **Argo Rollouts** (`Rollout`). 

La stratégie Canary consiste à acheminer une fraction du trafic de production vers la nouvelle version (Canary) tout en maintenant le reste sur l'ancienne version stable, puis à augmenter progressivement cette part après validation.

```
                  Trafic Utilisateur (annuaire.devhub.local Ingress)
                                     │
                     ┌───────────────┴───────────────┐
             (Stable - e.g. 80%)            (Canary - e.g. 20%)
                     │                               │
                     ▼                               ▼
             Service Stable                   Service Canary
          (annuaire-annuaire-stable)       (annuaire-annuaire-canary)
                     │                               │
                     ▼                               ▼
                 Pods V1                          Pods V2
             (Hash Stable)                    (Hash Canary)
```

---

### 5.2 Les Manifestes Kubernetes introduits

Pour mettre en œuvre ce routage progressif avec Nginx Ingress Controller, trois fichiers ont été créés dans `services/annuaire/chart/templates/` :

1. **`rollout.yaml`** : Définit la ressource personnalisée `Rollout` qui remplace le `Deployment` quand `useDeployment` est configuré à `false` dans les values Helm.
2. **`service-stable.yaml`** : Service Kubernetes ciblant uniquement la version stable. Son sélecteur est mis à jour dynamiquement par Argo Rollouts avec le hash de révision stable (`rollouts-pod-template-hash`).
3. **`service-canary.yaml`** : Service Kubernetes ciblant uniquement la version canary, géré de la même manière pour isoler la nouvelle version.

#### Routage Nginx Ingress

Pour répartir le trafic, Argo Rollouts utilise le contrôleur d'Ingress Nginx local. Il duplique automatiquement la définition de l'Ingress principal (`annuaire-annuaire`) pour créer un Ingress "Canary" temporaire portant l'annotation `nginx.ingress.kubernetes.io/canary: "true"` et `nginx.ingress.kubernetes.io/canary-weight: "<poids>"` ajusté à chaque étape.

---

### 5.3 Configuration du Rollout dans `values.yaml`

Afin d'activer Argo Rollouts, la variable `useDeployment` est passée à `false` dans le fichier de valeurs Helm, et la stratégie canary est paramétrée ainsi :

```yaml
# services/annuaire/chart/values.yaml
useDeployment: false

rollout:
  revisionHistoryLimit: 3
  canary:
    pauseDuration: "0"  # Pause indéfinie (attente de promotion manuelle via make rollout-promote ou kubectl argo rollouts)
    maxSurge: "1"       # Un seul pod supplémentaire démarré pendant le rollout
    maxUnavailable: "0" # Aucun pod indisponible pour préserver la capacité de service
```

La stratégie est définie avec 4 étapes progressives de trafic :
- **Étape 1** : Envoyer 20% du trafic sur le Canary, puis pause indéfinie.
- **Étape 2** : Envoyer 40% du trafic sur le Canary, puis pause indéfinie.
- **Étape 3** : Envoyer 60% du trafic sur le Canary, puis pause indéfinie.
- **Étape 4** : Promotion à 100% (le stable prend la nouvelle version).

---

### 5.4 Commandes CLI d'Administration via le Makefile

Des cibles dédiées ont été ajoutées dans le `Makefile` pour piloter et observer les rollouts :

```bash
# 1. Vérifier le statut actuel des rollouts (annuaire et planning)
make rollout-status

# 2. Suivre en direct la progression visuelle du déploiement
make rollout-watch

# 3. Promouvoir manuellement le rollout au palier suivant (ou terminer la pause)
make rollout-promote

# 4. Annuler immédiatement un rollout suspect et restaurer 100% du trafic stable
make rollout-abort
```

#### Équivalents CLI Argo Rollouts natifs :
- Statut : `kubectl argo rollouts status annuaire-annuaire -n devhub-dev`
- Visualisation : `kubectl argo rollouts get rollout annuaire-annuaire -n devhub-dev --watch`
- Promotion : `kubectl argo rollouts promote annuaire-annuaire -n devhub-dev`
- Annulation : `kubectl argo rollouts abort annuaire-annuaire -n devhub-dev`

---

### 5.5 Anti-Dilution des métriques RED

Lors d'un déploiement progressif, analyser l'ensemble des métriques d'un service (Canary + Stable) peut masquer les anomalies. Par exemple, si le Canary (20% du trafic) a un taux d'erreur de 10%, le taux d'erreur global moyenné sera de seulement 2%.

**Solution mise en place** :
Argo Rollouts ajoute l'étiquette `rollouts-pod-template-hash` sur chaque pod généré. Les requêtes PromQL RED dans Grafana ou dans les futures analyses automatiques (`AnalysisTemplate`) filtrent spécifiquement sur le hash du Canary grâce à cette étiquette, évitant toute dilution du signal d'erreur.

---

## Étape 6 — Guide opérationnel des scénarios manuels

Dans cette étape, nous documentons le pilotage manuel d'Argo Rollouts depuis le terminal ou via l'UI d'ArgoCD. 

### 6.1 Séquence de Steps configurée

Les étapes du Rollout configurées dans [`rollout.yaml`](file:///c:/Users/jammi/OneDrive%20-%20SUNELIS/Bureau/ESGI/TP%20Youssef/tp3-progressive-delivery/pulse-campus/services/annuaire/chart/templates/rollout.yaml) sont les suivantes :
```yaml
      steps:
        # Palier 1 : Envoi de 10% du trafic sur le Canary, pause indéfinie (attente d'action humaine)
        - setWeight: 10
        - pause: {}

        # Palier 2 : Envoi de 50% du trafic, pause d'une minute avant promotion finale à 100%
        - setWeight: 50
        - pause:
            duration: 1m
```

---

### 6.2 Scénario 1 : Promotion Nominale

#### 1. Déclenchement
Un commit d'une nouvelle version d'image ou d'une variable d'environnement sur le service `annuaire` est poussé et synchronisé par ArgoCD. Le contrôleur Argo Rollouts crée le nouveau ReplicaSet Canary, l'Ingress temporaire de routage, et lui affecte 10% du trafic. Il se met ensuite en état `Paused` de manière indéfinie.

```bash
# Vérification visuelle
make rollout-status
# Attendu : Le rollout est suspendu à l'étape 1/2 (poids 10%)
```

#### 2. Promotion
Après vérification des métriques RED sur le dashboard Grafana ou via des requêtes PromQL directes, l'opérateur valide la version.
```bash
# Commande de promotion
make rollout-promote
# CLI directe : kubectl argo rollouts promote annuaire-annuaire -n devhub-dev
```

#### 3. Comportement
Le rollout bascule immédiatement sur l'étape suivante : 50% du trafic. Une pause d'une minute (`duration: 1m`) s'enclenche automatiquement. À la fin de cette minute, si aucune intervention n'a eu lieu, le rollout passe à 100%, l'ancien ReplicaSet est mis hors service, et le déploiement est marqué comme terminé (`Healthy`).

---

### 6.3 Scénario 2 : Annulation Explicite (`Abort`)

#### 1. Déclenchement & Constat d'Anomalie
Pendant que le rollout est bloqué à la première étape (10% du trafic), les dashboards signalent une hausse importante d'erreurs 5xx ou une dégradation importante de la latence p95 sur les pods canary (détectée grâce au filtrage par `rollouts_pod_template_hash`).

#### 2. Annulation
L'opérateur décide de stopper immédiatement le déploiement :
```bash
# Commande d'annulation
make rollout-abort
# CLI directe : kubectl argo rollouts abort annuaire-annuaire -n devhub-dev
```

#### 3. Comportement
Le trafic sur le ReplicaSet Canary retombe instantanément à 0%. L'Ingress secondaire est détruit, protégeant ainsi les 90% d'utilisateurs restants. Le rollout passe en état `Degraded` et le ReplicaSet Canary est marqué comme `ScaledDown`.

> [!WARNING]
> **Gestion du Drift GitOps en cas d'Abort** : 
> L'exécution d'un `abort` manuel modifie l'état dans le cluster mais ne change pas les sources Git. Il y a donc un **drift** entre Git (qui demande toujours la nouvelle image version `V2`) et Kubernetes (qui utilise `V1` après l'abort).
> Pour corriger ce drift proprement, l'équipe SRE doit immédiatement effectuer un `git revert` sur Git de la PR ayant déclenché l'anomalie, afin qu'ArgoCD synchronise à nouveau la version stable légitime.

---

### 6.4 Scénario 3 : Promotion Forcée (`promote --full`)

#### 1. Déclenchement
Il peut arriver que l'opérateur veuille outrepasser complètement les étapes du déploiement progressif (ex: correction d'une faille de sécurité majeure, mise à jour critique bloquante).
```bash
# Commande CLI directe (hors Makefile)
kubectl argo rollouts promote annuaire-annuaire -n devhub-dev --full
```

#### 2. Comportement
Argo Rollouts ignore immédiatement tous les steps restants (les pauses de 10% et 50%) et bascule le trafic directement à 100% sur la nouvelle version.

#### 3. Analyse du risque en production
> [!CAUTION]
> **Le promote --full est extrêmement dangereux en production.**
> Il désactive tout le filet de sécurité et propage la nouvelle version instantanément. Si cette dernière contient un bug applicatif critique non détecté par les tests de CI, 100% des utilisateurs subiront la régression immédiatement (retour au scénario d'une RollingUpdate non sécurisée).
>
> **Cas acceptables :**
> - Déploiement d'une correction urgente ("Hotfix") déjà testée sur environnement de staging et destinée à corriger une panne bloquant 100% des utilisateurs de toute façon.
> - Patch de vulnérabilité de sécurité critique activement exploitée.
>
> **Précautions indispensables à prendre :**
> - Avoir vérifié au préalable l'image applicative sur un environnement de staging / sandbox identique.
> - S'assurer qu'un opérateur est présent pour surveiller en direct les dashboards RED durant les 10 premières minutes de la promotion forcée.
> - Être prêt à exécuter immédiatement un rollback rapide en cas d'alerte.

---


## Étape 7 — AnalysisTemplate automatique

L'analyse automatisée permet de supprimer l'intervention humaine lors du déploiement progressif. C'est le contrôleur **Argo Rollouts** qui interroge régulièrement la stack d'observabilité (Prometheus) pour vérifier les SLOs métier.

---

### 7.1 Intégration de l'AnalysisTemplate dans le Cycle de Vie

Nous avons inséré l'étape d'analyse automatique (`AnalysisRun`) juste après le palier de **25%** de trafic et avant d'autoriser le passage à **50%**.

```yaml
# Extrait de services/annuaire/chart/templates/rollout.yaml
      steps:
        # Étape 1 : 25% du trafic sur le Canary
        - setWeight: 25

        # Étape 2 : Analyse automatique de 5 minutes
        - analysis:
            templates:
              - templateName: {{ include "annuaire.fullname" . }}-analysis
            args:
              - name: canary-hash
                valueFrom:
                  # Le contrôleur injecte automatiquement le hash du ReplicaSet Canary
                  podTemplateHashValue: Latest

        # Étape 3 : Si l'analyse réussit, passage à 50% du trafic
        - setWeight: 50
        - pause:
            duration: 1m
```

---

### 7.2 Structure de l'AnalysisTemplate

Le fichier [`analysistemplate.yaml`](file:///c:/Users/jammi/OneDrive%20-%20SUNELIS/Bureau/ESGI/TP%20Youssef/tp3-progressive-delivery/pulse-campus/services/annuaire/chart/templates/analysistemplate.yaml) est intégré dans le chart Helm. Il définit :
- L'adresse du Prometheus interne : `http://prometheus-operated.monitoring.svc.cluster.local:9090`.
- Deux métriques évaluées toutes les 30 secondes (`interval: 30s`) sur une durée de 5 minutes (`count: 10`).
- Un comportement d'annulation immédiate au premier échec (`failureLimit: 0`).

---

### 7.3 Requêtes PromQL RED Anti-Dilution

#### 1. Métrique Taux d'Erreur (Disponibilité)

Nous mesurons uniquement le taux d'erreur 5xx du **Canary** (identifié par son hash) par rapport au volume total de requêtes qui lui sont adressées. Nous excluons les requêtes système de probes (`healthz|readyz|metrics`) pour ne pas fausser le calcul.

```promql
(
  sum(rate(http_requests_total{
    job="annuaire",
    status_class="5xx",
    rollouts_pod_template_hash="{{args.canary-hash}}",
    route!~"/healthz|/readyz|/metrics"
  }[2m]))
  /
  sum(rate(http_requests_total{
    job="annuaire",
    rollouts_pod_template_hash="{{args.canary-hash}}",
    route!~"/healthz|/readyz|/metrics"
  }[2m]))
) or vector(0)
```
- **Seuil critique (`successCondition`)** : `< 0.01` (moins de 1% d'erreurs).
- **Le rôle de `or vector(0)`** : Si aucun trafic ne passe sur le Canary (ex: test local sans charge), la division renverrait un résultat vide (`NaN`), ce qui ferait échouer l'analyse. `or vector(0)` renvoie par défaut 0% d'erreur, permettant de valider l'étape nominalement.

#### 2. Métrique Latence p95 (Performance)

Nous calculons la latence au 95ème centile uniquement sur les pods portant le hash de la version Canary.

```promql
(
  histogram_quantile(
    0.95,
    sum by (le) (
      rate(http_request_duration_seconds_bucket{
        job="annuaire",
        rollouts_pod_template_hash="{{args.canary-hash}}",
        route!~"/healthz|/readyz|/metrics"
      }[2m])
    )
  )
) or vector(0)
```
- **Seuil critique (`successCondition`)** : `< 0.3` (latence p95 inférieure à 300 ms).

---

### 7.4 Scénarios d'Échec et Rollback Automatique

#### Cas Nominal (Succès)
1. Le code V2 est sain.
2. L'AnalysisRun s'exécute pendant 5 minutes. Les 10 mesures successives renvoient un taux d'erreur = 0 et un p95 < 300ms.
3. Argo Rollouts promeut le déploiement à 50% automatiquement.

#### Cas Dégradé (Panne applicative)
1. Le code V2 génère par exemple 5% de requêtes 5xx.
2. Dès la première ou deuxième itération de l'AnalysisRun (intervalle de 30s), la métrique `error-rate` renvoie `0.05`.
3. Le seuil `successCondition (result[0] < 0.01)` n'est pas respecté. Comme `failureLimit` est configuré à `0`, la métrique passe immédiatement au statut `Failed`.
4. Le contrôleur Argo Rollouts arrête instantanément l'AnalysisRun, bascule le trafic vers le Stable à 100%, et passe le Rollout en état `Degraded`. Aucun opérateur humain n'a eu besoin d'intervenir.

```bash
# Vérifier l'état de l'analyse après un échec
kubectl argo rollouts get rollout annuaire-annuaire -n devhub-dev
# Visualiser le statut détaillé de l'AnalysisRun
kubectl get analysisruns -n devhub-dev
```

## Étape 8 — Stratégie Blue/Green pour Planning

### 8.1 Comparatif Architectural : Canary vs Blue/Green

Le choix entre un déploiement progressif (Canary) et une bascule instantanée (Blue/Green) dépend de contraintes de coût, de base de données, et de l'architecture logicielle :

| Critère | Stratégie Canary (ex: `annuaire`) | Stratégie Blue/Green (ex: `planning`) |
|---|---|---|
| **Consommation Ressources** | **Économe** : Ne nécessite qu'une légère surcharge temporaire de pods (ex: `maxSurge: 1`). | **Double** : Nécessite la duplication complète à 100% de la flotte de pods de test pendant la validation. |
| **Bascule Trafic** | **Progressive** : Transition par paliers (ex: 25%, 50%, 100%). Idéal pour détecter les régressions sous charge. | **Instantanée** : Bascule de 0 à 100% en modifiant le sélecteur du Service Kubernetes actif. |
| **Vitesse Rollback** | **Rapide** : Mais nécessite la destruction progressive ou immédiate des pods canary fautifs. | **Instantanée (Zero-downtime)** : Simple re-routage du Service actif vers l'ancienne version encore active. |
| **Compatibilité Schéma DB** | **Complexe** : Les versions stable et canary tournent en même temps et doivent supporter la même structure de base de données. | **Plus simple** : Mais l'analyse de preview peut toujours générer des écritures concurrentes. |
| **Cas d'usage recommandé** | Services HTTP à fort trafic, APIs stateless, microservices critiques. | Services stateful, applications monolithiques lourdes, ou backends avec traitement de tâches. |

---

### 8.2 Configuration GitOps pour `planning`

Le service `planning` est configuré en Blue/Green avec l'architecture suivante :

```
                                 Ingress Principal
                             (planning.devhub.local)
                                        │
                                        ▼
                             Service Active (stable)
                           (planning-planning-active)
                                        │
                         ┌──────────────┴──────────────┐
                 (Version Blue)                 (Version Green)
                   Pods V1 (100%)                  Pods V2 (0%)
                         ▲                               ▲
                         │                               │
              Active selector points           Preview selector points
             to stable ReplicaSet             to preview ReplicaSet
                         │                               │
                         └──────────────┬────────────────┘
                                        │
                                        ▼
                             Service Preview (test)
                           (planning-planning-preview)
                                        ▲
                                        │
                                 Ingress Preview
                         (planning-preview.devhub.local)
```

#### 1. Fichier `rollout.yaml` pour `planning`
La configuration intègre le double service et l'analyse automatisée de pré-promotion :

```yaml
# services/planning/chart/templates/rollout.yaml
spec:
  replicas: 2
  strategy:
    blueGreen:
      # Service servant la production
      activeService: planning-planning-active
      # Service servant aux tests internes avant bascule
      previewService: planning-planning-preview
      
      # Conserver l'ancienne version active pendant 5 min après promotion
      scaleDownDelaySeconds: 300
      
      # Promotion automatique après réussite de l'analyse
      autoPromotionEnabled: true
      
      # Analyse automatique de 2 minutes sur la version preview
      prePromotionAnalysis:
        templates:
          - templateName: planning-planning-analysis
        args:
          - name: preview-hash
            valueFrom:
              podTemplateHashValue: Latest
```

---

### 8.3 Ingress Split et Validation Manuelle

Pour inspecter la nouvelle version avant la bascule de production, nous avons séparé les points d'entrée :
- **Production** : [`planning.devhub.local`](file:///c:/Users/jammi/OneDrive%20-%20SUNELIS/Bureau/ESGI/TP%20Youssef/tp3-progressive-delivery/pulse-campus/services/planning/chart/templates/ingress.yaml#L11) (Pointe sur `planning-planning-active`)
- **Équipe Interne (Preview)** : [`planning-preview.devhub.local`](file:///c:/Users/jammi/OneDrive%20-%20SUNELIS/Bureau/ESGI/TP%20Youssef/tp3-progressive-delivery/pulse-campus/services/planning/chart/templates/ingress.yaml#L33) (Pointe sur `planning-planning-preview`)

#### Étapes de Validation d'un Déploiement :
1. Une mise à jour de `planning` est poussée.
2. Argo Rollouts déploie le ReplicaSet Green à 100% de sa capacité (2 pods).
3. `kubectl get pods -l app=planning` affiche **4 pods running** (2 Blue et 2 Green), doublant la charge.
4. L'URL `planning.devhub.local` sert toujours la V1.
5. L'équipe QA ou l'AnalysisRun teste la V2 sur `planning-preview.devhub.local`.
6. Si les tests passent (dans notre cas, l'AnalysisRun de 2 minutes avec 4 itérations réussit), Argo Rollouts effectue le swap :
   - Le sélecteur du Service `planning-planning-active` pointe désormais sur les pods Green.
   - Les utilisateurs de production basculent instantanément sur la V2 sans coupure.
7. L'ancien ReplicaSet (Blue) reste actif pendant 5 minutes (`scaleDownDelaySeconds: 300`). En cas de régression tardive, un `make rollout-abort` re-route instantanément la production sur le Blue encore chaud.

---


## Étape 9 — Routage avancé par headers HTTP

### 9.1 Cas d'Usage Métier

Le routage par en-tête HTTP ("Header-based routing") résout un besoin métier crucial : **valider en conditions réelles de production une nouvelle version applicative (Canary) par l'équipe interne (QA, développeurs, équipe produit) sans exposer le grand public**, et ce indépendamment de la répartition statistique du trafic (ex: même si le poids du Canary est configuré à 0%).

> *« Cela permettrait à l'équipe produit de tester chaque release sur leurs propres comptes (en injectant l'en-tête via une extension de navigateur) avant de l'ouvrir à n'importe quel utilisateur externe. »*

---

### 9.2 Configuration Technique dans le Rollout `annuaire`

Argo Rollouts s'interface avec le contrôleur Nginx Ingress pour lui injecter des annotations spécifiques sur l'Ingress Canary généré dynamiquement.

```yaml
# Extrait de platform-sre/services/annuaire/chart/values.yaml
rollout:
  canary:
    headerRouting:
      enabled: true
      header: "X-Beta-User"
      value: "true"
```

Cette configuration est interpolée dans le [`rollout.yaml`](file:///c:/Users/jammi/OneDrive%20-%20SUNELIS/Bureau/ESGI/TP%20Youssef/tp3-progressive-delivery/pulse-campus/services/annuaire/chart/templates/rollout.yaml#L96-L101) :
```yaml
      trafficRouting:
        nginx:
          stableIngress: {{ include "annuaire.fullname" . }}
          additionalIngressAnnotations:
            canary-by-header: "X-Beta-User"
            canary-by-header-value: "true"
```

Le contrôleur Ingress Nginx inspecte les en-têtes de chaque requête HTTP entrante. Si une requête porte l'en-tête `X-Beta-User: true`, Nginx contourne la règle de répartition de poids et l'adresse à 100% au service Canary.

---

### 9.3 Démonstration via `curl`

Voici les commandes pour vérifier le routage sélectif depuis le cluster local :

```bash
# ── Scénario A : Requête classique (Sans en-tête de test) ─────────────────────
curl -s -i http://annuaire.devhub.local/readyz | grep -E "HTTP/|X-Version"
# Sortie attendue (redirigée vers la version stable) :
#   HTTP/1.1 200 OK
#   X-Version: v1.0.0

# ── Scénario B : Requête Interne / Bêta (Avec en-tête X-Beta-User: true) ──────
curl -s -i -H "X-Beta-User: true" http://annuaire.devhub.local/readyz | grep -E "HTTP/|X-Version"
# Sortie attendue (redirigée vers la version Canary) :
#   HTTP/1.1 200 OK
#   X-Version: v1.1.0-canary
```

---

### 9.4 Intégration dans un Workflow de Delivery Produit

Cette technique s'intègre parfaitement avec des tests automatiques d'intégration :

1. **Promotion en mode "Mute"** : Argo Rollouts démarre le Canary à `setWeight: 0` mais avec le routage par header activé.
2. **Exécution des Tests E2E / Fumeurs (Smoke Tests)** : Une suite de tests automatisés (ex: Selenium, Playwright, ou des requêtes HTTP avec `curl`) appelle l'API publique `http://annuaire.devhub.local` en passant l'en-tête `X-Beta-User: true`.
3. **Analyse Automatique (AnalysisTemplate)** : L'AnalysisTemplate interroge Prometheus en ciblant uniquement le trafic du Canary (via `rollouts_pod_template_hash`).
4. **Bascule progressive** : Si les tests internes et l'AnalysisTemplate valident la stabilité sur ce panel restreint, le rollout passe au step suivant en ouvrant le trafic public par paliers statistiques (25%, 50%, 100%).

### 9.5 Règle de Nginx Ingress : "Le Header gagne"

> [!WARNING]
> **Le comportement n'est pas cumulatif**. Si une requête correspond aux conditions du header `X-Beta-User: true`, le contrôleur Ingress Nginx l'achemine au Canary **quel que soit le poids courant du rollout** (même à 0%). À l'inverse, si l'en-tête n'est pas présent ou ne correspond pas à la valeur attendue, la requête suit la répartition de poids aléatoire classique (ex: 25% Canary / 75% Stable).

---



## Étape 10 — Alerting & Notifications

Dans cette étape, nous mettons en place le système de détection des pannes et de routage des alertes de production (Prometheus et Alertmanager), ainsi que le reporting automatique des déploiements (Argo Rollouts).

---

### 10.1 Alertes Prometheus — Règles de Détection

Nous avons enrichi les règles de métrologie dans [`recording-rules.yaml`](file:///c:/Users/jammi/OneDrive%20-%20SUNELIS/Bureau/ESGI/TP%20Youssef/tp3-progressive-delivery/pulse-campus/platform-sre/recording-rules/recording-rules.yaml) avec un groupe dédié d'alertes Prometheus (`pulse_campus_alerting_rules`).

#### 1. Alerte Taux d'Erreur (Sévérité `page`)
Déclenchée si le taux d'erreur 5xx d'un service dépasse 1% pendant 5 minutes.
- **Requête PromQL** : `sli:http_request_error_ratio:rate5m > 0.01`
- **Fenêtre d'activation (`for`)** : `5m` (amortit les pics transitoires).
- **Label** : `severity: page` (incident critique, réveille l'ingénieur d'astreinte).

#### 2. Alerte Latence Dégradée (Sévérité `ticket`)
Déclenchée si la latence p95 dépasse le seuil SLO défini pour le service pendant 30 minutes.
- **Requêtes PromQL** (exemple pour `annuaire`, seuil 300 ms) :
  ```promql
  histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket{job="annuaire", route!~"/healthz|/readyz|/metrics"}[5m]))) > 0.3
  ```
- **Fenêtre d'activation (`for`)** : `30m` (évite de réveiller pour des ralentissements courts).
- **Label** : `severity: ticket` (non-urgent, traité le lendemain).

---

### 10.2 Routage Gradué Alertmanager

Alertmanager est configuré dans le fichier [`kube-prometheus-stack-values.yaml`](file:///c:/Users/jammi/OneDrive%20-%20SUNELIS/Bureau/ESGI/TP%20Youssef/tp3-progressive-delivery/pulse-campus/platform-sre/values/kube-prometheus-stack-values.yaml) pour acheminer différemment les alertes selon leur niveau d'urgence :

```yaml
# Extrait de platform-sre/values/kube-prometheus-stack-values.yaml
alertmanager:
  config:
    route:
      group_by: ['alertname', 'job', 'namespace']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      receiver: 'null'
      routes:
        # Route Page -> webhook-primary (SRE d'astreinte)
        - matchers:
            - severity = page
          receiver: 'webhook-primary'
        # Route Ticket -> webhook-secondary (QA / non-urgent)
        - matchers:
            - severity = ticket
          receiver: 'webhook-secondary'
    receivers:
      - name: 'null'
      - name: 'webhook-primary'
        webhook_configs:
          - url: 'https://webhook.site/placeholder-primary-sre-url'
            send_resolved: true
      - name: 'webhook-secondary'
        webhook_configs:
          - url: 'https://webhook.site/placeholder-secondary-sre-url'
            send_resolved: true
```

---

### 10.3 Notifications Automatiques d'Argo Rollouts

Le contrôleur de notifications d'Argo Rollouts a été configuré dans [`argo-rollouts-values.yaml`](file:///c:/Users/jammi/OneDrive%20-%20SUNELIS/Bureau/ESGI/TP%20Youssef/tp3-progressive-delivery/pulse-campus/platform-sre/values/argo-rollouts-values.yaml#L20) pour générer des webhooks de fin de cycle de vie de déploiement.

#### 1. Configuration des Services, Templates et Triggers
- **Triggers configurés** :
  - `on-rollout-completed` : Se déclenche quand la phase passe à `Healthy` (succès).
  - `on-rollout-aborted` : Se déclenche quand la phase passe à `Degraded` (échec).
- **Templates JSON structurés** :
  - **`rollout-success`** : Envoyé vers `webhook-product` (canal produit).
  - **`rollout-failure`** : Envoyé vers `webhook-primary` (le même que pour les pages d'astreinte).

```yaml
# Configuration dans argo-rollouts-values.yaml
notifications:
  secret:
    create: true
  configmap:
    create: true
  notifiers:
    webhook:
      webhook-primary:
        url: https://webhook.site/placeholder-primary-rollouts-url
      webhook-product:
        url: https://webhook.site/placeholder-product-rollouts-url
  templates:
    # Succès
    rollout-success:
      webhook:
        webhook-product:
          method: POST
          body: |
            {
              "rollout": "{{.rollout.metadata.name}}",
              "conclusion": "Promoted",
              "phase": "{{.rollout.status.phase}}",
              "image": "{{(index .rollout.spec.template.spec.containers 0).image}}",
              "stable_revision": "{{.rollout.status.stableRS}}",
              "duration": "5 minutes (analysis + pauses)"
            }
    # Échec (Abort)
    rollout-failure:
      webhook:
        webhook-primary:
          method: POST
          body: |
            {
              "rollout": "{{.rollout.metadata.name}}",
              "conclusion": "Aborted",
              "phase": "{{.rollout.status.phase}}",
              "image": "{{(index .rollout.spec.template.spec.containers 0).image}}",
              "stable_revision": "{{.rollout.status.stableRS}}",
              "failed_revision": "{{.rollout.status.currentPodHash}}",
              "duration": "analysis failed (auto-rollback triggered)"
            }
```

#### 2. Abonnement des Rollouts
Les ressources `Rollout` s'abonnent aux notifications via des annotations dans les métadonnées (présentes dans les templates de `annuaire` et `planning`) :
```yaml
metadata:
  annotations:
    # Réussite -> webhook produit
    notifications.argoproj.io/subscribe.on-rollout-completed.webhook-product: ""
    # Échec -> webhook d'astreinte primaire
    notifications.argoproj.io/subscribe.on-rollout-aborted.webhook-primary: ""
```

---


## Étape 11 — Comparer Argo Rollouts, Flagger, et la rolling-update native

### 11.1 Matrice de Comparaison Comparative

Cette matrice évalue chaque solution sur une échelle de **0 (inexistant / très mauvais)** à **5 (excellent / natif)** :

| Critère | RollingUpdate native | Argo Rollouts (projet Argo) | Flagger (écosystème Flux) |
|---|:---:|:---:|:---:|
| **Courbe d'apprentissage** | **5/5** : Natif, maîtrisé par tout développeur K8s. | **4/5** : Modèle déclaratif simple, remplace juste le Deployment. | **2/5** : Plus complexe, nécessite de gérer les Custom Resources Custom et des Ingress avancés. |
| **Intégration ArgoCD (GitOps)** | **5/5** : Parfaite, support natif. | **5/5** : Parfaite, même écosystème, interface ArgoCD intégrée. | **3/5** : Fonctionne, mais risque de conflits de réconciliation si non configuré en IgnoreDifferences. |
| **Intégration Flux (GitOps)** | **5/5** : Parfaite. | **3/5** : Moins naturelle, pas de console d'administration unifiée. | **5/5** : Parfaite, s'intègre nativement dans le cycle de réconciliation de Flux. |
| **Variété des stratégies** | **1/5** : Uniquement RollingUpdate basique (pas de Canary contrôlé ni de B/G). | **5/5** : Complet (Canary, Blue/Green, routage par header/cookie). | **4/5** : Complet (Canary, A/B testing, Traffic Mirroring), moins de B/G pur. |
| **Variété des metric providers** | **0/5** : Aucun (ne lit pas de métriques externes). | **5/5** : Très riche (Prometheus, Datadog, NewRelic, CloudWatch, Wavefront, Webhooks). | **4/5** : Très riche (Prometheus, Datadog, CloudWatch, Graphite), via l'outil Linkerd/Istio. |
| **UI / Dashboard prêt à l'emploi** | **1/5** : Uniquement la CLI kubectl de base ou Kubernetes Dashboard. | **5/5** : Console Web dédiée (`kubectl-argo-rollouts dashboard`) claire et visuelle. | **1/5** : Pas d'UI officielle (uniquement logs ou dashboards Grafana tiers). |
| **Coût opérationnel dans le cluster** | **5/5** : Nul (géré directement par le kube-controller-manager natif). | **4/5** : Faible (un seul pod contrôleur léger). | **3/5** : Moyen (nécessite souvent d'installer un Service Mesh type Linkerd ou Istio). |
| **Adapté à un Service Mesh** | **2/5** : Ne gère pas le routage fin au niveau L7 du Mesh. | **4/5** : Très bon (Linkerd, Istio, Traefik, Nginx, APISIX). | **5/5** : Excellent (conçu nativement pour s'appuyer sur Istio, Linkerd ou AppMesh). |
| **Communauté / Releases** | **5/5** : Le cœur de Kubernetes. | **5/5** : Projet CNCF Graduated très actif, fort soutien d'Intuit. | **4/5** : Projet CNCF Incubating, soutenu par Weaveworks (historiquement) et la communauté Flux. |
| **Risque si le contrôleur tombe** | **5/5** : Nul (le control plane K8s est hautement disponible). | **4/5** : Faible (les pods existants continuent de tourner, la promotion est juste figée). | **4/5** : Faible (le routage reste sur l'état courant, pas d'interruption). |

---

### 11.2 Analyse Technique SRE des Écarts (Justification des Notes)

#### 1. Pourquoi la RollingUpdate native obtient 0/5 en Metric Providers ?
La RollingUpdate native de Kubernetes est un automate aveugle. Elle ne prend sa décision de progression que sur la base des `readinessProbes` réseau et processus. Si une application démarre correctement (200 OK sur `/readyz`) mais corrompt les requêtes métier réelles (500 sur les endpoints utiles suite à un bug de schéma DB), la RollingUpdate propagera la panne à 100% de la flotte sans possibilité de s'arrêter ou d'effectuer un rollback automatique. Elle ne sait pas lire Prometheus.

#### 2. Pourquoi Flagger est noté 2/5 en Courbe d'Apprentissage ?
Contrairement à Argo Rollouts qui remplace la ressource `Deployment` par `Rollout` de manière intuitive, Flagger conserve le `Deployment` natif mais génère lui-même un déploiement secondaire "primaire" et manipule les sélecteurs de manière implicite. Pour un développeur, ce comportement de duplication de ressources en tâche de fond est difficile à appréhender et à déboguer au début. De plus, il impose presque systématiquement l'apprentissage d'un Service Mesh (Istio/Linkerd) pour découper le trafic L7, là où Argo Rollouts sait utiliser un simple Ingress Nginx existant.

#### 3. Pourquoi Argo Rollouts obtient 5/5 en UI / Dashboard ?
L'outil fournit le plugin `kubectl argo rollouts` qui embarque un serveur Web local complet (`kubectl argo rollouts dashboard -n devhub-dev`). Cette interface graphique liste tous les rollouts, montre les ReplicaSets actifs, affiche le poids du trafic en temps réel, permet de cliquer pour promouvoir ou annuler (abort) un canari, et intègre directement les logs des `AnalysisRun`. Pour les équipes opérationnelles (SRE et QA), ce niveau de visibilité réduit drastiquement le MTTR (temps de résolution) lors d'un incident de release.

---


## Étape 12 — Synthèse d'Architecture & Rétrospective

### 12.1 Livrable 1 — Rétrospective TP 2 → TP 3 : « Le même geste, trois paradigmes »

#### Rétrospective par ligne du tableau comparatif :
- **Déployer une nouvelle version** : La colonne TP 3 apporte une grande sérénité opérationnelle; la libération d'une version n'est plus un acte de foi mais une progression logique basée sur des preuves mathématiques d'observabilité.
- **Détecter qu'une version dégrade la prod** : C'est le jour et la nuit; nous passons de la dépendance passive aux plaintes des utilisateurs (TP 1 / TP 2) à une détection proactive et automatisée en moins de 30 secondes par Prometheus.
- **Faire un rollback** : Le rollback automatisé par le contrôleur (ou en une commande `make rollout-abort`) est infiniment plus rassurant que le stress d'un `git revert` dans l'urgence qui doit repasser par toute la pipeline CI/CD.
- **Limiter l'impact d'une mauvaise version** : Le fait de cantonner une anomalie à seulement 10% ou 25% du trafic utilisateur (Canary) pendant les phases de test est la brique de protection ultime pour l'image de marque de l'entreprise.
- **Savoir si le service tient son SLO** : Le passage de l'obscurité totale à la visibilité via un dashboard RED et des alertes graduées donne enfin une mesure objective et contractuelle de la qualité de service.
- **Décider de promouvoir une release** : On élimine le biais humain et le fameux « au feeling » pour le remplacer par une décision purement factuelle (comparaison des métriques réelles face aux seuils SLO).
- **Justifier un déploiement le vendredi à 17h** : TP 3 le rend possible car le rayon d'impact est maîtrisé à 10% et l'annulation est instantanée sans perte de paquets, éliminant la peur irrationnelle du déploiement.
- **Mesurer la fréquence et le taux d'échec (DORA)** : Obtenir ces métriques de manière native via les objets `Rollout` permet d'ancrer l'équipe dans une démarche d'amélioration continue et quantifiable.
- **Tester une version sur un cohort spécifique** : Le routage par header HTTP (Étape 9) offre une flexibilité incroyable pour valider les releases sur nos propres comptes de production sans impacter le public.

#### Justification économique pour une startup de 3 personnes :
Pour une structure très jeune (3 personnes), le surcoût opérationnel du TP 3 **n'est pas justifié** pour les deux opérations suivantes :
1. **L'écriture des `AnalysisTemplate` automatiques** : Développer et maintenir des requêtes PromQL complexes de rollback automatique coûte cher en temps d'ingénierie. Une simple bascule manuelle progressive supervisée visuellement sur Grafana pendant 10 minutes suffit amplement.
2. **La mise en place de la double stack `active`/`preview` avec routage de header** : Le coût de doublement des ressources (CPU/RAM pour avoir 2 versions complètes en parallèle) et la complexité des manifestes Ingress dépassent le gain pour un produit qui n'a pas encore de contraintes de haute disponibilité forte.

#### L'opération justifiant à elle seule le passage au TP 3 en PME en croissance :
**La détection et le rollback automatiques basés sur les métriques (AnalysisTemplate)**. En phase de croissance, le volume de releases augmente exponentiellement et les développeurs ne peuvent plus surveiller manuellement chaque déploiement. Automatiser cette garde-barrière évite les incidents majeurs en production, protégeant le chiffre d'affaires et la réputation de l'entreprise tout en libérant du temps pour l'équipe SRE/DevOps.

---

### 12.2 Livrable 2 — Ce que cette chaîne ne sait toujours pas faire (Failles & Remédiations)

#### 1. Traçabilité distribuée (Tracing)
- **Le risque** : Si le service `planning` ralentit, nous voyons sur Grafana que la latence globale augmente, mais il est impossible de savoir si le goulot d'étranglement vient d'un appel réseau lent vers `annuaire`, d'une requête SQL lente sur sa propre base, ou d'un blocage de l'event-loop.
- **L'outil complémentaire** : L'implémentation de la spécification **OpenTelemetry** dans le code des services pour propager un `traceparent` HTTP, couplé à **Jaeger** ou **Grafana Tempo** pour stocker et visualiser les traces distribuées.
- **Référence** : [OpenTelemetry Python Instrumentation Guide](https://opentelemetry.io/docs/languages/python/automatic/)

#### 2. Centralisation et corrélation des Logs
- **Le risque** : En cas d'erreur 500 sur le Canary, nous devons ouvrir un terminal et lire les flux de logs via `kubectl logs` sur plusieurs pods pour comprendre l'exception. Il n'y a aucun lien direct entre le pic d'erreur sur Grafana et la ligne de log exacte qui l'a causé.
- **L'outil complémentaire** : Déployer **Grafana Loki** avec **Promtail** (ou FluentBit) pour ingérer les logs, et utiliser les **Exemplars** Prometheus dans Grafana pour cliquer sur un point de métrique et ouvrir directement les logs du pod associé.
- **Référence** : [Grafana Loki Architecture & Configuration](https://grafana.com/docs/loki/latest/get-started/)

#### 3. Mesure de l'expérience utilisateur réelle (RUM)
- **Le risque** : Prometheus mesure la latence côté serveur. Si le navigateur de l'étudiant met 5 secondes à charger l'emploi du temps à cause d'un script JS lourd ou d'une mauvaise négociation TLS sur son réseau, le serveur enregistre 50ms (SLO vert), mais l'utilisateur vit une expérience désastreuse (SLO rouge en réalité).
- **L'outil complémentaire** : Intégrer un agent **RUM (Real User Monitoring)** comme OpenTelemetry JS ou Grafana Faro dans l'application frontend, afin de remonter les **Web Vitals** (LCP, FID, CLS) directement depuis le navigateur.
- **Référence** : [Google Web Vitals specs](https://web.dev/explore/vitals)

#### 4. Chaos Engineering applicatif
- **Le risque** : Nos alertes et notre logique de rollback automatique n'ont été testées que sur des scénarios nominaux et des pannes simulées simplement. Nous ne savons pas comment se comporte la plateforme si un nœud Kubernetes tombe, si le réseau subit 50% de perte de paquets, ou si l'API d'un tiers se met à répondre avec 10s de retard.
- **L'outil complémentaire** : Installer **Chaos Mesh** ou **LitmusChaos** pour injecter régulièrement et de manière contrôlée des pannes en production (pod kills, network latency) afin de valider la robustesse de notre boucle de rétroaction SRE.
- **Référence** : [Chaos Mesh documentation](https://chaos-mesh.org/docs/)

#### 5. Politique d'admission des manifests (Governance)
- **Le risque** : Un développeur peut pousser un manifeste Helm avec un conteneur configuré en `privileged: true`, sans limites de ressources CPU/RAM (surchargeant le cluster), ou oubliant d'activer le ServiceMonitor. ArgoCD va synchroniser cette configuration dangereuse sans contrôle préalable.
- **L'outil complémentaire** : Déployer **Kyverno** ou **OPA Gatekeeper** en tant que Validating Admission Webhooks pour bloquer à l'entrée du cluster tout manifeste non conforme aux règles de sécurité (ex: interdiction du mode root, limits obligatoires).
- **Référence** : [Kyverno Policy Library](https://kyverno.io/policies/)

#### 6. Signature et Provenance des Images (Supply Chain)
- **Le risque** : Une personne malveillante parvenant à pirater notre registre d'images (GHCR) pourrait écraser le tag `dev` avec une image compromise contenant un malware. Kubernetes va puller et exécuter cette image sans vérifier si elle provient bien de notre pipeline de build CI/CD légitime.
- **L'outil complémentaire** : Signer les images Docker dans la CI GitHub Actions avec **Cosign (Sigstore)**, et utiliser le contrôleur d'admission **Kyverno** pour rejeter tout pod dont l'image n'est pas signée par notre clé privée d'entreprise.
- **Référence** : [Sigstore Cosign documentation](https://docs.sigstore.dev/cosign/overview/)

#### 7. Sauvegarde applicative et Disaster Recovery (DR)
- **Le risque** : Si le cluster Kind local (ou un cluster de production hébergé) subit une corruption majeure d'etcd ou si l'hyperviseur physique lâche, nous perdons l'état du cluster, les configurations, et les états d'ArgoCD. Reconstruire la plateforme à la main prendrait plusieurs heures.
- **L'outil complémentaire** : Configurer **Velero** pour effectuer des sauvegardes planifiées des ressources de l'API Kubernetes et des snapshots des volumes persistants (PVC) vers un stockage objet compatible S3 (ex: MinIO en local, AWS S3 en prod).
- **Référence** : [Velero Disaster Recovery Guide](https://velero.io/docs/v1.12/)

---

### 12.3 Livrable 3 — Prise de Position d'Architecte Plateforme

> **Ma posture d'architecte face à une stack pour 10 services et 30 développeurs :**
>
> Pour cette taille d'équipe (30 développeurs) et de flotte (10 microservices), je **garde** la colonne vertébrale GitOps : **ArgoCD** en motif *App of Apps* pour centraliser les déploiements, couplé à **Argo Rollouts** qui apporte une autonomie totale et sécurisante aux équipes de dev grâce à la promotion visuelle.
> Cependant, je **remplace** la gestion des secrets brute du TP (externalisation manuelle) par **External Secrets Operator** connecté à un coffre-fort centralisé (ex: HashiCorp Vault ou AWS Secrets Manager) pour éliminer les copier-coller manuels de secrets.
> Enfin, j'**ajoute** immédiatement **Kyverno** pour interdire la mise en production de manifestes non sécurisés, et j'ajoute **OpenTelemetry** pour le tracing distribué, car à 10 microservices interconnectés, déboguer un goulot d'étranglement de latence réseau sans traces est impossible. J'évite d'ajouter un Service Mesh complet (trop complexe pour 10 services), et je conserve le découpage de trafic statistique simple basé sur **Nginx Ingress** qui est ultra-léger et suffisant.
