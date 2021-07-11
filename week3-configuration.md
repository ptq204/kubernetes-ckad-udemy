## Kubernetes configuration  

### 1. Recap Docker: Commands & Arguments  

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

