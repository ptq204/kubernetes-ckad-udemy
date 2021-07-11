## Kubernetes core concepts  

### 1. Components
**API server**
- front end of k8s
- user managemen devices
- CLI
- interact with k8s

**etcd**  
- distributed reliable KV store
- store data used to manage the cluster, ex: info of nodes in cluster
- implement locks to ensure no conflicts bewteen masters

**Scheduler**  
- distibuting work on containers accross nodes.
- create containers and assign them to nodes.  

**Controller**  
- Orchestration  
- notice and response when nodes, containers or end points go down.  
- make decision to bring up container in above case

**Container runtime**  
- software that used to run containers

**kubelet**  
- make sure the containers are running on the nodes.  

### 2. Build YAML file   
#### 2.1 POD  
```yml
# pod: v1
# service: v1
# replicaSet: apps/v1
# deployment: apps/v1
apiVersion: v1
kind: Pod
metadata:
    name: myapp-pod
    namespace: dev
    # labels to distinguish containers, pods
    # add any as you like
    labels:
        app: myapp
        type: back-end
spec:
    # List of containers
    containers:
        - name: nginx-container
          image: nginx
          # expose pod to cluster
          ports:
          - containerPort: 8088
```  
#### 2.2 Replication controller  
```yml
apiVersion: v1
kind: ReplicationController
metadata:
    name: myapp-rc
    # labels to distinguish containers, pods
    # add any as you like
    labels:
        app: myapp
        type: backend
spec:
    template:
        metadata:
            name: myapp-pod
            labels:
                app: myapp
                type: backend
        spec:
            containers:
            -   name: nginx-container
                image: nginx
    replicas: 3
```  

#### 2.3 Replica Set  
```yml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
    name: myapp-replicaset
    labels:
        app: my-app
        type: backend
spec:
    template:
        metadata:
            name: myapp-pod
            labels:
                app: myapp
                type: backend
        spec:
            containers:
            -   name: nginx-container
                image: nginx
    replicas: 3
    # selector indentifies what pods fall under replicaset
    # because replicaset also manages pods which are not part of that replicaset
    selector:
        matchLabels:
            type: backend
```

#### 2.4 Deployment  
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
    name: myapp-replicaset
    labels:
        app: my-app
        type: backend
spec:
    template:
        metadata:
            name: myapp-pod
            labels:
                app: myapp
                type: backend
        spec:
            containers:
            -   name: nginx-container
                image: nginx
    replicas: 3
    # selector indentifies what pods fall under replicaset
    # because replicaset also manages pods which are not part of that replicaset
    selector:
        matchLabels:
            type: backend
```  

#### 2.5 Namespace  
```yml
apiVersion: v1
kind: Namespace
metadata:
    name: dev
```  

#### 2.6 Quota - limit resources of namespace  
```yml
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

### 3. Commands  
#### 3.1 Pod  

Create pod based on YAML file:
```sh
kubectl create -f pod-definition.yml
```  

Get list of po  
```sh
kubectl get pods
```  

Get details of the pod  
```sh
kubectl describe pod <pod-name>
```  

Run a pod with predefined image
```sh
kubectl run <pod-name> --image=nginx
```  

Delete a pod
```sh
kubectl delete pod <pod-name>
```  

Extract pod's definition to a file  
```sh
kubectl get pod <pod-name> -o yaml > pod-definition.yaml
```

Edit pod properties  
```sh
kubectl edit pod <pod-name>
```  

#### 3.2 Replication controller  

Create replica set based on YAML file  
```sh
kubectl create -f rc-definition.yml
```  

Get list of created replication controller  
```sh
kubectl get replicationcontroller
```  

#### 3.3 Replica Set  

Create replica set based on YAML file  
```sh
kubectl create -f replicaset-definition.yml
```  

Get list of replica set  
```sh
kubectl get replicaset
```  

Update replica set with edied YAML file  
```sh
kubectl replace -f replicaset-definition.yml
```  

Scale number of replicas  
```sh
kubectl scale --replicas=6 -f replicaset-definition.yml
```  

Scale number of replicas based on replicaset name  
```sh
kubectl scale --replicas=6 replicaset <replicaset-name>
```  

Delete replicaset  
```sh
# Also delete underlying PODs
kubectl delete replicaset <replicaset-name>
```  

