## Services and Networking  

### 1. Services  
There are some kinds of services:  
- **NodePort**: the services make an internal pod accessible on a port on the node. This maps a port on the node to a port on the pod.  
- **ClusterIP**: the service create a virtual IP inside the cluster to enable communication between different services such as a set of front-end servers to a set of backend servers. This group a multiple pods together and provide a single interface to access pods in the group. The requests are fowarded to one of the pods randomly. This have advantages that it helps scale microservices easier: the IP of a pod is dynamic when pod is recreated or moved but it not impacts the communication between services because of single provided ClusterIP.  
- **LoadBalancer**: supports cloud provider, distribute load across to different web servers.  

#### 1.1 YAML definition file   

##### 1.1.1 NodePort  
```yaml
apiVersion: v1
kind: Service
metadata:
    name: myapp-service
spec:
    type: NodePort # kind of service, it could be ClusterIP, LoadBalancer
    ports:
    - targetPort: 80 # port of a pod. If not provided => the same with `port`
      port: 80       # port of a service
      nodePort: 3008 # port of a node. If not provided => auto allocate between 30000 - 32767
    # because there are many pods running on a node
    # we have to link a service to a specific pod by pod's labels
    selector:
        app: myapp
```  

**Note**:
- If there are many similar pods in the same node (even same labels - for scale purpose). The service will auto selects all that kinds of pods as endpoints to foward external requests
- In cases pods are distributed across multiple nodes. When we create a service without adding additional configs, K8s auto create a service that spans across all the nodes n the cluster and maps the target port to the same node port in the cluster. This way, you can access your application using the IP of any node in the cluster and using the same port number of the node.  

##### 1.1.2 ClusterIP  
```yaml
apiVersion: v1
kind: Service
metadata:
    name: back-end
spec:
    type: ClusterIP
    ports:
    - targetPort: 80
      port: 80
    selector:
        app: myapp
```  

#### 1.2 Commands  

- Get list of services  
```sh
kubectl get services
```  

### 2. Ingres Networking  
Ingress networking helps users to access your application using a single accessible URL that you can configure to route different services within your cluster. At the same time, it implements SSL as well.  
We still have to expose Ingress network as a NodePort or with a cloud native load balancer.  

#### 2.1 Ingress Controller  
We dont have Ingres Controller by default so we must deploy it. GCE and Nginx are supported and maintained by K8s.  

Firts, create a `Deployment`, here for Nginx:  

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-ingress-controller
spec:
  replicase: 1
  selector:
    matchLabels:
      name: nginx-ingress
  template:
    metadata:
      labels:
        name: nginx-ingress
    spec:
      containers:
      - name: nginx-ingress-controller
        image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.21.0
      # special build of Nginx to be used as ingress controller in k8s
      # within the image, the Nginx program is stored at localtion in nginx-ingress-controller
      args:
      # the command to start the nginx-controller service
      - /nginx-ingress-controller
      # path to the ConfigMap
      - --configmap=$(POD_NAMESPACE)/nginx-configuration
      env:
      - name: POD_NAME
        valueFrom:
          fieldRef:
            fieldPath: metadat.name
      - name: POD_NAMESPACE
        valueFrom:
          fieldRef:
            fieldPath: metadata.namespace
      ports:
      - name: http
        containerPort: 80
      - name: https
        containerPort: 443
```  

Create a `ConfigMap` to store the configs of Nginx:  

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: ngnix-configuration
```  

We the create a `Service` to expose the ingress-controller.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    name: http
  - port: 443
    targetPort: 443
    protocol: TCP
    name: https
  # Link to the POD in the Nginx Deployment
  selector:
    name: nginx-ingress
```  

The Ingress-controller in K8s has additional build-in to monitor the K8s cluser for ingress-resources and configure the underlying Nginx server when something is changed. To do this, it requires `ServiceAccount` with a set of permissions. Lets create one:  

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
```  

#### 2.2 Ingress resources  

Ingress resource is a set of rules and configurations applied on the ingress-controller. For ex: foward request, route traffics...  

Suppose we have endpoints `wear` and `watch` in our APIs. Lets create ingress-resource for `wear` endpoint:  

```yaml
# ingress-wear
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wear
spec:
  # backend defines where the traffic will be routed
  # the traffic does not go directly to the Pod, it has to go through the Service
  # if it is single backend, we dont have any rules
  backend:
    service:
      name: wear-service
      port:
        number: 80
```  

We may have rules at the top for each host or domain name. Within each rule, you have different paths to route traffic. For example:  
- wwww.myonline-store.com
  - /wear
  - /watch
  - /
- wwww.wear.myonline-store.com
  - /
  - /returns
  - /support  

For single rule (single host) and traffic by paths:  
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wear-watch
  # annotations:
  #   nginx.ingress.kubernetes.io/rewrite-target: /
  #   nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - https:
      paths:
      - path: /wear
        pathType: Prefix
        backend:
          service:
            name: wear-service
            port:
              number: 80
      - path: /watch
        pathType: Prefix
        backend:
          service:
            name: watch-service
            port:
              number: 80
```  

For multiple hosts and traffic by domain name:  
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wear-watch
spec:
  rules:
  - host: wear.myonline-store.com
    http:
      paths:
      - backend:
          service:
            name: wear-service
            port:
              number: 80
  - host: watch.myonline-store.com
    http:
      paths:
      - backend:
          service:
            name: watch-service
            port: 
              number: 80
```

#### 2.3 Network policy  
Example: The incoming requests from is `Ingress` traffic and the outgoing requests is `Egress` traffic.  
In cases we want to restrict the requests from services to services, we should create `NetworkPolicy` for pods and define rules in it.  

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: internal-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      name: internal
  ingress:
  - from:
    - podSelector:
        matchLabels:
          name: app
      # allow pod from specific namespace:
      namespaceSelector:
        matchLabels:
          name: prod
      # allow pod with ip range
    - ipBlock:
        cidr: 192.168.5.10/32
    ports:
    - port: 3000
  egress:
  - to:
    - podSelector:
        matchLabels:
          name: payroll
    ports:
    - port: 8080
  - to:
    - podSelector:
        matchLabels:
          name: mysql
    ports:
    - port: 3306
      protocol: TCP
```  

#### 2.4 Command  
- Get list of ingress:  
```yaml
kubectl get ingress
```  

- Get details of Ingress resource:  
```yaml
kubectl describe ingress <ingress-resource-name>
```  

- Create ingress-resource imperatively:  
```yaml
kubectl create ingress <ingress-name> --rule="<host-name>/<path>=<service-name>:<port>"
```  

- Get ingress in all namespace:  
```yaml
kubectl get ingress --all-namespaces
```  

- Edit ingress on specific namespace:  
```yaml
kubectl edit ingress --namespace <namespace>
```  




