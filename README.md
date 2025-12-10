# Kubernetes Tutorial

## Contents

### What is Kubernetes

####  What problems does Kubernetes solve?
* Monolith to microservices
* Increased usage of containers

####  What features do container orchestration tools offer?
* High availability or no downtime
* Scalability or high performance
* Disaster recovery - backup and restore

### Main K8s Components

####  Node & Pod
* Pod: smallest unit of k8s
* Pod: abstraction over container
* Interaction only with k8s layer
* Usually one application per pod
* Node: machine (physical or virtual) in a Kubernetes cluster that runs your applications.
* Node" contains the tools needed to run Pods, including the container runtime (like Docker), the Kubelet (agent), and the Kube proxy (networking).
* Each node gets own IP address
* New IP address on recreation
* Master node (Control Plane):  makes decisions, like where to run applications, handles scheduling, and keeps track of everything.
* Worker nodes: machines that actually run your apps inside containers.
* Worker nodes: has a Kubelet (agent), a container runtime (like Docker or containerd), and tools for networking and monitoring.
* Cluster: group of computers (called nodes) that work together to run your containerized applications.

#### Service & Ingress
* Service: permanent IP address
* Lifecycle of Pod and Service NOT connected
* App should be accessible through browser
* Ingress: Expose outside

####  ConfigMap & Secret
* Database URL usually in the build application!
* ConfigMap: external configuration of your application
* Don't put credential into ConfigMap!
* Secret: use to store secret data
* Secret: base64 encoded
* The build-in security mechanism is not enabled by default!
* Use it as environment variables or as a properties file

#### Volumes
* Data storage persisted
* Attach storage to pod
* Storage on local machine
* Or remote, outside od the k8s cluster
* k8s doesn't manage data persistence!

#### Deployment & StatefulSet
* Service: way to connect applications running inside your cluster. It gives your Pods a stable way to communicate, even if the Pods themselves keep changing.
* The replica is connected to the same service
* Service has two functionalities: permanent IP and load balancer
* Deployment: Kubernetes object used to manage a set of Pods running your containerized applications. It provides declarative updates, meaning you tell Kubernetes what you want, and it figures out how to get there.
* Deployment: define blueprints for pods and number of replicas
* Deployment: abstraction of pods; can be created
* Deployment: stateLESS Apps
* DB can't be replicated via Deployment! Has state
* StatefulSet: Avoid data inconsistencies
* StatefulSet: for STATEFUL apps or Databases
* Deploying StatefulSet is not easy!
* DB are ofter hosted outside k8s cluster

### K8s Architecture

####  Worker Nodes
* Each node has multiple posd on it
* Three processes must be installed on every node
* Worker Nodes do the actual work
* First process: Container runtime (can be Docker)
* Second process: Kubelet - interacts both with container and node
* Kubelets starts the process with a container inside
* Communication between Nodes is done via Services
* Third process: Kube proxy - forward the requests, must be installed on every node

####  Master Nodes
* Managing processes:
   * Schedule pod
   * Monitor
   * Re-schedule/re-start pod
   * Join a new Node
* Four processes on every master Node: API Server, Scheduler, Controller Manager, etcd 

####  Api Server
* Cluster gateway
* Gatekeeper for authentification

####  Scheduler
* Decides on which node to put the pod
* Kubelet puts pod

####  Controller Manager
* Detects cluster state changes
* Decides rescheduling

####  etcd - the cluster brain
* key/value store where cluster changes are stored
* Cluster brain; informs scheduler
* Application data NOT stored

### Minikube and kubectl - Local Setup

#### What is minikube?
* [minikube](https://minikube.sigs.k8s.io/docs/)
* To test on local machine
* One node cluster
* Master and worker processes run on one machine
* Requires Docker preinstalled
* Creates Virtual box on your machine
* Node runs in that virtual box

#### What is kubectl?
* [kubectl - command line tool](https://kubernetes.io/docs/reference/kubectl/)
* API server is the entry point used
* Creates pods, enables pods, run pods, destroys pods ...
* It is applicable to all K8s clusters

#### Install minikube and kubectl
* [Install minikube](https://minikube.sigs.k8s.io/docs/start/?arch=%2Flinux%2Fx86-64%2Fstable%2Fbinary+download)
* [Install kubectl](https://kubernetes.io/docs/tasks/tools/)
* Needs virtualization on machine - install Hypervisor
* [Commands](https://gitlab.com/nanuchi/youtube-tutorial-series/-/blob/master/basic-kubectl-commands/cli-commands.md)
* Create and start the cluster
```
minikube start
```
* Check nodes
```
kubectl get nodes
```
* Check status
```
minikube status
```

### Main Kubectl Commands - K8s CLI

#### Get status of different components
* `kubectl get nodes`
* `kubectl get pod`
* `kubectl get services`
* `kubectl get replicaset`
* `kubectl get deployment`

#### Layers of abstraction
* Deployment - abstraction over pods
   * Blueprint for creating pods
   * Basic configuration - name and image
* ReplicaSet
   * Manages replicas of a pod
   * Configured via deployment
* Lower levels managed automatically

#### Create a pod/deployment
* Pod is not created directly
* Create deployment `kubectl create deployment [name]`
* Create deployment `kubectl create deployment NAME --image=image [--dry-run] [options]`
* Example `kubectl create deployment nginx-depl --image=nginx:1.29.3`
* Another example `kubectl create deployment mongo-depl --image=mongo:8.2.2`
* Check `kubectl get deployment` , `kubectl get pod`

#### Change the pod/deployment
* Edit deployment `kubectl edit deployment [name]`
* Example `kubectl edit deployment nginx-depl`
* Autogenerated configuration file with default values
* Check `kubectl get pod` - old pod terminating, new pod starting
* Check old and new pod `kubectl get replicaset`

#### Debugging pods
* Log to console `kubectl logs [pod_name]`
* Get interactive terminal `kubectl exec -it [pod_name] -- bin/bash`
* Get `pod_name` from `kubectl get pod`
* Information `kubectl describe pod [pod_name]` - check state changes

#### Delete pod/deployment
* Delete deployment `kubectl delete deployment [name]`
* Clears all the undelying pods, replica sets, ...
* Check: `kubectl get deployment`, `kubectl get pod`, `kubectl get replicaset`

#### CRUD by applying configuration file
* Command `kubectl apply -f [file_name]`
* Basic file `nginx-deployment.yaml`
* Example `kubectl apply -f nginx-deployment.yaml`
* Creates and configures deployment

### K8s YAML Configuration File
*  3 parts of a Kubernetes config file (metadata, specification, status)
*  format of configuration file
*  blueprint for pods (template)
*  connecting services to deployments and pods (label & selector & port)
*  [demo](https://gitlab.com/nanuchi/youtube-tutorial-series/-/tree/master/kubernetes-configuration-file-explained)

### Demo Project
* Install two master nodes and three worker nodes
* Get new bare server
* Install all the master/worker nodes
* Add it to the cluster

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
* Ingress: way to manage external access to your services in a Kubernetes cluster. It provides HTTP and HTTPS routing to your services, acting as a reverse proxy.
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
