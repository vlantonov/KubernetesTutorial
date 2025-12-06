# Kubernetes Tutorial

## Contents
### What is Kubernetes
*  What problems does Kubernetes solve?
*  What features do container orchestration tools offer?

### Main K8s Components
*  Node & Pod
*  Service & Ingress
*  ConfigMap & Secret
*  Volumes
*  Deployment & StatefulSet

### K8s Architecture
*  Worker Nodes
*  Master Nodes
*  Api Server
*  Scheduler
*  Controller Manager
*  etcd - the cluster brain

### Minikube and kubectl - Local Setup
*  What is minikube?
*  What is kubectl?
*   install minikube and kubectl
* [Commands](https://gitlab.com/nanuchi/youtube-tutorial-series/-/blob/master/basic-kubectl-commands/cli-commands.md)

### Main Kubectl Commands - K8s CLI
*  Get status of different components
*  create a pod/deployment
*  layers of abstraction
*  change the pod/deployment
*  debugging pods
*  delete pod/deployment
*  CRUD by applying configuration file

### K8s YAML Configuration File
*  3 parts of a Kubernetes config file (metadata, specification, status)
*  format of configuration file
*  blueprint for pods (template)
*  connecting services to deployments and pods (label & selector & port)
*  [demo](https://gitlab.com/nanuchi/youtube-tutorial-series/-/tree/master/kubernetes-configuration-file-explained)

### Demo Project
*  Deploying MongoDB and Mongo Express
*  MongoDB Pod
*  Secret
*  MongoDB Internal Service
*  Deployment Service and Config Map
*  Mongo Express External Service
* [Project](https://gitlab.com/nanuchi/youtube-tutorial-series/-/tree/master/demo-kubernetes-components)

### Organizing your components with K8s Namespaces
*  What is a Namespace?
*  4 Default Namespaces
*  Create a Namespace
*  Why to use Namespaces? 4 Use Cases
*  Characteristics of Namespaces
*  Create Components in Namespaces
*  Change Active Namespace
* [Install Kubect](https://github.com/ahmetb/kubectx#installation)

### K8s Ingress explained
*  What is Ingress? External Service vs. Ingress
*  Example YAML Config Files for External Service and Ingress
*  Internal Service Configuration for Ingress
*  How to configure Ingress in your cluster?
*  What is Ingress Controller?
*  Environment on which your cluster is running (Cloud provider or bare metal)
*  Demo: Configure Ingress in Minikube
*  Ingress Default Backend
*  Routing Use Cases
*  Configuring TLS Certificate
* [Repo](https://gitlab.com/nanuchi/youtube-tutorial-series/-/blob/master/kubernetes-ingress/dashboard-ingress.yaml)

### Helm - Package Manager
*  Package Manager and Helm Charts
*  Templating Engine
*  Use Cases for Helm
*  Helm Chart Structure
*  Values injection into template files
*  Release Management / Tiller (Helm Version 2!)
* [Helm Hub](https://artifacthub.io/)
* [Helm charts GitHub Project](https://github.com/helm/charts)

### Persisting Data in K8s with Volumes
*  The need for persistent storage & storage requirements
*  Persistent Volume (PV)
*  Local vs Remote Volume Types
*  Who creates the PV and when?
*  Persistent Volume Claim (PVC)
*  Levels of volume abstractions
*  ConfigMap and Secret as volume types
*  Storage Class (SC)
* [kubernetes-volumes](https://gitlab.com/nanuchi/youtube-tutorial-series/-/tree/master/kubernetes-volumes)

### Deploying Stateful Apps with StatefulSet
*  What is StatefulSet? Difference of stateless and stateful applications
*  Deployment of stateful and stateless apps
*  Deployment vs StatefulSet
*  Pod Identity
*  Scaling database applications: Master and Worker Pods
*  Pod state, Pod Identifier
*  2 Pod endpoints

### K8s Services
*   What is a Service in K8s and when we need it?
*  ClusterIP Services
*  Service Communication
*  Multi-Port Services
*  Headless Services
*  NodePort Services
*  LoadBalancer Services

## Reference
* [Kubernetes Tutorial for Beginners](https://www.youtube.com/watch?v=X48VuDVv0do)
* [Learn Kubernetes Basics](https://kubernetes.io/docs/tutorials/kubernetes-basics/)
* [How to Learn Kubernetes - Complete Roadmap & Resources](https://devopscube.com/learn-kubernetes-complete-roadmap)
* [geeksforgeeks - Kubernetes Tutorial](https://www.geeksforgeeks.org/devops/kubernetes-tutorial/)
* [Helm](https://helm.sh/docs/)
* [Play with Kubernetes](https://labs.play-with-k8s.com/)
* [Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
* [Monitoring Kafka in Kubernetes with Prometheus and Grafana](https://vkontech.com/monitoring-kafka-in-kubernetes-with-prometheus-and-grafana/)
* [Kubernetes Observability with OpenTelemetry | A Complete Setup Guide](https://signoz.io/blog/kubernetes-observability-with-opentelemetry/)
* [Kubernetes Networking from Packets to Pods](https://www.lucavall.in/blog/kubernetes-networking-from-packets-to-pods)
* [From null to applications on Kubernetes - Roberth Strand - NDC Oslo 2025](https://www.youtube.com/watch?v=lwcGzsa2nho) , [code](https://github.com/robstrdev/null-to-apps-on-k8s/tree/ndcoslo25) , [author](https://github.com/roberthstrand)
