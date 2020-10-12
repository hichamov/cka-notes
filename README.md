# CKA

### Introduction

Prerequisites:

* Docker
* Basics of k8s
* Yaml
* Virtualization

Objectives:

* Scheduling
* Logging
* Application lifecycle management
* Cluster maintenance
* Security
* Storage
* Networking
* Installation, configuration & validation
* Troubleshooting

### Core Concepts

##### Cluster Architecture

* Control plan (Master node)

ETCD Cluster (Key/value store)

Kube-Scheduler

Controller-manager (Node cotroller - Replication controller)

Kube-apiserver

Container Runtime (Docker - RKT)

* Workers

Kubelet (Agent that runs on workers, used to monitor the status of the worker)

Kube-proxy (Ensure iptables rules are in place to connect different pods)

Container Runtime (Docker - RKT)

##### ETCD 

A key/value store (each object has its own document), when editing a specific document, it doesn't affect other documents
Default port: 2379
etcdctl is used tointeract with etcd, ex: etcdctl set key value, to retrieve the value of key: etcdctl get key

##### ETCD in kubernetes

Etcd is the single source of truth in k8s, it stores eveything about the cluster

##### Kube-apiserver

API server is the management point of the cluster, it authenticates all requests before retreiving data from etcd

##### Kube-controller-manager

consistently monitors the status of all objects in the cluster, and if the current state doesn't match the desired state, it acts to make them match

Monitors nodes (every 5s, after 40s the node is marked as down, after 5m pods are created in another node)

Monitors replication controllers

##### Kube-scheduler

Deciding which pod will be created on which node

Filter nodes, rank nodes

##### Kubelet

Is the responsable of each nodes, send the status of the objects in the node & send it to the control plan

##### Kube-proxy

Manage iptables rules to make all pods available through the cluster

##### POD

Containers are deployed in a package called pod, to scale the app, we create new pods, a pod contains 1 or more container

Containers on the same pod shares the same network space, they call each other via 127.0.0.1
Pods are deployed via yaml manifest ot kubectl run command, ex: kubectl run nginx --image nginx:latest

#####  POD with yaml

Pods can be created via yaml files, a yaml file contains 4 top level fields:

* apiVersion 
* kind 
* metadata 
* spec

ex:

```
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  labels: 
	app: myapp
	type: frontend
spec:
  containers:
    - name: nginx-container
      image: nginx
```

Then apply the file

```
kubectl apply -f filename.yaml
```

#### Controllers

##### Replication controller

If an app fails, the pod will be destroyed, and will not be created again, a replication controller is used to monitor the number of pods for specific app, and makes sure the the existing nbr matched the desired nbr

Replication controller is the version of the new 'replica set'

ex:

```
apiVersion: v1
kind: ReplicationController
metadata: 
  name: myapp-rc
  labels: 
	app: myapp
	type: frontend
spec:
  - template:
	metadata:
	  name: myapp
	  labels:                  	
		app: myapp         
		type: frontend
	spec:
	  containers:
	    - name: nginx-container
	      image: nginx
    replicas: 3
```

Replicasets:

```
apiVersion: apps/v1
kind: ReplicaSet          
metadata: 
  name: myapp-rc
  labels: 
	app: myapp
	type: frontend
spec:
  template:
	metadata:
	  name: myapp
	  labels:                  	
		app: myapp         
		type: frontend
	spec:
	  containers:
	    - name: nginx-container
	      image: nginx
  replicas: 3
  selector: 
	matchLabels:
		type: frontend
```

To create the replicaset

```
kubectl create -f filename.yml
```

##### Labels and Selectors

Replication controllers identifies pods via labels, it monitors only pods that has specific labels

##### Scaling

To scale a ReplicaSet, we can edit the manifest file 'replicas' section and run 'kubectl replace -f filename.yml', or we can run 'kubectl scale --replicas=x filename.yml', or 'kubectl scale --replicas=x replicaset rsname'

##### Deployments

A deployment is a higher level of abstraction that allow us to apply rolling updates & rolling back & garanties availability of pods
The deployment file is the same as ReplicaSet one except the kind

```
apiVersion: apps/v1
kind: Deployment          
metadata: 
  name: myapp-rc
  labels: 
	app: myapp
	type: frontend
spec:
  template:
	metadata:
	  name: myapp
	  labels:                  	
		app: myapp         
		type: frontend
	spec:
	  containers:
	    - name: nginx-container
	      image: nginx
  replicas: 3
  selector: 
	matchLabels:
		type: frontend
```

To create the deployment

```
kubectl create -f filename.yml

```

##### Namespaces

A namespace is an isolation that groups a group of pods and services to manage them, we can specify quota to each namespace

To create a namespace

```
kubectl create namespace name
```

To create a pod in a specific namespace, we have to add the namespace in the spec section

We can switch permanently to a namespace, ex: kubectl config set-context $(kubectl config current-context) --namespace=dev

We can view pods of all namespaces, ex: kubectl get pods --all-namespaces

To limit the resources of a namespace, use quota:


