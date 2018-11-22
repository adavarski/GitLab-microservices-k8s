# microservises app: bluepost

https://gitlab.com/bluepost
```
$ git clone https://github.com/adavarski/gitlab-group-microservices
$ cd gitlab-group-microservices
$ rm -rf .git
```

This is an example of a microservices application with **Gitlab CI** pipelenes (build, test, release)

In Gitlab UI, create a new group for bluepost application

Gitlab CI groups allow to group related projects. In our example, bluepost group will contain the microservices that the bluepost application consists of.

Create a new project in Gitlab CI web UI for each component of bluepost application. Each component of the bluepost application is contained in its own repository and has its own CI/CD pipeline defined in a .gitlab-ci-yml file stored in the root of each of the component’s directory.

Gitlab CI has its own container registry, but in this example we use Docker Hub,  you’ll need to make sure the CONTAINER_IMAGE variable is set according to your Docker Hub account name and repository for each image has been created. For each project define variables: Settings -> CI/CD. Define CI_REGISTRY_USER and CI_REGISTRY_PASSWORD variables to allow logging to Docker Hub. After define projects and setup CI/CD pipeline env variables execute:

Example: 
```
$ cd ui
$ git init
$ git add .
$ git commit -m "Init commit"
$ git remote add origin https://gitlab.com/bluepost/ui.git
$ git push origin master

$ cd ../post
$ git init
$ git add .
$ git commit -m "Init commit"
$ git remote add origin https://gitlab.com/bluepost/post.git
$ git push origin master
```

GitLab push to Dockerhub: davarski/ui and davarski/post images with tags latest, etc.

## Test microservices locally:
```
$ docker-compose up --build
```
Browser: http://localhost:9292

$ docker rm $(docker ps -a -q)

## Deploy to minikube:

```
$ minikube start
$ mkdir tmp
$ cp docker-compose.yml tmp

Setup dockerhub images and setup docker-compose version: "2":

$ cat docker-compose.yml
version: '2'

services:
  mongo:
    image: mongo:3.4

  post:
    image: davarski/post
    environment:
      - POST_DATABASE_HOST=mongo
    depends_on:
      - mongo

  ui:
    image: davarski/ui
    environment:
      - POST_SERVICE_HOST=post
      - POST_SERVICE_PORT=5000
    ports:
      - 9292:9292
    depends_on:
      - post

Generete k8s kubectl files:

$ rm docker-compose.yml
$ kompose convert
WARN Unsupported depends_on key - ignoring        
INFO Kubernetes file "mongo-service.yaml" created 
INFO Kubernetes file "post-service.yaml" created  
INFO Kubernetes file "ui-service.yaml" created    
INFO Kubernetes file "mongo-deployment.yaml" created 
INFO Kubernetes file "post-deployment.yaml" created 
INFO Kubernetes file "ui-deployment.yaml" created 

Setup nodeport for UI

Add NodePort and nodePort

$ cat ui-service.yaml 
apiVersion: v1
kind: Service
metadata:
  annotations:
    kompose.cmd: kompose convert
    kompose.version: 1.1.0 (36652f6)
  creationTimestamp: null
  labels:
    io.kompose.service: ui
  name: ui
spec:
  type: NodePort
  ports:
  - name: "9292"
    port: 9292
    targetPort: 9292
    nodePort: 30623
  selector:
    io.kompose.service: ui
status:
  loadBalancer: {}
  
  kubectl apply :
  
$ export KUBECONFIG=~/.kube/config
$ kubectl create -f .
deployment.extensions/mongo created
service/mongo created
deployment.extensions/post created
service/post created
deployment.extensions/ui created
service/ui created

$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
mongo-6bc479dbf6-2867b   1/1     Running   0          19m
post-659cbd6d8b-rdqxh    1/1     Running   0          19m
ui-7774c64655-fqp5c      1/1     Running   0          19m

$ minikube ip
192.168.99.100

Browser: http://192.168.99.100:30623
```


  
  
  
