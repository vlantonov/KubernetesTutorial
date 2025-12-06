# K8s communication

## Setups
```
minikube start
```

### ClusterIP (для внутреннего общения)
```
kubectl apply -f ./deployment‑clusterip.yaml -f service‑clusterip.yaml
```
Check service is created
```
kubectl get svc hello-clusterip-service
```

### NodePort (для внешнего доступа в dev‑среде)
```
kubectl apply -f service-nodeport.yaml
```
Check
```
kubectl get svc hello-nodeport-service
```
Paste resuls in browser - does not seem to work

### LoadBalancer (для продакшена в облаке)
Start in separate terminal
```
minikube tunnel
```
Start the service
```
kubectl apply -f service-loadbalancer.yaml
```
Check
```
get svc hello-loadbalancer-service
```

## End
```
minikube delete
```

## References
* [Управление сервисами в Kubernetes или как заставить их общаться](https://habr.com/ru/companies/otus/articles/971292/)