```
apiVersion: v1
kind: ResourceQuota
metadata: 
  name: compute-quota
  namespace: dev
spec: 
  hard:
    pods: "10"
    requests.cpu: "4"
    requests.memory: 5Gi
    limits.cpu: "10"
    limits.memory: 10Gi
```


#### Services

A service enables communication between pods and other elements in or out the cluster

NodePort: A service can listens on a nodeport (called NodePort), it maps a port on the pod to a port on the node, nodeports range is (30000-32767) 

ex:

```
apiVersion: v1
kind: Service 
metadata:
	name: myapp
spec:
	type: NodePort
	ports: 
	 - targetPort: 80 # Port of the pod
	   port: 80 # Port of the service
	   nodePort: 30008 # Port of the node
	selector: 
           app: myapp
	   type: frontend
```
When a nodePort is created, the port is created on all clustet nodes

ClusterIP: An IP of the services to connect to group of pods, it is accessible inside the cluster only, 
ex:

```
apiVersion: v1
kind: Service 
metadata:
	name: backend
spec: 
	type: ClusterIP
	ports:
	 - targetPort: 80
	   port: 80
	selector: 
	  app: myapp
	  type: backend
```

Loadbalancer: Provides load balancing to a group of pods 

### Scheduling

* Manual shceduling

We can shcedule a pod to a specific node by adding a 'nodeName' property in the pod manifest,

ex:

```
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  labels: 
    app: myapp
    type: frontend
spec:
  containers:
    - name: nginx-container
      image: nginx
  nodeName: name
```

To assign an existing pod to another node, we have to create a 'Binding object' and send a POST request to the API

```
apiVersion: v1
kind: Binding
metada:
   name: nginx
target:
  apiVersion: v1
  kind Node
  name: nodename
```

```
curl --header "Content-Type:application/json" --request POST --data '{"apiVersion":"v1", "kind":"Binding" ...}' http://$SERVER/api/v1/namespaces/default/pods/$PODNAME/binding/
```

* Labels & Selectors

A standard method to group objects, is to attach properties (called labels) to these objects, ex: 'application-type: backend' 

Selectors helps us filter these objects, ex: selects all backend applications

Labels can be specified in the metadata/labels section in the manifest file

We can use labels, to make a service route traffic to specific pods, (selector in service: app: frontend) & (label in pod: app: frontend)

* Annotations

Are used to record details abouthe the pod, ex: 'buildversion' 'email' ...

* Taints And Tolerations

Used to specify which container can run on what node, taints are set on nodes, and tolerations are set on pods 

ex: taint a node with 'nfs' taint, and add a toleration to nfs pods to tolerate the nfs taint

Add a taint to a node:

```
kubectl taint nodes node-name key=value:taint-effect
```

The 'taint-effect' defines what will happen to pods that don't tolerate this taint, and it accepts 3 values 

NoSchedule: Pods will not be scheduled on this node 

PreferNoSchedule: The system will try to avoid scheduling pods on this node, but if it doesn't find resources on other nodes, it will schedule them

NoExecute: New pods will not be scheduled on this node, and existing pods on the node will be evicted if they don't tolerate the taint

```
kubectl taint nodes node1 app=blue:NoSchedule
```

Add a toleration to a pod:

```
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec: 
  containers:
  - name: nginx-container
    image: nginx
  tolerations:
  - key: app
    operator: "Equal"
    value: "blue"
    effect: "NoSchedule"
```

Taints & toleration does not tell the pod to go to a specific node, but tells the node to accept pods with specific tolerations, to guarantee that a pod will be scheduled on a specific node, use node affinity

* Node Selectors

We can set a limitation on the pod level, to make them only run on specific nodes, by using node selectors, ex:

in the pod manifest:

```
avpiVersion: v1
kind: Pod
metadata:
 name: myapp
spec: 
  containers:
  - name: data-processing
    image: data-processor

  nodeSelector:
    size: Large
```

The 'size: Large' label must be set on the wanted node

```
kubectl label nodes <node-name> <label-key>=<label-value>
kubectl label nodes node1 size=Large
```

Nodeselectors have limiations, we can not make complex conditions ex:( put the pod on small or medium nodes), that's why we will use node affinity  

* Node Affinity

Node affinity ensures that specific pod with advanced conditions

ex:

```
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity

spec:
containers:
  - name: with-node-affinity
    image: k8s.gcr.io/pause:2.0
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
```

The 'In' operator checks whether  a labels exists in the given list or not, it exists a lot of other operators, ex: 'Exists'.
 
Types of Nodeaffinity:

requiredDuringSchedulingIgnoredDuringExecution: if we schedule a pod with a label 'key: value', and no node on the cluster have this label, the pod will NOT be scheduled

preferredDuringSchedulingIgnoredDuringExecution: if we schedule a pod with a label 'key: value', and no node on the cluster have this label, the pod will be scheduled

|       | DuringScheduling | DuringExecution |
|-------|------------------|-----------------|
| Type1 | Required         | Ignored         |
| Type2 | Preffered        | Ignored         |
| Type3 | Required         | Required        |

