## Pod Design

### 1. Labels & Selector  

### 1.1 Commands  

- Select pod with a label:  
```sh
kubectl get pods --selector <label_key>=<label_value>
```  

### 1.2 Pod definition file  

```yaml
apiVersion: apps/v1
kind: Replicaset
metadata:
    name: simple-webapp
    # labels of the replicaset itself
    labels:
        app: App1
        function: front-end
spec:
    replicas: 3
    selector:
        # use this to connect replicaset with a pod
        matchLabels:
            app: App1
    template:
        metadata:
            # labels configured on the pod
            labels:
                app: App1
                function: front-end
        spec:
            containers:
            - name: simple-webapp
              image: simple-webapp
```  

### 2. Annotations  
Annotations are used to record other details for informatory purpose. For example: build version, phone numbers, email...  which used for some kind of integration purposes.  

```yaml
apiVersion: apps/v1
kind: Replicaset
metadata:
    name: simple-webapp
    # labels of the replicaset itself
    labels:
        app: App1
        function: front-end
    annotations:
        buildVersion: 1.1.0
spec:
    replicas: 3
    selector:
        # use this to connect replicaset with a pod
        matchLabels:
            app: App1
    template:
        metadata:
            # labels configured on the pod
            labels:
                app: App1
                function: front-end
        spec:
            containers:
            - name: simple-webapp
              image: simple-webapp
```  

### 3.  Rolling updates & Rollbacks in Deployments  
When we first create a deployment, it triggers a rollout, a new rollout creates a new deployment. In the future, when the application is upgraded, meaning when the container version is updated to a new one, a new rollout is triggered and a new deployment revision is created. This helps us keep track of the changes made to our deployment and enables us to roll back to a previous version of deployment if necessary.  

The default deployment strategy is `Rolling update` which means it will down->up one by one.  

Create deployment -> create replicaset automatically. When we upgrade the applications, k8s create a new replicaset and start deploying the containers there. At the same time, taking down the pods in the old replicaset following rolling update strategy.  

Rollback means k8s destroys all the pods in the new replicaset then bring the older ones up in the old replicaset.  

#### 3.1 Commands  
- To see the status of the rollout:  
```sh
kubectl rollout status deployment/<deployment-name>
```  

- To see revisions and history of the our deployment:  
```sh
kubectl rollout history deployment <deployment-name>
```  

- To check status of specific revision, use **--revision** flag:  
```sh
kubectl rollout history deployment <deployment_name> --revision=<revision_name>
```  

- To record a change, use **--record** flag:  
```sh
kubectl edit deployments nginx --record
```  

- To rollback:  
```sh
kubectl rollout undo deployment/<deployment_name>
```  

#### 3.2 Rollout strategy

##### 3.2.1 Recreate  
A deployment defined with a strategy of type `Recreate` will terminate all the running instances then recreate them with the newer version.

```yaml
spec:
    replicas: 3
    strategy:
        type: Recreate
```  

##### 3.2.2 Rolling update  
a secondary ReplicaSet is created with the new version of the application, then the number of replicas of the old version is decreased and the new version is increased until the correct number of replicas is reached.  

```yaml
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
        maxSurge: 2        # how many pods we can add at a time
        maxUnavailable: 0  # maxUnavailable define how many pods can be unavailable # during the rolling update
```  

### 4. Jobs  
The default restart policy set on the pod is `Always`. That means the pod always recretaes the container when it exists.  

```yaml
apiVersion: v1
kind: Pod
metadata:
    name: math-pod
spec:
    containers:
    - name: math-add
      image: ubuntu
      command: ['expr', '3', '+', '5']
    restartPolicy: Always # Failure | Never
```  

### 4.1 YAML definition file  
```yaml
apiVersion: batch/v1
kind: Job
metatdata:
    name: math-add-job
spec:
    completions: 3 # set 3 pods to run in this job. This job will create a new pods until there are 3 successful pods.
    # Instead of creating pods sequentially, we can create them in parallel.
    parallelism: 3 # create 3 pods in parallel
    template:
        # just move the spec in pod-definition file
        spec:
            containers:
            - name: math-add
              image: ubuntu
              commnad: ['expr', '3', '+', '5']
            restartPolicy: Never
```  

The job created above will be executed immediately. To create a CronJob that runs periodically, use following definition file:  
```yaml
apiVersion: batch/v1beta1
kind: CronJob
metatdata:
    name: reporting-cron-job
spec:
    schedule: "/1 * * * *"
    jobTemplate:
        spec:
            completions: 3 # set 3 pods to run in this job. This job will create a new pods until there are 3 successful pods.
            # Instead of creating pods sequentially, we can create them in parallel.
            parallelism: 3 # create 3 pods in parallel
            template:
                # just move the spec in pod-definition file
                spec:
                    containers:
                    - name: reporting-tool
                      image: reporting
                    restartPolicy: Never
```

### 4.2 Commands  

- See list of created job:  
```sh
kubectl get jobs
```  

- Delete a job:  
```sh
kubectl delete job <job-name>
```  

- See list of cronjobs:  
```sh
kubectl get cronjob
```  




