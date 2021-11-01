## Observability

### 1. Concepts  

**Pod's state transition**  

`Pending` -> `ContainerCreating` -> `Running`.  

When a pod is first created, is is in **Pending** state. This is where scheduler tries to figure out where to place a pod.  

Once all a containers in a pod, it switchs to **Running** state.  

**Pod's condition**  

- PodScheduled
- Initialized
- ContainersReady
- Ready  

Sometimes, the pod's **Ready** condition is true but the application inside that pod is not actually running (some applications, services take some times to start). So we have a term `readinessProbe`. Add this field in pod's definition file will tell the k8s not to set `Ready` condition be true immediately. Instead, it will perform a test to see if the API of the service running in a pod response positively.  

### 2. Readiness probe  

```yaml
apiVersion: v1
kind: Pod
metadata:
    name: webapp
    labels:
        kind: backend
spec:
    containers:
      - name: webapp
        image: simple-webapp
        ports:
        - containerPort: 8080
        # Define readinessProbe
        readinessProbe:
            # for http
            httpGet:
                # health check api
                path: /api/ready
                port: 8080
            # if we know the application will take 10s to warm up
            # define delay time before checking status
            initialDelaySeconds: 10
            # specify how often to probe:
            periodSeconds: 5
            # by default it takes 3 attemps to probe the application
            # modify this:
            failureThreshold: 8

            # for tcp (like database connection...)
            tcpSocket:
                port: 3306
            # for executing a command
            exec:
                command:
                - cat
                - /app/is_ready
```  

### 3. Liveness probe  

Everytime an container in a pod exited, k8s will make an attempt to restart the container to restart service to user. However, we have a case that the application is not really working but the contaienr continues to stay alive. K8s will assume the container is up but it actually cannot serve users. This is where `livenessProbe` works. A liveness probe can be configured on a container to periodically test whether the application within the container is actually healthy.  

```yaml
apiVersion: v1
kind: Pod
metadata:
    name: webapp
    labels:
        kind: backend
spec:
    containers:
    - name: webapp
      image: simple-web-app
      ports:
        containerPort: 8080
      # define liveness probe
      livenessProbe:
        # for http
        httpGet:
            path: /api/health
            port: 8080
        initialDelaySeconds: 10
        periodSeconds: 8
        failureThreshold: 8
        # for tcp:
        tcpSocket:
            port: 3306
        # for executing command:
        exec:
            command:
            - cat
            - /app/is_ready
```  

### 4. Logging  

- View the logs. The `-f` option is used to stream the log lines.  
```sh
kubectl logs -f <pod_name>
```  

- In cases when there are multiple containers in a pod, we have to specify the container name:  
```sh
kubectl logs -f <pod_name> <container_name>
```  

### 5. Monitoring  
K8s runs an agent on each node known as the kubelet which is responsible for receiving instructions from the k8s API master server and running PODs on the nodes. `cAdvisor` is responsible for retrieving metrics from pods and exposing them through the kubelet API to meet the metrics available for the metrics server.  

#### 5.1 In memory metrics server: Metric-server  
- For minikube:  
```sh
minikube addons enable metrics-server
```  

- For others:  
```sh
git clone https://github.com/kubernetes-sigs/metrics-server
```  

```sh
kubectl create -f deploy/<version>/
```  

To view performance metrics of node:  
```sh
kubectl top node
```  

To view performance metrics of pod:  
```sh
kubectl top pod
```  