* Node Affinity VS Taints & Tolerations

Both techniques can be combined to dedicate specific nodes to specific pods and prevent some pods from being scheduled on specific nodes

* Resource Requirements and Limits

Resources are specified on three sections (CPU, Memory & storage), the scheduler lokks for a node with sufficient resources beforse affecting a pod to a node, if no node has the necessary amount of resources, the pod remains in 'Pending' state, 

By default, k8s assumes each container in a pod requires 0.5 CPU and 256 Mi memory, this is called Resource Requests, to modify the request resources:

  
```
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
  labels: 
    name: simple-webapp-color
spec:
  containers:
  - name: simple-webapp-color
    image: myimage
    ports:
      - containerPort: 8080
    resources:
      requests:
	memory: "1Gi"
	cpu: 1
```

CPU:

| Unit  | Equivalent    |
|-------|---------------|
| 1 CPU | 1 AWS vCPU    |
| 1 CPU | 1 GCP Core    |
| 1 CPU | 1 Azure Core  |
| 1 CPU | 1 Hyperthread |

MEMORY:

1 Mi = 1 Mebibyte = 1,048,576 bytes

Resource Limits: by default, k8s sets a limit of 1 vcpu, and 512 Mi, these limits can be changed in the spec section


```
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
  labels: 
    name: simple-webapp-color
spec:
  containers:
  - name: simple-webapp-color
    image: myimage
    ports:
      - containerPort: 8080
    resources:
      requests:
	memory: "1Gi"
	cpu: 1
    limits:
	memory: "2Gi"
	cpu: 2
```

Exceed limits: if a container tries to consume more than the cpu limit, k8s prevent it (hard limit), in the case of memory, a container can consume more memory, but if it does often, it will be terminated

* DaemonSets

A daemonset is a replication controller which ensures that 1 instance of pod is running on each node on the cluster, if a new node is added to the cluster, a new pod will be scheduled in it, Use cases: (Monitoring Solution - Logs Viewer)

ex:

```
apiVersion: apps/v1
kind: DaemonSet          
metadata: 
  name: myapp-rc
  labels: 
    app: myapp
    type: frontend
spec:
  template:
    metadata:
      name: myapp
      labels:                      
        app: myapp         
        type: frontend
    spec:
      containers:
        - name: nginx-container
          image: nginx
  replicas: 3
  selector: 
    matchLabels:
        type: frontend
```

To view the existing daemonsets:

```
kubectl get daemonsets
```

Daemonsets uses nodeaffinity to place one pod into one node

* Static PODs

The kubelet can manage a node independently, if it's isolated from the the control plan, the only task that kubelet can do is to create pods, we can make the kubelet creates pods, without passing by the apiserver, by putting our manifests in '/etc/kubernetes/manifests' directory, kubelet poriodecly checks this directory and reads the files and creates the pods, and monitor them, these pods are known as 'Static Pods', we can put our manifests in a different directory, and pass the path as an argument to kubelet,ex:

```
--pod-manifest-path=/etc/kubernetes/static
```

or we can configure a file that contains the path in systemd unit, ex:

```
--config=kubeconfig.yaml
```
kubeconfig.yaml content:

```
staticPodPath: /etc/kubernetes/manifests
```

Kubelet can take inputs from many sources (manifets directory - apiServer) at the same time, and the apiserver will be aware of the pods created from the static directory, because kubelet creates a mirror object for the pod in the apiserver, we can only view static pods via the api, not modify or delete them.

* Multiple Schedulers

We can write our own scheduler, and have as the main or additional schedulers, k8s can have multiple schedulers at the same time

The scheduler configuration: It's mandatory to give a different name for your new scheduler, to avoid conflicts

```
--scheduler-name= my-custom-scheduler
```

Another important parameter is the '--leader-elect=True', when we have a multimaster cluster, only one scheduler can be active at a time, if we have a new scheduler in a multimaster cluster, we have to set '--lock-object-name=my-custom-scheduler' 

After starting our scheduler, we have to configure the pod to use the new scheduler, ex: 

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec: 
  containers:
  - image: nginx
    name: nginx

  schedulerName: my-custom-scheduler
```

To view the logs of the scheduler

```
kubectl logs my-custom-scheduler --namespace=kube-system
```

* Configuring kubernetes scheduler

```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    component: kube-scheduler
    tier: control-plane
  name: kube-scheduler
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-scheduler
    - --authentication-kubeconfig=/etc/kubernetes/scheduler.conf
    - --authorization-kubeconfig=/etc/kubernetes/scheduler.conf
    - --bind-address=127.0.0.1
    - --kubeconfig=/etc/kubernetes/scheduler.conf
    - --leader-elect=true
    image: k8s.gcr.io/kube-scheduler:v1.18.3
    imagePullPolicy: IfNotPresent
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 10259
        scheme: HTTPS
      initialDelaySeconds: 15
      timeoutSeconds: 15
    name: kube-scheduler
    resources:
      requests:
        cpu: 100m
    volumeMounts:
    - mountPath: /etc/kubernetes/scheduler.conf
      name: kubeconfig
      readOnly: true
  hostNetwork: true
  priorityClassName: system-cluster-critical
  volumes:
  - hostPath:
      path: /etc/kubernetes/scheduler.conf
      type: FileOrCreate
    name: kubeconfig
