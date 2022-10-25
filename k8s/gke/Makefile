include $(dir $(word $(words $(MAKEFILE_LIST)),$(MAKEFILE_LIST)))/../Makefile

CLUSTER := rabbitmq-workshop-$(USER)
# Find out what versions are available: make k8s-versions
# K8S versions valid at 04 December 2020
# CLUSTER_VERSION ?= 1.23.12-gke.100
# CLUSTER_NODE_VERSION ?=1.23.12-gke.100
CLUSTER_RELEASES ?= stable
# CLUSTER_NODE_TYPE ?= n2-standard-4
# CLUSTER_NODES_PER_ZONE ?= 2

# You may want to overwrite this with your GCP project, e.g. export GCP_PROJECT=my-project-name
GCP_PROJECT ?= rabbitmq-workshop
# You may want to overwrite this with your preferred GCP region, e.g. export GCP_REGION=us-east1
GCP_REGION ?= europe-west1

# https://github.com/rabbitmq/cluster-operator/releases
RABBITMQ_OPERATOR_VERSION := v1.14.0

KUBECONFIG_DIR := $(XDG_CONFIG_HOME)/kubectl
KUBECONFIG := $(KUBECONFIG_DIR)/config
export KUBECONFIG

CLOUDSDK_CONFIG := $(XDG_CONFIG_HOME)/gcloud/configurations/config_default
export CLOUDSDK_CONFIG
$(CLOUDSDK_CONFIG): $(GCLOUD)
	$(GCLOUD) auth login \
	&& $(GCLOUD) config set project $(GCP_PROJECT) \
	&& $(GCLOUD) config set compute/region $(GCP_REGION)

$(KUBECONFIG_DIR):
	mkdir -p $(@)
$(KUBECONFIG): | $(KUBECTL) $(KUBECONFIG_DIR) $(CLOUDSDK_CONFIG)
	$(GCLOUD) container clusters get-credentials $(CLUSTER)

.PHONY: k9s
k9s: | $(K9S) $(KUBECONFIG) ## Interact with our K8S cluster via a terminal UI
	$(K9S) --all-namespaces

.PHONY: k9
k9: | $(K9S) $(KUBECONFIG)
	$(K9S) --namespace default --headless

define ENV
export PATH=$(LOCAL_BIN):$$PATH
export GCP_PROJECT="$(GCP_PROJECT)"
export GCP_REGION="$(GCP_REGION)"
export KUBECONFIG="$(KUBECONFIG)"
export XDG_CONFIG_HOME="$(XDG_CONFIG_HOME)"
export CLOUDSDK_CONFIG="$(CLOUDSDK_CONFIG)"
unalias k 2>/dev/null; alias k=kubectl
unalias m 2>/dev/null; alias m=make
endef
export ENV
.PHONY: env
env:: | $(CLOUDSDK_CONFIG) $(KUBECONFIG_DIR) $(KUBECTL) ## Configure shell env - eval "$(make env)" OR source .env
	@echo "$$ENV"

define LIST_INSTANCES
$(GCLOUD) compute instances list --filter='name ~ $(CLUSTER)'
endef
instances: | $(CLOUDSDK_CONFIG) ## List all instances
	$(LIST_INSTANCES)

watch-instances: | $(CLOUDSDK_CONFIG) ## Watch all instances
	watch -c "$(LIST_INSTANCES)"

watch-nodes: | $(KUBECONFIG) ## Watch all K8S nodes
	watch -c "$(KUBECTL) get nodes --output=wide"

disks: | $(CLOUDSDK_CONFIG) ## List all disks
	$(GCLOUD) compute disks list --filter='name ~ $(CLUSTER)'

.PHONY: k8s-versions
k8s-versions: | $(CLOUDSDK_CONFIG) ## List all available K8S versions on GCP (GKE)
	$(GCLOUD) container get-server-config

.PHONY: k8s
k8s: | $(CLOUDSDK_CONFIG) ## Create a managed K8S cluster on GCP (GKE) - up to 4 minutes
	$(GCLOUD) container clusters describe $(CLUSTER) \
	|| time $(GCLOUD) container clusters create-auto $(CLUSTER) \
	   --release-channel $(CLUSTER_RELEASES) \
	   --network "projects/$(GCP_PROJECT)/global/networks/default" \
	   --subnetwork "projects/$(GCP_PROJECT)/regions/$(GCP_REGION)/subnetworks/default" \
	   --cluster-ipv4-cidr "/17" \
	   --services-ipv4-cidr "/22"

.PHONY: base
base: | k8s monitoring-stack rabbitmq-operator ## Create the cluster & all stacks needed for the episode in GKE, except for Chaos Mesh

.PHONY: k8s-ls
k8s-ls: | $(CLOUDSDK_CONFIG) ## List all GKE clusters running on GCP
	$(GCLOUD) container clusters list

.PHONY: k8s-rm
k8s-rm: | $(CLOUDSDK_CONFIG) ## Delete our GKE cluster
	$(GCLOUD) container clusters delete $(CLUSTER)

