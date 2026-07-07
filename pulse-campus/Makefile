SHELL := /bin/bash
CLUSTER ?= pulse
ARGOCD_CHART_VERSION ?= 7.6.12
INGRESS_CHART_VERSION ?= 4.11.3

SERVICES := annuaire planning notif
SHA := $(shell git rev-parse --short HEAD 2>/dev/null || echo dev)
# Adaptez GHCR_USER à votre compte GitHub.
GHCR_USER ?= changeme

.PHONY: help tools-check tools-check-tp3 promtool-check helm-lint-all \
        rollout-status rollout-promote rollout-abort rollout-watch \
        cluster-up cluster-down argocd-install argocd-password \
        hosts-print helm-lint images-build images-load clean

help:
	@echo "Cibles disponibles :"
	@echo "  tools-check       - vérifie les CLI requises (TP 2 + TP 3 basique)"
	@echo "  tools-check-tp3   - vérification complète TP3 [OK]/[KO] avec exit 1"
	@echo "                      (docker, kubectl, helm, kind, argocd, rollouts,"
	@echo "                       promtool, amtool, jq)"
	@echo "  promtool-check    - valide toutes les PrometheusRule YAML avec promtool"
	@echo "  helm-lint-all     - helm lint sur les 3 charts (avec values-dev.yaml)"
	@echo "  rollout-status    - kubectl argo rollouts status annuaire + planning"
	@echo "  rollout-promote   - promeut le palier canary courant (annuaire)"
	@echo "  rollout-abort     - annule le rollout en cours et retourne au stable"
	@echo "  rollout-watch     - stream du status rollout en temps réel"
	@echo "  cluster-up        - démarre un cluster kind 2 nœuds"
	@echo "  cluster-down      - détruit le cluster"
	@echo "  argocd-install    - installe ingress-nginx + ArgoCD (Helm)"
	@echo "  argocd-password   - affiche le mot de passe admin initial d'ArgoCD"
	@echo "  hosts-print       - lignes à ajouter dans /etc/hosts"
	@echo "  helm-lint         - helm lint sur les charts des trois services"
	@echo "  images-build      - build des trois images applicatives (tag SHA)"
	@echo "  images-load       - kind load des trois images dans le cluster"
	@echo "  clean             - détruit le cluster et nettoie les artefacts locaux"

tools-check:
	@missing=0; \
	for t in docker kubectl helm kind argocd promtool; do \
		if ! command -v $$t >/dev/null 2>&1; then echo "manquant : $$t"; missing=1; fi; \
	done; \
	if ! kubectl argo rollouts version >/dev/null 2>&1; then \
		echo "manquant : plugin kubectl-argo-rollouts (kubectl argo rollouts)"; missing=1; \
	fi; \
	if [ $$missing -ne 0 ]; then exit 1; fi
	@echo "tous les outils sont présents."
	@docker version --format '  docker  : {{.Server.Version}}' 2>/dev/null || true
	@kubectl version --client 2>/dev/null | head -n1 | sed 's/^/  kubectl : /'
	@helm version --short | sed 's/^/  helm    : /'
	@kind version | sed 's/^/  /'
	@argocd version --client --short 2>/dev/null | sed 's/^/  /'
	@kubectl argo rollouts version 2>/dev/null | head -n1 | sed 's/^/  rollouts: /'
	@promtool --version 2>&1 | head -n1 | sed 's/^/  /'

