## Kubernetes configuration  

### 1. Recap Docker

#### 1.1 Docker Commands & Arguments  

Suppose we have an dockerfile:  
- `ENTRYPOINT` will append the argument, whereas `CMD` will replace the argument.  

example:  
```Dockerfile
FROM image

CMD ["sleep", "5"]
```  

then this command will replace `5` by `10`:  

```sh
# This will sleep 10 instead of sleep 5
docker run --name sleeper image-name 10
```  

If use `ENTRYPOINT`, we can dynamically pass argument because it will be appended to the command in `ENTRYPOINT`:  
```Dockerfile
FROM image

ENTRYPOINT ["sleep"]
```  

Then we run:  
```sh
docker run --name sleeper image-name 10
```  

If we want to change the arguments in `ENTRYPOINT` during runtime, use flag `--entrypoint`:  

```sh
# will trigger command: sleep2.0 10
docker run --name sleeper image-name --entrypoint sleep2.0 10
```  

We can specify default argument using `CMD` and combine with `ENTRYPOINT`:  
```Dockerfile  
# When run the docker image => sleep 5
FROM image

ENTRYPOINT ["sleep"]

CMD ["5"]
```  

#### 1.2 Docker Security  
Docker uses Linux capabilities to limit the privilege of users in each Docker Containter.  

```sh
/usr/include/linux/capability.h
```  

Provide privilege, for example: MAC_ADMIN:  
```sh
docker run --cap-add MAC_ADMIN <image_name>
```  

Drop privilege:  
```sh
docker run --cap-drop KILL <image_name>
```  

Run container with all privileges:  
```sh
docker run --privileged <image_name>
```  

### 2. Command and Arguments in Kubernetes  
```yaml
apiVersion: v1
kind: Pod
metadata:
    name: sleeper
spec:
    containers:
    - name: sleeper-container
      image: ubuntu-sleeper
      # Corresponding to ENTRYPOINT in Docker
      command: ["sleep2.0"]
      # Passing arguments like when we run docker image.
      # corresponding to CMD but not overrides
      args: ["10"]    
```  

### 3. Environment variables  
```yaml
apiVersion: v1
kind: Pod
metadata:
    name: mypod
spec:
    containers:
    - name: myapp-contaniner
      image: myapp
      ports:
        - containerPort: 8080
      env:
        - name: APP_COLOR
          value: pink
        - name: PORT
          value: 3000
```  

### 4. ConfigMaps  
When using environment variables in each pod-definition files, it is difficult to manage these varibles within the various file.  
We can manage those varibales seperately from pod-definition file using `ConfigMap`.  
`ConfigMap` pass varibales in the form of key-value pairs. When a pod is created, inject the configmap into the pod.  

#### 4.1 ConfigMap Commands  
- Create ConfigMap imperatively  
```sh
kubectl create configmap <config-name> \
--from-literal=<KEY_1>=<VALUE_1> \
--from-literal=<KEY_2>=<VALUE_2> \
...
```  

- Create ConfigMap from file  
```sh
kubectl create configmap <config-name> \
--from-file=<config_file_name.properties>
```  

- Create ConfigMap from YAML file  
```sh
kubectl create -f config-map.yaml
```  

- Get list of created ConfigMap
```sh
kubectl get configmaps
```  

- Get details of all ConfigMaps  
```sh
kubectl describe configmaps
```  

#### 4.2 ConfigMap YAML file  
```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_COLOR: blue
  APP_MODE: prod
```  

The in pod-definition file:  
```yml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
  labels:
    name: simple-web-app
spec:
  containers:
    - name: web-app-container
      image: webapp
      ports:
        - containerPort: 8080
      # Inject configmap here
      envFrom:
        - configMapRef:
            # Name of configmap
            name: app-config
```  

If we just want to inject single env from ConfigMap:  
```yml
....
env:
  - name: APP_COLOR
    valueFrom:
      configMapKeyRef:
        # Name of configmap
        name: app-config
        # Key defined in configmap
        key: APP_COLOR
```  

Inject whole configmap as file in volumes:  
```yml
...
volumes:
  - name: app-config-volume
    configMap:
      name: app-config
```  

### 5. Secrets  
Similar to ConfigMap except that they are stored in an encoded or hashed format.  

#### 5.1 Secret commands  
- Create Secret imperatively  
```sh
kubectl create secret generic <secret-name> \
--from-literal=<KEY_1>=<VALUE_1> \
--from-literal=<KEY_2>=<VALUE_2> \
...
```  

- Create ConfigMap from file  
```sh
kubectl create secret generic <secret-name> \
--from-file=<config_file_name.properties>
```  

- Get list of secrets  
```sh
kubectl get secrets
```  

- Get details of secrets  
```sh
kubectl describe secrets
```  

- View specific secret  
```sh
kubectl get secret <secret-name> -o yaml 
```  

#### 5.2 Secret YAML file  
Because data in Secret is encoded format so we first encode the data using command:  
```sh
# encode data
echo -n 'data' | base64

# decode data
echo -n 'data' | base64 --decode
```  

```yml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
data:
  DB_HOST: bXlzcWw= # mysql
  DB_USER: cm9vdA== # root
  DB_PASSWORD: cGFzc3dvcmQ= # password
```  

Inject secret into the pod-definition file:  
```yml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
  labels:
    name: simple-web-app
spec:
  containers:
    - name: web-app-container
      image: webapp
      ports:
        - containerPort: 8080
      # Inject secret here
      envFrom:
        - secretRef:
            # Name of secret
            name: app-secret
```  

If we just want to inject single env from Secret:  
```yml
....
env:
  - name: DB_HOST
    valueFrom:
      secretKeyRef:
        # Name of secret
        name: app-secret
        # Key defined in secret
        key: DB_HOST
```  