Edit replicaset  
```sh
kubectl edit replicaset <replicaset-name>
```  

#### 3.4 Deployments  

- Create deployment  
```sh
kubectl create deployment --image=<image-name> <deployment-name>
```  
**note**: create deployment auto create replicaset  

- Create deployment based on YAML file
```sh
kubectl create -f deployment-definition.yml
```  

- Get list of all objects created  
```sh
kubectl get all
```  

- Generate Deployment with number of Replicas
```sh  
kubectl create deployment nginx --image=nginx --replicas=<numer-of-replicas>
```  

- Scale deployments  
```sh
kubectl scale deployment nginx --replicas=4
```  

#### 3.5 Output format  
syntax:  
> kubectl [command] [TYPE] [NAME] -o <output_format>  

`-o json`: Output a JSON formatted API object.  

`-o name`: Print only the resource name and nothing else.  

`-o wide`: Output in the plain-text format with any additional information.  

`-0 yaml`: Output a YAML formatted API object.  

for example:  

```sh
master $ kubectl create namespace test-123 --dry-run -o json
{
    "kind": "Namespace",
    "apiVersion": "v1",
    "metadata": {
        "name": "test-123",
        "creationTimestamp": null
    },
    "spec": {},
    "status": {}
}
master $
```  

```sh
master $ kubectl create namespace test-123 --dry-run -o yaml
apiVersion: v1
kind: Namespace
metadata:
  creationTimestamp: null
  name: test-123
spec: {}
status: {}
```  

#### 3.6 Namespace  
Get list of pods under namespace:  
```sh
kubectl get pods --namespace=<name-space>  
```  

Create pod with specific namespace:  
```sh
kubectl create -f pod-definition.yaml --namespace=<name-space>
```  

Create namespace based on YAML file:  
```sh
kubectl create -f namspace-definition.yaml
```  

Create namespace:  
```sh
kubectl create namespace <name-space>
```  

List all pods in all namespaces:  
```sh
kubectl get pods --all-namespaces
```  

Switch to target namespace:  
```
kubectl config set-context $(kubectl config current-context) --namespace=<name-space>
```  

#### 3.7 Imperative commands  

`--dry-run`: By default as soon as the command is run, the resource will be created. If you simply want to test your command, use the --dry-run=client option. This will not create the resource, instead, tell you whether the resource can be created and if your command is right.  

`-o yaml`: This will output the resource definition in YAML format on the screen.  

Use the above two in combination to generate a resource definition file quickly, that you can then modify and create resources as required, instead of creating the files from scratch.  

- Generate POD Manifest YAML file (-o yaml). Don't create it(--dry-run)  
```sh
kubectl run nginx --image=nginx  --dry-run=client -o yaml
```  

```sh  
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > nginx-deployment.yaml
```  

- Create a Service named redis-service of type ClusterIP to expose pod redis on port 6379 (This will automatically use the pod's labels as selectors).  
```sh
kubectl expose pod redis --port=6379 --name redis-service --dry-run=client -o yaml
```  

or  (This will not use the pods labels as selectors, instead it will assume selectors as app=redis. You cannot pass in selectors as an option. So it does not work very well if your pod has a different label set. So generate the file and modify the selectors before creating the service):  

```sh
kubectl create service clusterip redis --tcp=6379:6379 --dry-run=client -o yaml  
```  


- Create a Service named nginx of type NodePort to expose pod nginx's port 80 on port 30080 on the nodes (This will automatically use the pod's labels as selectors, but you cannot specify the node port. You have to generate a definition file and then add the node port in manually before creating the service with the pod.):  

```sh
kubectl expose pod nginx --port=80 --name nginx-service --type=NodePort --dry-run=client -o yaml
```  

or (This will not use the pods labels as selectors):  

```sh
kubectl create service nodeport nginx --tcp=80:80 --node-port=30080 --dry-run=client -o yaml
```  

Both the above commands have their own challenges. While one of it cannot accept a selector the other cannot accept a node port. I would recommend going with the `kubectl expose` command. If you need to specify a node port, generate a definition file using the same command and manually input the nodeport before creating the service.  







### 4. Replication controller  
- Help running multiple instances of single pod in k8s cluster => high availability.  
- If there is one single pod => auto bring up new one if an existing one fails.  
- The new one of this is **Replica Set**.  