# ──────────────────────────────────────────────────────────────────────
# tools-check-tp3 : vérification complète de l'outillage TP 3
#   Outils TP2 : docker, kubectl, helm, kind, argocd
#   Outils TP3 : kubectl-argo-rollouts, promtool, amtool, jq
#
#   Affiche [OK] / [KO] par outil et retourne exit 1 si au moins un
#   outil est absent, pour CI-gate dans le pipeline.
# ──────────────────────────────────────────────────────────────────────
tools-check-tp3:
	@echo "═══════════════════════════════════════════════════════"
	@echo "       tools-check-tp3 — TP Progressive Delivery"
	@echo "═══════════════════════════════════════════════════════"
	@overall=0; \
	_check() { \
		tool=$$1; shift; \
		if command -v $$tool >/dev/null 2>&1; then \
			ver=$$($$@ 2>&1 | head -n1); \
			printf "[OK] %-28s : %s\n" "$$tool" "$$ver"; \
		else \
			printf "[KO] %-28s : NON INSTALLE\n" "$$tool"; \
			overall=1; \
		fi; \
	}; \
	_check docker       docker version --format '{{.Server.Version}}'; \
	_check kubectl      kubectl version --client --output=yaml | grep 'gitVersion' | head -n1 | awk '{print $$2}'; \
	_check helm         helm version --short; \
	_check kind         kind version; \
	_check argocd       argocd version --client --short; \
	_check promtool     promtool --version; \
	_check amtool       amtool --version; \
	_check jq           jq --version; \
	if kubectl argo rollouts version >/dev/null 2>&1; then \
		ver=$$(kubectl argo rollouts version 2>&1 | head -n1); \
		printf "[OK] %-28s : %s\n" "kubectl-argo-rollouts" "$$ver"; \
	else \
		printf "[KO] %-28s : NON INSTALLE\n" "kubectl-argo-rollouts"; \
		overall=1; \
	fi; \
	echo "───────────────────────────────────────────────────────"; \
	if [ $$overall -eq 0 ]; then \
		echo "Tous les outils requis sont présents. TP 3 prêt."; \
	else \
		echo "ERREUR : des outils manquent (voir [KO] ci-dessus)."; \
		echo "  kubectl-argo-rollouts : https://github.com/argoproj/argo-rollouts/releases"; \
		echo "  promtool/amtool       : https://github.com/prometheus/prometheus/releases"; \
		echo "  jq                    : https://jqlang.github.io/jq/download/"; \
		exit 1; \
	fi

# ──────────────────────────────────────────────────────────────────────
# promtool-check : valide la syntaxe PromQL de toutes les PrometheusRule
#   Utilise `promtool check rules` qui retourne exit 1 si une expression
#   est syntaxiquement invalide, même si le cluster n'est pas démarré.
#   Idéal comme gate de CI avant git push.
# ──────────────────────────────────────────────────────────────────────
promtool-check:
	@echo "══════════════════════════════════════════════════════"
	@echo "  promtool check rules — Pulse Campus SRE"
	@echo "══════════════════════════════════════════════════════"
	@failed=0; \
	for f in $$(find platform-sre -name '*.yaml' | xargs grep -l 'kind: PrometheusRule' 2>/dev/null); do \
		echo "  >>> checking $$f"; \
		promtool check rules "$$f" || { echo "  [KO] $$f"; failed=1; }; \
		[ $$failed -eq 0 ] && echo "  [OK] $$f"; \
	done; \
	echo "──────────────────────────────────────────────────────"; \
	if [ $$failed -eq 0 ]; then \
		echo "Toutes les PrometheusRule sont syntaxiquement valides."; \
	else \
		echo "ERREUR : des règles invalides ont été détectées."; \
		exit 1; \
	fi

# ──────────────────────────────────────────────────────────────────────
# helm-lint-all : lint les 3 charts avec values-dev.yaml (plus proche prod)
# ──────────────────────────────────────────────────────────────────────
helm-lint-all:
	@for svc in $(SERVICES); do \
		echo ">>> helm lint $$svc (values-dev.yaml)"; \
		helm lint services/$$svc/chart \
			--values services/$$svc/chart/values.yaml \
			--values services/$$svc/chart/values-dev.yaml || exit 1; \
	done

# ──────────────────────────────────────────────────────────────────────
# Cibles Argo Rollouts (Étape 5)
# Utilisation :
#   make rollout-status    → affiche le status des 2 Rollouts
#   make rollout-promote   → promeut le palier canary annuaire (et optionnellement planning)
#   make rollout-abort     → revient au stable (abort le rollout en cours)
#   make rollout-watch     → flux en temps réel (dépend de `watch`)
#
# Le namespace est surchargeble : make rollout-status NS=devhub-staging
# ──────────────────────────────────────────────────────────────────────
NS ?= devhub-dev

