### Certificates  
Inspect these files:  
- Kube-API server: `/etc/kubernetes/manifests/kube-apiserver.yaml`  
- ETCD server: `/etc/kubernetes/manifests/etcd.yaml`  

### 2. RBAC  
#### 2.1 Definition file
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["list", "get", "create", "delete", "update"]  
    resourceNames: ["blue", "orange"] ## Restrict for specific resources
```  

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
subjects:
  - kind: User
    name: dev-user
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```  

#### Commands  

- Check if has sufficent permission:  
```sh
kubectl auth can-i create deployments
```  

- Check permission for specific user:  
```sh
kubectl auth can-i delete pods --as <user name>
```  

### 3. Admission controller  

`kube-apiserver` file located at `/etc/kubernetes/manifests/kube-apiserver.yaml`  

#### 3.1 Commands  
- View enabled Admission controllers  
```sh
ps -ef | grep kube-apiserver | grep enable-admission-plugins
```  

- View admission controllers in `kubeadm`:  
```sh
kubectl exec kube-apiserver-controlplane -n kube-system | grep enable-admission-plugins
```  

- Turn on admission controller  
```sh
kube-apiserver --enable-admission-plugins=NamespaceLifecycle,LimitRanger ...
```  

- Turn off admission controller  
```sh
kube-apiserver --disable-admission-plugins=PodNodeSelector,AlwaysDeny ...
```  

### 4. Cluster roles  

Give access to cluster means give access to all resources in that cluster  

#### 4.1 Definition files  
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-admin
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["list", "get", "create", "delete", "update"]  
```  

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-binding
subjects:
  - kind: User
    name: cluster-admin
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```  

#### 4.1 Commands  
- See list of namespaced resources:  
```sh
kubectl api-resources --namespaced=true
```  

### 5. API version/deprecation  

To enable version for apigroup, edit flag `--runtime-config` flag in `kube-apiserver.yaml` file  

#### 5.1 Commands  
- Convert to new version  
```sh
kubectl convert -f nginx.yaml --output-version <version>
```  

### 6. Helm  
#### 6.2 Commands  
- Seach package in repo  
```sh
helm search repo <package_name>
```  

- Search package in Artifact hub:  
```sh
helm search hub <package_name>
```  

- Download package  
```sh
helm pull --untar bitnami/apache
```  

- List helm repos  
```sh
helm repo list
```  
- List package  
```sh
helm list
```



