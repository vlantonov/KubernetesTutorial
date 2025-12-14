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
####  3 parts of a Kubernetes config file (metadata, specification, status)
* First part: metadata
* Second part: specification - specific attributes to the kind!

`nginx-deployment.yaml`
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels: ...
spec:
  replicas: 2
  selector:
  template:
```
`nginx-service.yaml`
```
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
  ports:
```

* Third part: status - automatically generated and added by K8s
* K8s compares the desired and actual state - used for self-healing
* K8s updates state continuously from etcd

#### Format of the configuration file
* YAML
* Strict identation - use validator
* Store the config file with your code or own repository

#### Blueprint for pods (template)
* See in the `spec` section
```
spec:
  ...
  template:
    metadata:
      ...
    spec:
      ...
```
* Has own `metadata` and `spec` section
* Applies to a pod
* Blueprint for a pod - port, image, name, ...

#### Connecting services to deployments and pods (label & selector & port)
* `metadata` contains the labels
* `spec` part contains selectors
* Any key-value pair for component - `app: nginx`
* Pods get the label through the template blueprint
* The label is matched by the `selector`
* `port` - where the service is available
* `targerPort` - where the pod is listening for service. Should match the container port of the deployment
* [demo](https://gitlab.com/nanuchi/youtube-tutorial-series/-/tree/master/kubernetes-configuration-file-explained)
```
kubectl apply -f nginx-deployment.yaml
kubectl apply -f nginx-service.yaml
kubectl get pod
kubectl get service
kubectl describe service nginx-service
kubectl get pod -o wide # get IPs
```
* Get resulting/updated configuration of deployment (from etcd) - can be saved to yaml file
```
kubectl get deployment nginx-deployment -o yaml
```
* Lots of generated stuff that can be removed
* Can be deleted with configuration file
```
kubectl delete -f nginx-deployment.yaml
kubectl delete -f nginx-service.yaml
```

### Demo Project

#### Setup
* Two Deployments / Pods
* Two Services
* A ConfigMap
* A Secret
* minikube running
```
kubectl get all
```
* Files in `k8s-configuration`
* Deployment Config File is checked into repository
* Username and password should not go there!
* Secret lives in K8s, not in the repository!

#### Secret
* Must be created before the Deployment!
* `kind` : Secret
* `metadata` \ `name` : a random name
* `type`: Opaque - default for arbitrary key-value pairs
* `data`: Actual contents - key-value pairs
* Values in pairs are not plain text, but base64 encoded. Not secure
* Generate base64 on terminal
```
echo -n 'username' | base64
```
* There are built-in mechanism (like encryption) for basic security, not enabled by default
* Apply the secret
```
kubectl apply -f mongo-secret.yaml
kubectl get secret
```
* Secrets now can be referenced in Deployment

####  MongoDB Pod
* MongoDB deployment
```
kubectl apply -f mongo.yaml
kubectl get all
```
* Watch progress
```
kubectl get pod --watch
```
* More information
```
kubectl describe pod [pod_name]
```

#### MongoDB Internal Service
* Need for other components to talk to pod
* Multiple documents can be done in one file
* Document separation: `---`
* Deployment and service in one file because they belong together!
* Service configuration file
   * `kind` : Service
   * `metadata` \ `name` : a random name
   * `selector` : to connect to Pod through label
   * `ports`: see next
   * `port` : Service port
   * `targetPort` : containerPort of Deployment
* Apply configuration
```
kubectl apply -f mongo.yaml
kubectl get service
kubectl describe service mongodb-service
kubectl get pod -o wide
```

#### Config Map
* External configuration
* Centralized
* Other components can use it
* `kind` ConfigMap
* `metadata` \ `name` : a random name
* `data`: Actual contents - key-value pairs
* Apply
```
kubectl apply -f mongo-configmap.yaml
```
* configMap now can be referenced in Deployment

#### Deploying Mongo Express
* ConfigMap must already be in the cluster when referencing it!
* Needs ConfigMap for MongoDB server name
* Needs MongoDB Address / Internal Service
* Needs credentials to authentificate
* Apply
```
kubectl apply -f mongo-express.yaml
```

#### Mongo Express External Service
* Access Mongo Express from a browser
* Create in the same file as Mongo Express deployment
* Configuration is similar to internal service
* To expose externally add to config `type` : LoadBalancer
* Internal service is `ClusterIP` by default
* Internal service also acts as a load balancer!
* Here LoadBalancer assigns service external IP address and so accepts external requests
* `nodePort` - port where external IP address will be open. Must be between 30000 and 32767
* Apply updated configuration
```
kubectl apply -f mongo-express.yaml
```
* Provide external IP address
```
minikube service mongo-express-service
```
* Default - username `admin`, password `pass`
* [Project](https://gitlab.com/nanuchi/youtube-tutorial-series/-/tree/master/demo-kubernetes-components)

### Organizing your components with K8s Namespaces

####  What is a Namespace?
* Resources are organized in namespaces
* Virtual cluster inside a cluster

#### Four Default Namespaces
```
kubectl get namespaces
```
* kube-system
   * DO NOT create or modify in there
   * System processes
   * Master and kubectl processes
* kube-public
   * publicly accessible data
   * a configmap, which contains cluster information
```
kubectl cluster-info
```
* kube-node-lease
   * hearbeats of the nodes
   * each node has associated lease object in Namespace
   * determine the availability of the node
* default
   * resources you created are located there

####  Create a Namespace
* With command
```
kubectl create namespace my-namespace
kubectl get namespaces
```
* With a configuration file (ConfigMap)
```
metadata:
   name: mysql-configmap
   namespace: my-namespace