status: {}

```

### Logging & Monitoring

* Monitoring

Resource consumption: 

Node Level:

- number of nodes in the cluster 
- nodes heath status
- Performance metrics (CPU - Memory - Disk - Network)

Pod Level:

- Number of pods 
- Performance metrics of each pod (CPU - Memory - Network)

Available 3rd party solution: 

- Prometheus
- Elastic Stack
- DATADOG
- dynatrace

All metrics are generated by the 'cAdvisor' which is a component of the kubelet agent

To implement the metric server on a kubernetes cluster:

Link: https://github.com/kubernetes-sigs/metrics-server

Or apply the following manifest

```
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:aggregated-metrics-reader
  labels:
    rbac.authorization.k8s.io/aggregate-to-view: "true"
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
rules:
- apiGroups: ["metrics.k8s.io"]
  resources: ["pods", "nodes"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: metrics-server:system:auth-delegator
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: metrics-server-auth-reader
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: extension-apiserver-authentication-reader
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
---
apiVersion: apiregistration.k8s.io/v1beta1
kind: APIService
metadata:
  name: v1beta1.metrics.k8s.io
spec:
  service:
    name: metrics-server
    namespace: kube-system
  group: metrics.k8s.io
  version: v1beta1
  insecureSkipTLSVerify: true
  groupPriorityMinimum: 100
  versionPriority: 100
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: metrics-server
  namespace: kube-system
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metrics-server
  namespace: kube-system
  labels:
    k8s-app: metrics-server
spec:
  selector:
    matchLabels:
      k8s-app: metrics-server
  template:
    metadata:
      name: metrics-server
      labels:
        k8s-app: metrics-server
    spec:
      serviceAccountName: metrics-server
      volumes:
      # mount in tmp so we can safely use from-scratch images and/or read-only containers
      - name: tmp-dir
        emptyDir: {}
      containers:
      - name: metrics-server
        image: k8s.gcr.io/metrics-server-amd64:v0.3.6
        imagePullPolicy: IfNotPresent
        args:
          - --cert-dir=/tmp
          - --secure-port=4443
          - --kubelet-preferred-address-types=InternalIP
          - --kubelet-insecure-tls
        ports:
        - name: main-port
          containerPort: 4443
          protocol: TCP
        securityContext:
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
        volumeMounts:
        - name: tmp-dir
          mountPath: /tmp
      nodeSelector:
        kubernetes.io/os: linux
        kubernetes.io/arch: "amd64"
---
apiVersion: v1
kind: Service
metadata:
  name: metrics-server
  namespace: kube-system
  labels:
    kubernetes.io/name: "Metrics-server"
    kubernetes.io/cluster-service: "true"
spec:
  selector:
    k8s-app: metrics-server
  ports:
  - port: 443
    protocol: TCP
    targetPort: main-port
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:metrics-server
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - nodes
  - nodes/stats
  - namespaces
  - configmaps
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:metrics-server
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:metrics-server
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
```

Then, retrieve metrics

```
kubectl top nodes
kubectl top pods
```

* Logging

To see the logs of a specific container:

```
kubectl logs -f pod-name
```

If the has multiple containers:

```
kubectl logs -f pod-name container-name
```

### Application Lifecycle Management

* Rolling Updates

The default deployment strategy in k8s is rolling updates, bringing up 1 pod of the new version & shutting down one pod of the older version, & continue until all pods are updated, k8s keeps a revision for each time we update a deployment.

To see the status of the rollout of a deployment

```
kubectl rollout status deployment/deployment-name
```

To see the history of revisions

```
kubectl rollout history deployment/deployment-name
```

There are two Deployment Strategies in k8s:

Recreate 
Rolling updates (Default)

To update the deployment, edit the manifest file, and then run:

```
kubectl apply -f deployment.yaml
```

We can edit the version of the image from the command line:

```
kubectl set image deployment/deployment-name image:newtag
```

But this is not recommended, as it will keep the older tag in the manifest file

Upgrades: when a deployment is updated, it creates a new replicaset, and starts deploying pods there, this can be seen by listing the replicasets:

```
kubectl get replicasets
```

In order to rollback a deployment:

```
kubectl rollout undo deployment/deployment-name
```

PS: kubectl RUN: this command creates a deployment & not just a pod

Deployments commands cheatsheet:

| Command                                                   | Task                                                          |
|-----------------------------------------------------------|---------------------------------------------------------------|
| kubectl create -f deployment.yml                          | Create a new deployment described in the manifest file        |
| kbectl get deployments                                    | List the available deployments in the current namespace       |
| kubectl apply -f deployment.yml                           | Apply changes in the manifest file                            |
| kubectl set image deployment/deployment-name image:newtag | Change the version of the image used in a specific deployment |
| kubectl rollout status deployment/deployment-name         | Get the satus of a rollout of a specific deployment           |
| kubectl rollout history deployments                       | List the history of rollouts of a specific deployment         |



* Commands in pod definition file

Reminder (Docker)

In Docker, we specify a CMD entry, in the dockerfile, to indicate which process will start the container, the CMD entry accepts arguments, and can be overriden from the docker run command.

To specify just the argument in the docker run command and not all the command, we have to use the entrypoint entry, ex:

```
ENTRYPOINT ["sleep"]
docker run myimage 5 
```

in order to specify a default argument, we combine both entrypoint and cmd, ex:

```
ENTRYPOINT ["sleep"]
CMD ["5"]
```

If we don't specify an argument, the 5 value will be used

To override the entrypoint, use the '--entrypoint <command>' with the docker run command

We can apply the same concepts to kubernetes pods, to override the entrypoint we will use 'command', and to override the argument, we will use 'args', ex:

```
apiVersion: v1
kind: Pod
metadata: 
  name: sleeper
spec: 
  containers:
  - name: sleeper
    image: ubuntu-sleeper
    command: ["sleep2.0"]
    args: ["10"]
```

* Environment Variables 

We can set an environment variable in the pod as follows:

```
apiVersion: v1
kind: Pod
metadata: 
  name: environment
spec: 
  containers:
  - name: environment
    image: nginx-sleeper
    env:
      - name: key
        value: value
```

env is a list, it can contain multiple variables.

* Configmaps

When specifying a long list of variables into a pod, the manifest file will be a mess, that's why we will store all environment variables in a single file called configmap, and ask the container to load env variables from that file, ex:

Pod configuration:

```
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec: 
  containers:
  - name: mypod
    image: httpd
    envFrom:
    - configMapRef:
         name: <Name-of-the-configmap>
```

Configmap manifest:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DB_URL: xxxxx
  BD_USER: yyyyy
  DB_PASSWORD: zzzz
```

We can create configmaps in an imperative way, ex:

```
kubectl cteate configmap configmap-name --from-literal=<key>=<value>
```

Or from a file that contains key: value data, ex:

```
kubectl create configmap configmap-name --from-file=filename
```

* Secrets

Secrets are similar to configmaps, but they are used to store sensitive data, secrets encode the data into a base64 format before storing them, there are two ways to create a secret, imperative and declarative, ex:

Imperative

```
kubectl create secret generic secret-name --from-literal=<key>=<value>
```

Or from a file

```
kubectl create secret generic secret-name --from-file=filename
```

Declarative

```
apiVersion: v1
kind: Secret
metadata: mysecret
data:
  DB_HOST: encoded_value
  DB_USER: encoded_value
  DP_PASSWORD: encoded_value
```

To encode a value, type the following command:

```
# echo -n 'value' | base64
```

To decode an encoded base64 value:

```
# echo -n 'encoded_value' | base64 --decode
```

After creating the secret, we have to mount in in the pod,

```
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec: 
  containers:
  - name: mypod
    image: httpd
    envFrom:
    - secretRef:
         name: <Name-of-the-secret>
```

* Multi Container PODs

It's recomended to run one container per Pod, but sometimes, we may need to run multiple containers in one single pod (an pllication and a log agent - a side car container ...), in order to run multiple containers in one pod, we have to declare them in the manifest file, ex:

```
apiVersion: v1
kind: Pod
metadata: 
  name: composed
spec:
  containers:
  - name: app-container
    image: webapp
  - name: log_agent
    image: log-agent
```

### Cluster Maintenance

* OS Upgrades

If a node is down, kubernetes waits for 5 minutes (pod eviction timeout), then, is the pods are part of a replicaset, creates the pods in another node

'Pod Eviction Timeout' is a paramater of the controller manager component 'default: --pod-eviction-timeout=5m0s'

A best practice to upgrade node's os, is to drain the node, so all pods are moved to other nodes, ex:

```
kubectl drain nodename
```

When a node is drained, it's marked as unschedulable, after upgrading the os, and when the node is ready again to join the cluster, we have to remove the drain:

```
kubectl uncordon nodename
```

Another way to mark the node as unschedulable, is the cordon command, but it doesn't terminate the pods on the node, it simply makes sure that new pods are not scheduled on this node:

```
kubectl cordon nodename
```

* Kubernetes Releases

k8s follows a standard versioning strategy, major, minor and bugfixes, an alpha where new features and bugfixes are there but disabled, a beta where new features and bugfixes are there & enabled, then they are merged t the stable version again.

When downloading k8s control plan, all components will be on the same version (apiserver - controller-manager - scheduler - kubelet - kubeproxy), other components like kubelet, etcd and coreDNS are separate projects, so they will have different versions

* Cluster Upgrade Process

It's MANDATORY to have the same version for all control plan components

As the kube-apiserver is the main component in the cluster, it must always have the higher version in the cluster, other components can have lower versions than the API, but the opposit is not true, except the kubectl utility

At anytime, kubernetes supports only the 3 latest minor versions, so we should keep our cluster near the latest version, ex: when v1.13 is released, only 1.12 and 1.11 will be supported beside it, v1.10 and lower versions will not be supported

The recommended approach of upgrade is to upgrade one minor version at a time, ex: from v1.12 to v1.13 

The upgrade process depends on the way we deployed the cluster:

For managed services, it's done via a gui of the cloud

For kubeadm it's done by 'kubectl upgrade apply'

The hardest way is when we deploy from scratch

Upgrading a cluster have two major steps:

  - Upgrading Master Nodes

  - Upgrading Worker Nodes 

When we are upgrading our master nodes, the worker nodes are not impacted, all applications will be up, unless a pod gets terminated, it will not be restarted because the controller-manager is down

The recommended way to upgrade worker nodes is to upgrade one at a time

Another way of upgrading worker nodes is to add new nodes with the newer version installed, and remove the older ones, one at a time

The kubeadm way:

In order to check is there are new upgrades available:

```
kubeadm upgrade plan
```
 
This command will give us all the infos and steps needed to upgrade the cluster

the first step is to upgrade the kubeadm itself

```
apt-get upgrade -y kubeadm=1.19.0-00
```

Then, upgrade the control plan

```
kubeadm upgrade apply v1.19.0
```

Then upgrade the kubelet on all nodes manually

```
apt-get upgrade -y kubelet=1.19.0-00
systemctl restart kubelet
```

For the worker nodes upgrade, we'll start by draining the node

```
kubectl drain nodename
```

Then, upgrade kubeadm and kubelet on the worker node

```
apt-get upgrade -y kubeadm=1.19.1-00
apt-get upgrade -y kubelet=1.19.0-00
kubeadm upgrade node config --kubelet-version v1.19.0
systemctl restart kubectl
```

Finally, we'll undrain the node

```
kubectl uncordon nodename
```

Then, the next nde & so on

* Backup and restore 

Backup Candidats:

Manifests: must be stored on a git repository, we can retrieve manifests directly from the API server:

```
kubectl get all --all-namespace -o yaml > all-deploy-services.yaml
```
Or we can use third party software like 'velero' to backup the cluster

ETCD Cluster: 

We have to backup the datadir of etcd, default is '/var/lib/etcd' 

Or backup the database using the ETCDCTL utility:

```
ETCDCTL_API=3 etcdctl snapshot save snapshot.db
```

we can see the status of the backup:

```
ETCDCTL_API=3 etcdctl snapshot status snapshot.db
```

To restore the cluster from this backup, stop the kube-apiserver service, then, run the following command

```
ETCDCTL_API=3 etcdctl snapshot restore snapshot.db --data-dir /var/lib/etcd-from-backup --initial-cluster master1=https://1.1.1.1:2380,master2=https://2.2.2.2:2380 --initial-cluster-token etcd-cluster-1 initial-advertise-peer-urls https://${INTERNAL_IP}:2380
```

NB: the token must be new

& then, edit etcd config file, to use the new data directory and the new token, then reload the etcd service & kube-apiserver

NB: if ssl is used we have to specify the ca, and certificate in the command

Each way of these two are used in separate environments, for managed services, it's recommended to backup the manifests

Persistent volumes

### Security

* Security primitives

Securing hosts: (Password-based authentication disabled - SSH key based authentication - ssh root login disabled - firewall - Fail2ban ...)

Kube-apiserver: 
   
Who can access (Authentication): Files Username & password - Files Username & Tokens - Certificates - External Authentication providers - LDAP ... - Service Accounts (For machines)
What can they do (Authorization): RBAC Authorization - ABAC - Node Authorization - Webhook Mode 

All communications between cluster components must be secured using tls encryption

We can restrict communications between k8s components by using network policies

* Authentication

Users types: Admins - Developers - Application End-Users - Bots

Admins and Developrs are managed via user accounts, k8s doesn't manage users, this can be done via files/Certificates/Directory-service 

All requests to the API server are authenticated then processed

Authentication mechanisms: Static Password File - Static Token File - Certificate - Identity Services

###### Static Password File

```
cat user-details.csv
password123,user123,u0001
password456,user456,u0002
```

The format of the csv file is 'password,username,uid'

In order to make the api-server use this file for authentication, we have to add the following line in the start command

```
/usr/local/bin/kube-apiserver --basic-auth-file=user-details.csv
```

If you setup your cluster using kubeadm tool, then modify the '/etc/kubernetes/manifests/kube-apiserver.yaml'

If we want to add the user to a specific group, we can add a 4rd column in the csv file

###### Static Token file

Similarly to the static password file, a token file contains four columns:

token,user,uid,group

And we have to add the following flag to kube-apiserver command:
 
```
/usr/local/bin/kube-apiserver --token-auth-file=user-details.csv
```

#### NB: This method of authentication is not recommended

* TLS

Recap of Public Key Infrastructure

* TLS in kubernetes

Naming conventions: certificate (Public keys) must contain .crt or .pem extensions, and private keys must have .key or *-key.pem extensions

In k8s, all communications between components must be secured via certificates, even the access of the admin

We need to create server certificates for servers, and client certificates for clients

Server Components:

kube-apiserver: apiserver.crt, apiserver.key

Etcd server: etcdserver.crt, etcdserver.key

Kubelet: kubelet.crt, kubelet.key 

Client components:

Administrators: admin.crt, admin.key

Scheduler: scheduler.crt, scheduler.key

Kube-controllermanager: controller-manager.crt, controller-manager.key

kube-proxy: kube-proxy.crt, kube-proxy.key

We need a certificates authority to sign and valid all certificaes, and it will have its own certificate, ca.crt and ca.key

####### Generating the certificates

Tools can be used: cfssl - openssl - easyrsa

Openssl:

CA:

```
key:
openssl genrsa -out ca.key 2048
ca.key

csr:
openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr 
ca.csr

cert:
openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt
```

Admin user:

```
key:
openssl genrsa -out admin.key 2048

csr:
openssl req -new -key admin.key -subj "/CN=kube-admin/O=system:masters" -out admin.csr

cert:
openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -out admin.crt
```

Scheduler: Same as admin user, the only thing to add is the prefix system in the CN part 'SYSTEMKUBE-SCHEDULER'

Controller Manager: CN is 'SYSTEMCONTROLLER-MANAGER'

kube-proxy: CN is SYSTEMKUBE-PROXY

Etcd: CN is 'ETCD-SERVER', in the case of an etcd cluster, we need to create a key/cert pair for each node

Apiserver: The CN is KUBE-APISERVER, but it needs more SANS ans IPs, such as [kubernetes - kubernetes.default - kubernetes.default.svc - kubernetes.default.svc.cluster.local - 127.0.0.1 - THE INTERNAL IP ADDRESS ]

to specify the alternative names, we need to create a configuration file:

```
[req]
req_extensions = v3_req
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage: nonRepudiation,
subjectAltName = @alt_names
[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
IP.1 = 192.168.1.11
IP.2 = 127.0.0.1
```

And pass this configuration file to the csr command:

```
openssl -new -key apiserver.key -subj "CN=kube-apiserver" -out apiserver.csr -config openssl.cnf
```

All this files must be passed to the apiserver command:

--client-ca-file
--tls-cert-file
--tls-private-key
--etcd-cafile
--etcd-certfile
--etcd-keyfile

Kubelet: Each node on the cluster needs to have its own kubelet certificate, whiche will be named after the name of the node, after adding the certs, we need to add them in the configuration file 

authentication:
   x509:
      clientCAFile: 'ca.pem'
tlsCertFile: node1.crt
tlsPrivatekeyFile: node1.key

In order for the kubelet to communicate to the apiserver, we will create a client certificates for each node, with the CN 'system:node:nodename' And the group 'O=SYSTEM:NODES'

To list the content of the certificate:

```
openssl x509 -in cert.crt -text -noout 
```

When setting up the cluster the hard way, look for the logs to see the issues 'journalctl -u etcd.service -l' 

###### Certificate API

K8s offers an API managed by the controller manager to sign certificates, while new people are joining the administrator's team, it will be a tough task to manage certificates manually, every new user can generate a key, then a csr, then submit it to k8s, and the administrator can view the csrs and approve them, then the certificate can be retrieved and sent to the apropriate user

In order to create a certificateSigningRequest Object:

1) Create a key:

