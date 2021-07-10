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

### 4. Replication controller  
- Help running multiple instances of single pod in k8s cluster => high availability.  
- If there is one single pod => auto bring up new one if an existing one fails.  
- The new one of this is **Replica Set**.  