rollout-status:
	@echo "═══════════════════════════════════════════════════"
	@echo "  Rollout Status — Pulse Campus (namespace: $(NS))"
	@echo "═══════════════════════════════════════════════════"
	@kubectl argo rollouts status annuaire-annuaire -n $(NS) || true
	@echo "─────────────────────────────────────────────────"
	@kubectl argo rollouts status planning-planning -n $(NS) || true

rollout-promote:
	@echo ">>> Promotion palier canary annuaire ($(NS))"
	kubectl argo rollouts promote annuaire-annuaire -n $(NS)
	@echo ">>> Promotion palier canary planning ($(NS))"
	kubectl argo rollouts promote planning-planning -n $(NS)

rollout-abort:
	@echo ">>> ABORT rollout annuaire — retour au Stable ($(NS))"
	kubectl argo rollouts abort annuaire-annuaire -n $(NS)
	@echo ">>> ABORT rollout planning — retour au Stable ($(NS))"
	kubectl argo rollouts abort planning-planning -n $(NS)

rollout-watch:
	@echo ">>> Watch rollout annuaire (Ctrl+C pour arrêter)"
	kubectl argo rollouts get rollout annuaire-annuaire -n $(NS) --watch

cluster-up:

	kind create cluster --name $(CLUSTER) --config cluster/kind-config.yaml
	@echo ">>> cluster prêt — contexte courant : kind-$(CLUSTER)"

cluster-down:
	kind delete cluster --name $(CLUSTER)

argocd-install:
	helm repo add argo https://argoproj.github.io/argo-helm
	helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
	helm repo update
	helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
		--namespace ingress-nginx --create-namespace \
		--version $(INGRESS_CHART_VERSION) \
		--set controller.service.type=NodePort \
		--set-string controller.nodeSelector."ingress-ready"=true \
		--set "controller.tolerations[0].key=node-role.kubernetes.io/control-plane" \
		--set "controller.tolerations[0].operator=Exists" \
		--set "controller.tolerations[0].effect=NoSchedule" \
		--set controller.hostPort.enabled=true \
		--set controller.publishService.enabled=false
	kubectl wait --namespace ingress-nginx \
		--for=condition=ready pod \
		--selector=app.kubernetes.io/component=controller \
		--timeout=180s
	helm upgrade --install argocd argo/argo-cd \
		--namespace argocd --create-namespace \
		--version $(ARGOCD_CHART_VERSION)

argocd-password:
	@kubectl -n argocd get secret argocd-initial-admin-secret \
		-o jsonpath='{.data.password}' | base64 -d ; echo

hosts-print:
	@echo "ajoutez ces lignes à votre fichier hosts :"
	@echo "  macOS/Linux : /etc/hosts"
	@echo "  Windows     : C:\\Windows\\System32\\drivers\\etc\\hosts"
	@echo ""
	@echo "127.0.0.1  argocd.devhub.local"
	@echo "127.0.0.1  grafana.devhub.local"
	@echo "127.0.0.1  prometheus.devhub.local"
	@echo "127.0.0.1  rollouts.devhub.local"
	@echo "127.0.0.1  annuaire.devhub.local"
	@echo "127.0.0.1  planning.devhub.local"
	@echo "127.0.0.1  notif.devhub.local"

helm-lint:
	@for svc in $(SERVICES); do \
		echo ">>> helm lint $$svc"; \
		helm lint services/$$svc/chart || exit 1; \
	done

images-build:
	@for svc in $(SERVICES); do \
		echo ">>> docker build $$svc:$(SHA)"; \
		docker build -t ghcr.io/$(GHCR_USER)/$$svc:$(SHA) services/$$svc; \
	done

images-load:
	@for svc in $(SERVICES); do \
		echo ">>> kind load ghcr.io/$(GHCR_USER)/$$svc:$(SHA)"; \
		kind load docker-image ghcr.io/$(GHCR_USER)/$$svc:$(SHA) --name $(CLUSTER); \
	done

clean:
	kind delete cluster --name $(CLUSTER) 2>/dev/null || true