```
openssl genrsa -out newuser.key 2048
```

2) Create a csr:

```
Openssl req -new -key newuser.key -subj "/CN=newuser" -out newuser.csr
```

3) Send the csr to the administrator so he can create a signingrequest object as follows:

```
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: newuser
spec:
  groups:
  - system: authenticated
  usages:
  - digital signature
  - key encipherment
  - server auth
  request:
    This fialed will contain the csr encoded in base64
```

To list all the csrs:
  
```
kubectl get csr
```

To approve the request:

```
kubectl certificate approve newuser
```

After approving the certificate, we can view it:

```
kubectl get csr newuser -o yaml
```

This will display the certificate encoded id base64

In order to make the controller manager sign the certificates, we need to specify the cert and the key of the ca in the starting command:

```
--cluster-signing-cert-file=/path/to/cert
--cluster-signing-key-file=/path/to/key
```

* Kubeconfig

A kubeconfig file is used to avoid typing the master ip and the path of the certificate and key each time with the kubectl command

By default, kubectl looks for the kubeconfig file in '$HOME/.kube/config', if not found, we have to identify the path of the config file, '--kubeconfig config'

The kubeconfig file has 3 sections:

Clusters: Which contains the list of clusters you have access to, ex: development, Production and test

Users: User accounts to access these clusters, ex: Admin, Dev User, Prod User

