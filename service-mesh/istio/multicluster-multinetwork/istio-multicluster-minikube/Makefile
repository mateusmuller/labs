.DEFAULT_GOAL := up

CLUSTER1_NAME = "cluster-1"
CLUSTER2_NAME = "cluster-2"

create:
	@minikube start --driver=virtualbox -p cluster-1
	@minikube start --driver=virtualbox -p cluster-2

down:
	@minikube delete -p cluster-1
	@minikube delete -p cluster-2

metallb:
	@kubectl config use-context cluster-1
	@minikube addons enable metallb -p cluster-1
	@kubectl apply -f manifests/cluster-1/address-pool.yaml
	@kubectl config use-context cluster-2
	@minikube addons enable metallb -p cluster-2
	@kubectl apply -f manifests/cluster-2/address-pool.yaml

clone:
	@test -d istio || git clone git@github.com:istio/istio.git

certs: clone
	@make -f istio/tools/certs/Makefile.selfsigned.mk root-ca
	@make -f istio/tools/certs/Makefile.selfsigned.mk cluster-1-cacerts
	@make -f istio/tools/certs/Makefile.selfsigned.mk cluster-2-cacerts
	@kubectl apply -f manifests/cluster-1/namespace.yaml --context cluster-1
	@kubectl create secret generic cacerts -n istio-system \
      --from-file=cluster-1/ca-cert.pem \
      --from-file=cluster-1/ca-key.pem \
      --from-file=cluster-1/root-cert.pem \
      --from-file=cluster-1/cert-chain.pem \
			--context cluster-1 || true
	@kubectl apply -f manifests/cluster-2/namespace.yaml --context cluster-2
	@kubectl create secret generic cacerts -n istio-system \
      --from-file=cluster-2/ca-cert.pem \
      --from-file=cluster-2/ca-key.pem \
      --from-file=cluster-2/root-cert.pem \
      --from-file=cluster-2/cert-chain.pem \
			--context cluster-2 || true

install-istio:
	@istioctl install --context=cluster-1 -f manifests/cluster-1/istiod.yaml -y
	@istioctl install --context=cluster-2 -f manifests/cluster-2/istiod.yaml -y

install-gateway:
	@bash scripts/gen-eastwest-gateway.sh --mesh mesh1 --cluster cluster-1 --network network1 | istioctl install --context=cluster-1 -y -f -
	@bash scripts/gen-eastwest-gateway.sh --mesh mesh1 --cluster cluster-2 --network network2 | istioctl install --context=cluster-2 -y -f -
	@kubectl apply -f manifests/gateway.yaml --context cluster-1
	@kubectl apply -f manifests/gateway.yaml --context cluster-2

endpoint-discovery:
	@istioctl create-remote-secret \
    --name=cluster-1 \
		--context cluster-1 | \
    kubectl apply -f - --context=cluster-2
	@istioctl create-remote-secret \
    --name=cluster-2 \
		--context cluster-2 | \
    kubectl apply -f - --context=cluster-1

sample:
	@kubectl create --context cluster-1 namespace sample || true
	@kubectl create --context cluster-2 namespace sample || true
	@kubectl label --context cluster-1 namespace sample \
    istio-injection=enabled
	@kubectl label --context cluster-2 namespace sample \
    istio-injection=enabled
	@kubectl apply --context cluster-1 \
    -f https://raw.githubusercontent.com/istio/istio/release-1.21/samples/helloworld/helloworld.yaml \
    -l service=helloworld -n sample
	@kubectl apply --context cluster-2 \
    -f https://raw.githubusercontent.com/istio/istio/release-1.21/samples/helloworld/helloworld.yaml \
    -l service=helloworld -n sample
	@kubectl apply --context cluster-1 \
    -f https://raw.githubusercontent.com/istio/istio/release-1.21/samples/helloworld/helloworld.yaml \
    -l version=v1 -n sample
	@kubectl apply --context cluster-2 \
    -f https://raw.githubusercontent.com/istio/istio/release-1.21/samples/helloworld/helloworld.yaml \
    -l version=v2 -n sample
	@kubectl apply --context cluster-1 \
    -f https://raw.githubusercontent.com/istio/istio/release-1.21/samples/sleep/sleep.yaml -n sample
	@kubectl apply --context cluster-2 \
    -f https://raw.githubusercontent.com/istio/istio/release-1.21/samples/sleep/sleep.yaml -n sample

up: create metallb certs install-istio install-gateway endpoint-discovery sample