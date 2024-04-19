.DEFAULT_GOAL := up

CLUSTER1_NAME = "cluster-1"
CLUSTER2_NAME = "cluster-2"

create:
	@minikube start --driver=virtualbox -p $(CLUSTER1_NAME)
	@minikube start --driver=virtualbox -p $(CLUSTER2_NAME)

down:
	@minikube delete -p $(CLUSTER1_NAME)
	@minikube delete -p $(CLUSTER2_NAME)

metallb:
	@kubectl config use-context $(CLUSTER1_NAME)
	@minikube addons enable metallb -p $(CLUSTER1_NAME)
	@kubectl apply -f manifests/$(CLUSTER1_NAME)/address-pool.yaml
	@kubectl config use-context $(CLUSTER2_NAME)
	@minikube addons enable metallb -p $(CLUSTER2_NAME)
	@kubectl apply -f manifests/$(CLUSTER2_NAME)/address-pool.yaml

clone:
	@test -d istio || git clone git@github.com:istio/istio.git

certs: clone
	@make -f istio/tools/certs/Makefile.selfsigned.mk root-ca
	@make -f istio/tools/certs/Makefile.selfsigned.mk $(CLUSTER1_NAME)-cacerts
	@make -f istio/tools/certs/Makefile.selfsigned.mk $(CLUSTER2_NAME)-cacerts
	@kubectl apply -f manifests/$(CLUSTER1_NAME)/namespace.yaml --context $(CLUSTER1_NAME)
	@kubectl create secret generic cacerts -n istio-system \
			--from-file=$(CLUSTER1_NAME)/ca-cert.pem \
			--from-file=$(CLUSTER1_NAME)/ca-key.pem \
			--from-file=$(CLUSTER1_NAME)/root-cert.pem \
			--from-file=$(CLUSTER1_NAME)/cert-chain.pem \
			--context $(CLUSTER1_NAME) || true
	@kubectl apply -f manifests/$(CLUSTER2_NAME)/namespace.yaml --context $(CLUSTER2_NAME)
	@kubectl create secret generic cacerts -n istio-system \
			--from-file=$(CLUSTER2_NAME)/ca-cert.pem \
			--from-file=$(CLUSTER2_NAME)/ca-key.pem \
			--from-file=$(CLUSTER2_NAME)/root-cert.pem \
			--from-file=$(CLUSTER2_NAME)/cert-chain.pem \
			--context $(CLUSTER2_NAME) || true

install-istio:
	@istioctl install --context=$(CLUSTER1_NAME) -f manifests/$(CLUSTER1_NAME)/istiod.yaml -y
	@istioctl install --context=$(CLUSTER2_NAME) -f manifests/$(CLUSTER2_NAME)/istiod.yaml -y

install-gateway:
	@bash scripts/gen-eastwest-gateway.sh \
		--mesh mesh1 \
		--cluster $(CLUSTER1_NAME) \
		--network network1 |\
		istioctl install \
		--context=$(CLUSTER1_NAME) -y -f -
	@bash scripts/gen-eastwest-gateway.sh \
		--mesh mesh1 \
		--cluster $(CLUSTER2_NAME) \
		--network network2 |\
		istioctl install \
		--context=$(CLUSTER2_NAME) -y -f -
	@kubectl apply -f manifests/gateway.yaml --context $(CLUSTER1_NAME)
	@kubectl apply -f manifests/gateway.yaml --context $(CLUSTER2_NAME)

endpoint-discovery:
	@istioctl create-remote-secret \
		--name=$(CLUSTER1_NAME) \
		--context $(CLUSTER1_NAME) | \
		kubectl apply -f - --context=$(CLUSTER2_NAME)
	@istioctl create-remote-secret \
		--name=$(CLUSTER2_NAME) \
		--context $(CLUSTER2_NAME) | \
		kubectl apply -f - --context=$(CLUSTER1_NAME)

sample:
	@kubectl create --context $(CLUSTER1_NAME) namespace sample || true
	@kubectl create --context $(CLUSTER2_NAME) namespace sample || true
	@kubectl label --context $(CLUSTER1_NAME) namespace sample \
		istio-injection=enabled
	@kubectl label --context $(CLUSTER2_NAME) namespace sample \
		istio-injection=enabled
	@kubectl apply --context $(CLUSTER1_NAME) \
		-f https://raw.githubusercontent.com/istio/istio/release-1.21/samples/helloworld/helloworld.yaml \
		-l service=helloworld -n sample
	@kubectl apply --context $(CLUSTER2_NAME) \
		-f https://raw.githubusercontent.com/istio/istio/release-1.21/samples/helloworld/helloworld.yaml \
		-l service=helloworld -n sample
	@kubectl apply --context $(CLUSTER1_NAME) \
		-f https://raw.githubusercontent.com/istio/istio/release-1.21/samples/helloworld/helloworld.yaml \
		-l version=v1 -n sample
	@kubectl apply --context $(CLUSTER2_NAME) \
		-f https://raw.githubusercontent.com/istio/istio/release-1.21/samples/helloworld/helloworld.yaml \
		-l version=v2 -n sample
	@kubectl apply --context $(CLUSTER1_NAME) \
		-f https://raw.githubusercontent.com/istio/istio/release-1.21/samples/sleep/sleep.yaml -n sample
	@kubectl apply --context $(CLUSTER2_NAME) \
		-f https://raw.githubusercontent.com/istio/istio/release-1.21/samples/sleep/sleep.yaml -n sample

up: create metallb certs install-istio install-gateway endpoint-discovery sample