# helm search repo --versions prometheus-community/kube-prometheus-stack
KUBE_PROMETHEUS_STACK_VERSION ?= 41.5.1

ADMIN_EMAIL:= $(shell gcloud info --format='value(config.account)')
.PHONY: monitoring-stack
monitoring-stack: | $(KUBECONFIG) $(HELM) ## Integrate Prometheus & Grafana with K8S, including system metrics
	$(KUBECTL) create clusterrolebinding owner-cluster-admin-binding --clusterrole cluster-admin --user $(ADMIN_EMAIL)
	$(KUBECTL) create namespace grafana-stack
	$(HELM) repo add prometheus-community https://prometheus-community.github.io/helm-charts
	$(HELM) repo update
	$(HELM) install grafana-stack prometheus-community/kube-prometheus-stack --namespace grafana-stack --set coreDns.enabled=false --set kubeControllerManager.enabled=false --set kubeDns.enabled=false --set kubeEtcd.enabled=false --set kubeProxy.enabled=false --set kubeScheduler.enabled=false --set nodeExporter.enabled=false --set alertmanager.enabled=false
	$(KUBECTL) apply \
	  --filename ../monitoring-stack/grafana-stack-roles.yaml \
	  --filename ../monitoring-stack/prometheus-auto-discovery.yaml

.PHONY: teardown-monitoring-stack
teardown-monitoring-stack: | $(KUBECONFIG) $(HELM) ## Teardown the whole monitoring stack
	$(KUBECTL) delete \
	  --filename ../monitoring-stack/grafana-stack-roles.yaml \
	  --filename ../monitoring-stack/prometheus-auto-discovery.yaml
	$(HELM) uninstall --namespace grafana-stack grafana-stack
	$(HELM) repo remove prometheus-community
	$(KUBECTL) delete namespace grafana-stack --ignore-not-found
	$(KUBECTL) delete clusterrolebinding owner-cluster-admin-binding

.PHONY: rabbitmq-operator
rabbitmq-operator: | $(KUBECONFIG) ## Install RabbitMQ Cluster Operator into K8S
	$(KUBECTL) apply --filename https://github.com/rabbitmq/cluster-operator/releases/download/$(RABBITMQ_OPERATOR_VERSION)/cluster-operator.yml

.PHONY: teardown-rabbitmq-operator
teardown-rabbitmq-operator: | $(KUBECONFIG) ## Teardown the RabbitMQ Cluster Operator
	$(KUBECTL) delete --ignore-not-found --filename https://github.com/rabbitmq/cluster-operator/releases/download/$(RABBITMQ_OPERATOR_VERSION)/cluster-operator.yml

.PHONY: rabbitmq-production-cluster
rabbitmq-production-cluster: | rabbitmq-operator ## Install the production-ready RabbitMQ cluster
	$(KUBECTL) apply -f ../rabbitmq/clusters/production-ready-cluster.yaml
	
.PHONY: teardown-production-cluster
teardown-production-cluster: | $(KUBECTL) teardown-quorum-clients ## Teardown the production-ready RabbitMQ cluster
	$(KUBECTL) rabbitmq delete rabbitmq

.PHONY: rabbitmq-user
rabbitmq-user: | $(KUBECONFIG) ## Exec into RabbitMQ Pod to create the rabbit user
	$(KUBECTL) exec -it svc/rabbitmq -- rabbitmqctl add_user rabbit rabbit
	$(KUBECTL) exec -it svc/rabbitmq -- rabbitmqctl set_user_tags rabbit administrator
	$(KUBECTL) exec -it svc/rabbitmq -- rabbitmqctl set_permissions -p "/" "rabbit" ".*" ".*" ".*"

.PHONY: quorum-clients
quorum-clients: | $(KUBECONFIG) ## Create RabbitMQ clients to start quorum queues
	$(KUBECTL) apply --filename ../rabbitmq/clients/perf-test-quorum.yaml

.PHONY: teardown-quorum-clients
teardown-quorum-clients: | $(KUBECONFIG) ## Delete quorum queue RabbitMQ clients
	$(KUBECTL) delete --ignore-not-found --filename ../rabbitmq/clients/perf-test-quorum.yaml

.PHONY: check-rebalance-queues
check-rebalance-queues: | $(KUBECONFIG) ## Exec into RabbitMQ Pod to check the queue leaders across cluster
	$(KUBECTL) exec svc/rabbitmq -- rabbitmqctl --no-table-headers --quiet list_queues leader | sort | uniq -c

.PHONY: rebalance-queues
rebalance-queues: | $(KUBECONFIG) ## Exec into RabbitMQ Pod to rebalance queue leaders across cluster
	$(KUBECTL) exec svc/rabbitmq -- rabbitmq-queues rebalance all

.PHONY: rabbitmq-observer
rabbitmq-observer: | $(KUBECONFIG) ## Exec into RabbitMQ Pod to launch the RabbitMQ diagnostics observer
	$(KUBECTL) exec -it svc/rabbitmq -- rabbitmq-diagnostics observer

