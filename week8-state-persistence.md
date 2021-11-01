## State persistence

### 1. Volumes  
Volumes make data permanent even after the pod is deleted.  

`Persistent Volume (PV)` is a cluster wide pool of storage volumes configured by an administrator. It helps manage the volumes created by each POD in a central way. Users can now select storage from this pool using `Persistent Volume Claim (PVC)`.  

If there are no volumes available the Persistent Volume Claim will remain in a pending state until newer volumes are made available to the cluster. 

#### 1.1 YAML definition file  

- Define volume mount in POD:  
```yaml
apiVersion: v1
kind: Pod
metadata:
	name: myapp
spec:
	containers:
	- name: alpine
	  image: alpine
		command: ["/bin/sh", "-c"]
		args: ["shuf -i 0-100 -n 1 >> /opt/number.out;"]
		# define volume mount field
		# to access the volume in the node
		volumeMounts:
		# mount path in the container
		- mountPath: /opt
			name: data-volume
	volumes:
	- name: data-volume
		hostPath:
			# directory in a node
			path: /data
			type: Directory
```  

- Define Persisten Volume (created by administrator):  
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
	name: pv-voll
spec:
	persistentVolumeReclaimPolicy: Retain
	accessModes:
	- ReadWriteOnce
	# specify the amount of storage to be reserved
	capacity:
		storage: 1Gi
	# define volume type
	# local directory
	# not to be used in prod env
	hostPath:
		path: /tmp/data
	#awsElasticBlockStore:
		#volumeID: <volume-id>
		#fsType: ext4
```  

- Define Persistent Volume Claim (request by user):  
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
	name: myClaim
spec:
	accessModes:
	# have to match with access_mode of Persistent Volume
	- ReadWriteOnce
	resources:
		requests:
			# should be bound by capacity of Persisten Volume
			storage: 500Mi
```  

Or we can use PVC directly in a POD:  
```yaml
...
spec:
	containers:
	...
	volumes:
	- name: myVol
		persistentVolumeClaim:
			claimName: myClaim
```  

The `persistentVolumeReclaimPolicy` defines how volume is treated when the claim is deleted:  
- **Retain (default)**: the persistent volume of the claim will remain until it is deleted manually.  
- **Delete**: will be deleted automatically  
- **Recycle**: the data volume will be scrubbed before making it available to other claims.  

When delete PVC, it still remains in `Terminating` state because the volume is still used by the POD.  

#### 1.2 Commands  

Get list of persistent volume:  
```sh
kubectl get persistenvolume
```  

Get persisten volume claim:  
```sh
kubectl get pvc
```

Delete persisten volume claim:  
```sh
kubectl delete persistentvolumeclaim
```  

### 2. Storage Classes  
Suppose we mount volume data to a disk on GCE. Each time we create a PVC, we have to manually create a disk on GCE. That is static provisioning volumes.  

If the volumes gets provisioned automatically when the application requires it, it is called dynamic provisioning volumes.  

With `StorageClass`, we can define a provisioners such as Google storage that can be automatically provisioned and attach that when a claim is required.  

#### 2.1 YAML definition file  
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
	name: google-storage
provisioner: kubernetes.io/gce-pd
parameters:
	# type of disk
	type: pd-standard [ pd-standard | pd-ssd ]
	replication-type: none [ node | regional-pd ]
```  

then in PVC definition file:  
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
	name: myClaim
spec:
	accessModes:
	# have to match with access_mode of Persistent Volume
	- ReadWriteOnce
	# define storage class if it is used:
	storageClassName: google-storage
	resources:
		requests:
			# should be bound by capacity of Persisten Volume
			storage: 500Mi
```  

#### 2.2 Commands  
- Get list of storage classes  
```sh
kubectl get sc
```  

### 3. Stateful Sets  
`StatefulSets` is used for managing the deployment and scaling of a set of Pods, and provides guarantees about the ordering and uniqueness of these Pods.  

The pods in StatefulSets are created from the same spec, but are not interchangeable: each has a persistent identifier that it maintains across any rescheduling.  

Each Pod in a StatefulSet derives its hostname from the name of the StatefulSet and the ordinal of the Pod. The pattern for the constructed hostname is `$(statefulset name)-$(ordinal)`.  

#### 3.1 YAML file  
```yaml
apiVersion: apps/v1
kind: StatefulSets
metadata:
	name: mysql
	labels:
		app: mysql
spec:
	template:
		metadata:
			labels:
				app: mysql
		spec:
			containers:
			- name: mysql
				image: mysql
	replicas: 3
	selector:
		matchLabels:
			app: mysql
	# name of headless service
	serviceName: mysql-h
	# if you want the StatefulSets to not follow `order` approach
	podManagementPolicy: Parallel
```  

The above defintion will create 3 PODS named: mysql-1, mysql-2, mysql-3  

#### 3.2 Commands  

- Create from YAML file:  
```yaml
kubectl create -f statefulset-definition.yaml
```  

- Scale:  
```yaml
kubectl scale statefulset <statefulset-name> --replicase=<number_to_scale>
```  

- Delete:  
```yaml
kubectl delete statefulset mysql
```  

### 4. Headless service  

A headless service is created as a normal service but does not have an IP like a cluster IP. It does not perform load balancing either.  

Headless service creates DNS entries for each pod using the pod's name and a subdomain: `<podname>.<headless-servicename>.<namespace>.svc.<cluster-domain>.example`.  

#### 4.1 YAML file  
**Service**:  
```yaml
apiVersion: v1
kind: Service
metadata:
	name: mysql-h
spec:
	ports:
	- port: 3306
	selector:
		app: mysql
	# This make headless service
	clusterIP: None
```  

**Pod**  
```yaml
apiVersion: v1
kind: Pod
metadata:
	name: myapp-pod
	labels:
		app: mysql
spec:
	containers:
	- name: mysql
		image: mysql
	# Define subdomain whose name is the same with
	# the name of the headless service
	subdomain: mysql-h
	hostname: mysql-pod
```  

=> Example above creates a pod with DNS name: `mysql-pod.mysql-h.default.svc.cluster.local`. Howerver, if we use `Deployment` to deploy multiple pods, they will have the same DNS name. To solve that, we can use `StatefulSet` instead. Notice when using `StatefulSet`, we do not need to specify a subdomain or hostname. `StatefulSet` automatically assign the right hostname for each pod based on the pod's name, and automatically assign the subdomain based on the name of the headless service:  

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
	name: mysql-deployment
	labels:
		app: mysql
spec:
	# Specify name of headless service
	serviceName: mysql-h
	replicas: 3
	matchLabels:
		app: mysql
	template:
		metadata:
			name: myapp-pod
			labels:
				app: mysql
		spec:
			containers:
			- name: mysql
				image: mysql
```  











