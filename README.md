# microservises app: bluepost

https://gitlab.com/bluepost
```
$ git clone https://github.com/adavarski/GitLab-group-microservices-minikube-deploy
$ cd GitLab-group-microservices-minikube-deploy
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
$ mv docker-compose.yml docker-compose.yml.dockerhub
$ cp docker-compose.yml.ORIG docker-compose.yml
$ docker-compose up --build
```
Browser: http://localhost:9292

$ docker rm $(docker ps -a -q)

## Deploy to minikube using kubectl:

```
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

If you want you can setup images based on GitLab build:
Example: davarski/post:master_43e751df70a228f3f6ceee088b1db975aa06b275

DockerHub Tags for image davarski/post:
latest
2.0.0
master_43e751df70a228f3f6ceee088b1db975aa06b275
master_0439d6744484ad8cd0862e69a5022c54ec735564
master_14b278978c4e5122678de0dd5efb791eac100d40

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

Clean:
$ kubectl delete --all svc --namespace=default
$ kubectl delete --all deployments --namespace=default
```

## Deploy to minikube using Helm:

```
$ helm init
$ helm --kube-context=minikube install --set usePassword=false --name mongodb  stable/mongodb
$ helm upgrade post post/charts/post --install --kube-context=minikube --set image.tag=latest
$ helm upgrade ui ui/charts/ui --install --kube-context=minikube --set image.tag=latest

$ kubectl get pod
NAME                       READY   STATUS    RESTARTS   AGE
mongodb-69fc4cddff-zqdkk   1/1     Running   0          11m
post-65c8d5c598-gjlpb      1/1     Running   0          10m
ui-7d95c8ff49-dsjvr        1/1     Running   0          9m
ui-7d95c8ff49-hg7kb        1/1     Running   0          9m

$ kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP          1h
mongodb      ClusterIP   10.101.221.251   <none>        27017/TCP        3m
post         ClusterIP   10.104.186.193   <none>        5000/TCP         1m
ui           NodePort    10.106.85.96     <none>        9292:32299/TCP   1m

$ minikube ip
192.168.99.100

Browser: http://192.168.99.100:32299/

Clean:

helm del --purge post
helm del --purge ui
helm del --purge mongodb
```

### Deploy with Jenkins and Helm

```
Setup jenkins https://github.com/adavarski/K8S-with-jenkins-and-helm

jenkins-volume.yml 
-------------------
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkins-volume
  namespace: default
spec:
  storageClassName: jenkins-volume
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 10Gi
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /data/jenkins/
    
 $ kubectl create -f jenkins-volume.yml 
 
 values.yml 
 ---------------------
Master:
  ServicePort: 8080
  ServiceType: NodePort
  NodePort: 32123
  ScriptApproval:
    - "method groovy.json.JsonSlurperClassic parseText java.lang.String"
    - "new groovy.json.JsonSlurperClassic"
    - "staticMethod org.codehaus.groovy.runtime.DefaultGroovyMethods leftShift java.util.Map java.util.Map"
    - "staticMethod org.codehaus.groovy.runtime.DefaultGroovyMethods split java.lang.String"
  InstallPlugins:
    - kubernetes:1.6.4
    - workflow-aggregator:2.5
    - workflow-job:2.21
    - credentials-binding:1.16
    - git:3.9.1
Agent:
  volumes:
    - type: HostPath
      hostPath: /var/run/docker.sock
      mountPath: /var/run/docker.sock

Persistence:
  Enabled: true
  StorageClass: jenkins-volume
  Size: 10Gi

NetworkPolicy:
  Enabled: false
  ApiVersion: extensions/v1beta1

rbac:
  install: true
  
 $  helm install
  --name jenkins
  --namespace default
  --values values.yml
  stable/jenkins

Get admin user password:
printf $(kubectl get secret --namespace jenkins jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo

 $ kubectl get svc|grep NodePort
jenkins         NodePort    10.100.88.246   <none>        8080:32123/TCP   45m

$ minikube ip
192.168.99.100

Login to jenkins http://192.168.99.100:32123 user admin with password from printf output

Setup k8s RBAC for jenkins:

$ kubectl create clusterrolebinding kube-system-default-admin --clusterrole cluster-admin --serviceaccount=kube-system:default
$ kubectl create clusterrolebinding default-sa-admin --user system:serviceaccount:default:default  --clusterrole cluster-admin

Install stable/mongodb helm chart (we will not build chart for mongodb)

$ helm  install --set usePassword=false --name mongodb  stable/mongodb

Deploy blueprint app

Create a multibranch pipline with a fork of this repository as the Git source for the pipeline using provided Jenkinsfile.


````
### Deploy with GitLab and Helm : TODO

```
Install GitLab

Import project from github:

Setup deploy pipeline: add .gitlab-ci.yml to root of repo:

# deploy to staging environment
deploy_staging:
  stage: deploy
  image: ibmcom/k8s-helm:v2.6.0
  before_script:
    - export KUBECONFIG=~/.kube/config ???? -kube-context=minikube
    - helm init --client-only
  script:
    - helm upgrade  upgrade post post/charts/post --install --set image.tag=latest
    - helm upgrade  upgrade ui ui/charts/ui --install  --set image.tag=latest

  only:
  - master
```


  
  
  
  