```

####  Why to use Namespaces? Four Use Cases
* Everything in one Namespace
   * Problem - no overview
   * Group resources in Namespaces - database, monitoring, elastic stack, ...
   * Do not use for smaller projects
   * Case: Structure your components
* Conflicts: Many teams, same application
   * Can overwrite same deployments!
   * Each team works in their own namespace
   * Case: Avoid conflicts between teams
* Resource sharing: Staging and Development
   * Reuse components in both environments
   * Blue-green deployment - Two different versions of production
   * Case: Share service between different environments
* Access and Resource Limits on Namespaces
   * Control access
   * Limit: CPU, RAM, Storage per Namespace - define resource quota
   * Each team has own, isolated enironment
   * Case: Access and Resource Limits on Namespaces Level

#### Characteristics of Namespaces
* You can't access most resources from another Namespace - separate ConfigNap, Secret needed
* However Service can be shared between Namespaces - namespace at the end - `.database`
```
data:
  db_url:mysql-service.database
```
* Some components cannot be created within Namespace
* These live globally in the cluster and cannot be isolated
* Example: Volume, PersistentVolume, Node
* These components can be listed with
```
kubectl api-resources --namespaced=false
```

#### Create Components in Namespaces
* By default components are created in the default Namespace
* Add to command
```
kubectl apply -f my-configmap.yaml --namespace=my-namespace
```
* Or inside the configuration file (see above) - preferred
* Get the component - add `-n` to command
```
kubectl get configmap -n my-namespace
```

#### Change Active Namespace
* Change the namespace wihout constantly adding the `-n` - use kubens command
* [Kubens, Kubectx](https://github.com/ahmetb/kubectx#installation) - tool to switch between contexts (clusters) on kubectl faster
* Available to install on Ubuntu
* List of namespacec - `kubens`
* Change namespace - `kubens my-namespace`

### K8s Ingress explained

#### What is Ingress? External Service vs. Ingress
* Ingress: way to manage external access to your services in a Kubernetes cluster
* It provides HTTP and HTTPS routing to your services, acting as a reverse proxy.
* External service is only good for test cases and fast protyping
* With Ingress the IP and port is not opened - request reaches Ingress first

#### Example YAML Config Files for External Service and Ingress
* Used for - `kind` : Service ; `spec`, `type` : LoadBalancer
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
spec:
  rules:
  - host: myapp.com
    http:
      paths:
      - backend:
            service:
              name: myapp-internal-service
              port:
                number: 8080
```
* Routing rules - forward request to the internal service
* In browser: `http://myapp.com`
* `paths` - the URL paths
* HTTPS configuration later
* Incoming request gets forwared to the internal service

#### Internal Service Configuration for Ingress
* Defined by `backend`
* Names and ports should correspond
```
apiVersion: v1
kind: Service
metadata:
  name: myapp-internal-service
spec:
  selector:
    app: myapp
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
```
* No `nodePort` in internal Service!
* Instead of `LoadBalancer` default type - `ClusterIP`
* Host
   * Valid domain address
   * Map domain name to Node's IP address, which is the entrypoint

#### Configure Ingress in your cluster
* The YAML created Ingress component
* An implementation for Ingress is needed - Ingress Controller. Different third party implementations exist.
* Evaluates and processes Ingress rules, manages all redicrections, entrypoint to cluster
* Ingress Controller: Environment on which your cluster is running (Cloud provider or bare metal)
* K8s Nginx Ingress controller - one of implementations
* Cloud service provider (AWS, Google Cloud ...) - out of the box k8s solutions or own virtualized load balancer. Different ways to configure.
* Bare metal - configure some kind of entrypoint. Either inside of the cluster or outside as a separate server.

