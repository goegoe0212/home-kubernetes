# RKE2


## MEMO
aposixの検証
```sh
helm install apisix apisix/apisix \
  --set service.type=LoadBalancer \
  --set ingress-controller.enabled=true \
  --create-namespace \
  --namespace ingress-apisix \
  --set ingress-controller.config.apisix.serviceNamespace=ingress-apisix \
  --set ingress-controller.config.apisix.adminAPIVersion=$ADMIN_API_VERSION \
  --set etcd.persistence.enabled=true \
  --set etcd.persistence.size=20Gi \
  --set etcd.persistence.storageClass=longhorn
```