Contexts: defines the relation between a user and a cluster, which user will be used to connect to each cluster, ex: Admin@Production, means the Admin account will be used to connect to Production cluster

Example:

```
apiVersion: v1
kind: Config

clusters: 
- name: my-kube-playground
  cluster:
    certificate-authority: ca.crt
    server: https://my-kube-playground:6443

contexts:
- name: my-kube-admin@my-kube-playground
  context:
    cluster: my-kube-playground
    user: my-kube-admin

users:
- name: my-kube-admin
  user:
    client-certificate: admin.crt
    client-key: admin.key
```

In case of specifiyng multiple contexts in the configuration file, we need to add a default context:

```
current-context: dev-user@google
```

To change the context used, run the following command:

```
kubectl config use-context prod-user@prod
```

If your cluster has multiple namespaces, you can specify the ne you want under the contexts section:

```
contexts:
- name: admin@production
  context: 
    cluster: production
    user: admin
    namespace: production
```

Instead of specifiyng the path of certificates, we can specify the certificate content encoded in base64, ex:

```
clusters:
- name: prod
  cluster:
    certificate-authority-data: 'paste here'
```

* API Groups

k8s API is devided into multiple api groups, ex: /metrics, /healthz, /version, /api, /apis, /logs

The APIs that are responsible of the cluster functionality are /api and /apis