#### Configure Ingress in Minikube
* Install Ingress Controller in minikube
```
minikube addons enable ingress
kubectl get pod -n ingress-nginx
```
* Automatically starts Nginx Ingress controller
* Create Ingress rule (Dashboard is needed)
```
minikube addons enable dashboard
kubectl get all -n kubernetes-dashboard
```
* Service: `kubernetes-dashboard` - and related pod
* Namespace should be same as the service and pod!
* Always configure resolving IP to host name - create Ingress rule
* Apply rule
```
kubectl apply -f dashboard-ingress.yaml
kubectl get ingress -n kubernetes-dashboard
sudo nano /etc/hosts # add IP dasboard.com
```

#### Ingress default backend
* Check for `Default backend` in
```
kubectl describe ingress dashboard-ingress -n kubernetes-dashboard
```
* If there is no mapping defined then this service will handle the request
* Can be used for custom error messages
* Configuration - `port` and `name` from command output
```
kind: Service
metadata:
  name: default-http-backend
spec:
  selector:
    app: default-response-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

#### Routing Use Cases
* Multiple paths for same host - one domain, many services
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fanout-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 3000
```
* Multiple sub-domains or domains: `http://myapp.com/analytics` as `http://analytics.myapp.com`
* Configuring TLS Certificate - `tls`, app name and secret name for TLS certificates
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-example-ingress
spec:
  tls:
  - hosts:
    - myapp.com
    secretName: myapp-secret-tls
  rules:
  - host: myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-internal-service
            port:
              number: 8080

```
Secret configuration for the certificate
```
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secret-tls
  namespace: default # Same namespace as the Ingress component
type: kubernetes.io/tls
data:
  # Base64 encoded TLS certificate
  # Replace with your actual base64 encoded certificate
  tls.crt:
  
  # Base64 encoded private key
  # Replace with your actual base64 encoded private key
  tls.key:
```
* [Repo](https://gitlab.com/nanuchi/youtube-tutorial-series/-/blob/master/kubernetes-ingress/dashboard-ingress.yaml)

### Helm - Package Manager

#### What is Helm
* Package Manager for K8s
* Packages YAML files and distributes in public and private repositories
* [Helm Docs](https://helm.sh/docs/)

#### Helm Charts
* Bundle of YAML files
* Own charts can be created with Helm
* Push these to Helm repository
* Download and use existing ones

#### How to use Helm Charts
* Search `helm search <keyword>`
* Go to [Helm Hub](https://artifacthub.io/)
* Company can have private registry for internal sharing

#### Templating Engine
* Deployment and Service configuration almost the same!
* Define a common blueprint
* Dynamic values are replaced by placeholders
```
apiVersion: v1
kind: Pod
metadata:
  name: {{ .Values.name }}
spec:
  containers:
  - name: {{ .Values.container.name }}
    image: {{ .Values.container.image }}
    port: {{ .Values.container.port }}
```
* Extrernal configuration comes from additional YAML file - like `values.yaml`
```
name: my-app
container:
  name: my-app-container
  image: my-app-image
  port: 9001
```
* `Values` is an obect created with file above or `--set` flag

#### Use Cases for Helm
* Practical for CI/CD
* Same applications across different environments
* Like Development, Staging and Production clusters

#### Helm Chart Structure
* Directory structure
```
mychart/
  Chart.yaml
  values.yaml
  charts/
  templates/
  ...
```
* `mychart` - name of the chart
* `Chart.yaml` - metainfo about the chart (name, version, dependencies...)
* `values.yaml` - values for the template files. Default values that can be overwritten.
* `charts/` - chart dependencies (other charts)
* `templates/` - the actual template files
* Deployed into K8s by `helm install <chartname>`
* Optionally ReadMe or license files
* Values injection - `helm install --values=my-values.yaml <chartname>`
* Or single value - `helm install --set version=2.0.0 <chartname>`

#### Release management
* Difference between version 2 and version 3
* Version 2 - Client (Helm CLI) and Server (Tiller)
* Request for component creation is executed on Server
* Server stores copy of each configuration - keep track of all chart executions
* Changes are applied on existing environment - `helm upgrade <chartname>`
* On problems - `helm rollback <chartname>`
* Downsides of Tiller - too much power inside the K8s cluster. Security issue
* Removed Tiller in Helm 3
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
