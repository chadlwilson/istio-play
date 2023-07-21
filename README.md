
Going through https://istio.io/latest/docs/setup/getting-started/

# Create a cluster to work with
```shell
colima start --profile k8s-istio --cpu 8 --memory 6 --kubernetes
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.18.0 sh -
```

# Demoing installation and dashboards
```shell
istioctl install --set profile=demo -y
kubectl apply -f istio-1.18.0/samples/addons
istioctl dashboard kiali
istioctl dashboard grafana

# https://istio.io/latest/docs/examples/bookinfo/
kubectl apply -f istio-1.18.0/samples/bookinfo/platform/kube/bookinfo.yaml
kubectl apply -f istio-1.18.0/samples/bookinfo/networking/bookinfo-gateway.yaml

# Open http://localhost/productpage
```

# Demoing traffic management
```shell
kubectl label namespace default istio-injection=enabled
k rollout restart deployment

# Let's drive some traffic through our ingress
hey -c 1 -n 10000 -q 10 -m GET http://localhost/productpage

# Make our destinations have routing rules applicable
kubectl apply -f istio-1.18.0/samples/bookinfo/networking/destination-rule-all.yaml

# Let's route traffic just to v1 and v3 and stop routing to v2
cat istio-1.18.0/samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml
kubectl apply -f istio-1.18.0/samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml

# Look at Kiali and Grafana now - didn't need to bounce services
```

## Demoing mTLS enforcement
```shell
k create ns new
kubectl run -i --tty --rm debug --image=ghcr.io/containeroo/alpine-toolbox:latest --restart=Never -n new -- sh
# curl -vvv http://reviews.default:9080

kubectl apply -n istio-system -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
spec:
  mtls:
    mode: STRICT
EOF

# curl -vvv -k https://reviews.default:9080

# apk add openssl && openssl s_client -showcerts -servername -connect reviews.default:9080
# https://redkestrel.co.uk/products/decoder/
```


### Cleanup

```shell
kubectl label namespace default istio-injection-
kubectl delete -f istio-1.18.0/samples/addons
istio-1.18.0/samples/bookinfo/platform/kube/cleanup.sh
kubectl delete -f istio-1.18.0/samples/bookinfo/networking/bookinfo-gateway.yaml
kubectl delete -f istio-1.18.0/samples/bookinfo/platform/kube/bookinfo.yaml
istioctl uninstall -y --purge
k delete ns istio-system
colima stop --profile k8s-istio
colima start --profile k8s-istio --cpu 8 --memory 6 --kubernetes
```