/api: core group, /v1, responsible of rc, events, namespaces, pods, configmaps, secrets, services and so on

/apis: named groups which are more organized, responsible if /apps, /extensions, /networking.k8s.io, /storage.k8s.io ...

Each object in k8s is associated with an API, ex: /apis has /apps which has /v1 which has /deployments, and wa can interact with each object by using verbs, ex: list, get, create ...

* RBAC

In order to create a role, ex:

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""] #leave this blank for core groups
  resources: ["pods"]
  verbs: ["list", "get", "create", "update", "delete"]
- apiGroups: [""]
  resources: ["ConfigMap"]
  verbs: ["create"]
```

After creating the role, we have link the user to the group, we call this rolebinding, ex:

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: devuser-developer-binding
  namespace: production #Optional
subjects:
- kind : User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

To list the roles  and rolebinding:

```
kubectl get roles
kubectl get rolebindings
```

For users, to check whether they have access to some resources, they can use the following command:

```
kubectl auth can-i <command>
ex:
kubectl auth can-i delete pod
```

The output will be 'yes' or 'no'

As an administrator, to test the permissions of some user, you can impersonate their account

```
kubectl auth can-i create deployments --as dev-user
```

In the role definition file, we can specify resource name to limit the persmissions, for ex, if we have 4 pods in default namespace, we can give access to only one pod:

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""] #leave this blank for core groups
  resources: ["pods"]
  verbs: ["list", "get", "create", "update", "delete"]
- apiGroups: [""]
  resources: ["ConfigMap"]
  verbs: ["create"]
  resourceNames: ["orange", "blue"]
```