Inject whole configmap as file in volumes (Each attribute in the secret is created as a file with the value of the secret as its content).  
```yml
...
volumes:
  - name: app-secret-volume
    secret:
      name: app-secret
```  

### 6. Security context  
Apply security capabilities on POD or Container level
```yml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
  labels:
    name: simple-web-app
spec:
  # define security context at POD level
  securityContext:
    # ex: run as user = 1000
    runAsUser: 1000
  containers:
    - name: web-app-container
      image: webapp
      ports:
        - containerPort: 8080
      # Inject secret here
      envFrom:
        - secretRef:
            # Name of secret
            name: app-secret
      # define security context at Container level
      securityContext:
        runAsUser: 1000
        # add more capabilities
        # only works at Container level
        capabilities:
          add: ["MAC_ADMIN"]
```  

### 7. Service Account  
**External Services**  
To make external services interact with Kubernetes, we have to create an account for each service. For example: Prometheus pull metrics from k8s api, or Jenkins deploys applications to k8s,... When we create a service account, it also creates a token automatically. The external services have to use that token to authenticate to k8s API. It also creates a **Secret** object that stores the token and that **Secret** object are linked to the service account. The token is used as **Bearer** authorization.  

**Internal services (deployed in k8s cluster)**  
The whole process of exporting the service account token and confiuring the third party application to use it can be made simply by automatically mounting the service token secret as a volume inside the pod hosting the third party service.  

K8s auto mount the default service account it we have not explicitly defined any.  
#### 7.1 Service Account Commands

- Create a service account  
```sh
kubectl create service account <service-account-name>
```  

- List all service accounts  
```sh
kubectl get serviceaccount
```  

- Get details of service account. The `Tokens` field is the name of the `Secret` object that stores the actual token.  
```sh
kubectl describe serviceaccount <service-account-name>
```  

#### 7.2 Define service account in Pod  
```yml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: my-container
    image: myapp
  serviceAccount: <service-account-name>
  # use this to disable
  # auto mount service account
  # if we have not defined it
  automountServiceAccountToken: false
```  

**Note**: We cannot modify service account of a pod once we created a pod. Instead, we have to delete and re-create the pod. However, in case of `Deployment`, we can change the service account so the definition file will automatically trigger a new rollout for the deployment. The deployment will take care of deleteing and re-creating new pods with the right service account.  

### 8. Resource requirements  
#### 8.2 Define container's resource in YAML file  

```yml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: my-container
      image: myapp
      ports:
        - containerPort: 8080
      # define resources requirements
      resources:
        # minimum requirements
        requests:
          memory: 1Gi
          cpu: 1
        # maximum requirements
        limits:
          memory: 2Gi
          cpu: 2
```  

Remember, the limits and requests are set for each container within the pod.  
The container can exceeds its memory limitation but not exceeds its CPU limitation because in case of CPU, k8s **throttle** the CPU so that it does not go beyond the specified limit.  
In case a pod try to consume more memory than its limit, the pod will be terminated.  

When a pod is created the containers are assigned a default CPU request of .5 and memory of 256Mi". For the POD to pick up those defaults you must have first set those as default values for request and limit by creating a **LimitRange** in that namespace.  

```yml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
spec:
  limits:
  - default:
      memory: 512Mi
    defaultRequest:
      memory: 256Mi
    type: Container
```  

```yml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-limit-range
spec:
  limits:
  - default:
      cpu: 1
    defaultRequest:
      cpu: 0.5
    type: Container
```  

### 9. Taints and Tolerations  
This is used for restricting nodes from excepting certain nodes but not telling the pod to go to particular nodes.  
A taint is auto set on master node.  

#### 9.1 YAML definition file  
- Taint a pod using defintion file:  
```yml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: my-container
      image: myapp
  # All of these values in tolerations
  # must be encoded in dounle quotes
  tolerations:
    - key: "app"
      value: "blue"
      operator: "Equal"
      effect: "NoSchedule"
```  
#### 9.2 Commands  

- To taint a node:  
```sh
kubectl taint nodes <node-name> key=value:taint-effect
```  
The `taint-effect` defines what would happens to the pods if they do not tolerate the taint. There are 3 kinds of taint-effect:  
`NoSchedule`: the pod will not be scheduled on the node.  
`PreferNoSchedule`: system tries to avoid placing pod on the node but not guarantees.  
`NoExcecute`: Pod will not be executed on the node and existing pods on that node will be evicted if they do not tolerate.  

- To see a taint of a node:  
```sh
kubectl describe node <master> | grep Taint
```  

- To remove a taint of a node:  
```sh
kubectl taint nodes <node-name> key=value:taint-effect-
```  

### 10. Node selectors  

#### 10.1 Commands  
- Labels a node  
```sh
kubectl label nodes <node-name> <label-key>=<label-value>
```  

#### 10.2 Define node for pod in YAML file  
```yml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: data-processor
    image: data-processor
  nodeSelector:
    # Labels of a node
    # for example: key as size and value as Large
    size: Large
```  
Note that we cannot provide advanced expressions like OR or NOT. for ex: Large OR Medium...  
### 11. Node affinity  
The purpose is to ensure that pods are hosted on particular nodes.  

There are two states in the life cycle of a node when considering node affinity: `during scheduling` and `during execution`. `During scheduling` is the state where a pod does not exist and is created for the first time. `During execution` is the state where a pod has been running and a change is made in the environment that affects node affinity, such as a change in the label of a node.  

More details: [assign pods to nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)  

#### 11.1 Commands

#### 11.2 YAML definition file  
```yml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: data-processor
    image: data-processor
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: size
            operator: In
            values:
            - Large
            - Medium
            # operator: Exists -> check if label size exist on the nodes
```  
