* Cluster Roles & Cluster RoleBindings

Objects in k8s are categorized into:

namespaced: Resources that are know in a specific namespace such as pods, replicasets, jobs, deployments, services, roles, rolebindings, configmaps ...

Cluster scoped Resources that are known in all namespaces such as nodes, PV, clusterroles, clusterrolebindings, namespaces and certificatesigningrequests

To get the full list of namespaced resources or cluster scoped resources, run the following command:

```
kubectl api-resources --namespaced=true
or
kubectl api-resources --namespaced=false
```

To give a user access to a cluster scoped resource, we use cluster roles and cluster rolebindings

ex:

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-administrator
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["list", "get", "create", "delete"]
```

To link the user to this clusterrole, we will use a clusterrolebinding:

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-role-binding
subjects:
- kind: User
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-administrator
  apiGroup: rbac.authorization.k8s.io
```

We can create cluster roles for namespaced objects too, ex, a cluster role for pod, will give access to all pods in all namespaces

* Image Security

When specifiyng a docker image name, such as 'nginx' docker pulls the image from the public repository 'docker.io'

In case we create our private repository, we can specify the credentials in a secret, and then specify the secret in the pod definition, ex:

Secret:

``` 
kubectl create secret docker-registry regcred --docker-server=privatr-registry.io --docker-username=registry-user --docker-password=registry-password --docker-email=mail@org.com
``` 

Pod definition:

``` 
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec: 
  containers:
  - name: nginx
    image: private-registry.io/apps/internal-app
  imagePullSecrets:
  - name: regcred
``` 

The naming convention of docker images is repository-address/username/image-name


* Security Contexts 

We can add security contexts such as the uid of the user who will run the container, this can be specified at the level of the container or the pod, if specified in both, those in the container will override those at the pod level, ex:

``` 
apiVersion: v1
kind: Pod
metadata: 
  name: web-pod
spec:
  containers: 
    - name: ubuntu
      image: ubuntu
      command: ["sleep", "3600"]
      securityContext:
        runAsUser: 1000
        capabilities:
           add: ["MAC_ADMIN"]
``` 

* Network Security

A network policy is used to restrict access between pods, for example, we can allow traffic to port 3306 in a db pod, only from the backend pod

Network traffic in k8s is devided into 2 types, ingress (incoming) and Egress (outgoing)

We use labels and selectors to link a specific network policy to one or a set of pods

ex:

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyType: 
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          name: api-pod
    port:
    - protocol: TCP
      port: 3306
```

All network solutions support network policies, expet flannel












































Coupon : DCUBEOFFER
