# microservises app: bluepost

We will build, test and release with GitLab:

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

 $ cd ./jenkins    
 $ kubectl create -f jenkins-volume.yml 
 $ helm install --name jenkins --values values.yml stable/jenkins

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

$ kubectl get pod
NAME                       READY   STATUS    RESTARTS   AGE
jenkins-9f8486cb5-kfb6s    1/1     Running   0          2h
mongodb-69fc4cddff-b2hqz   1/1     Running   1          59m
post-65c8d5c598-shhhm      1/1     Running   6          31m
ui-7d95c8ff49-4wwl2        1/1     Running   6          29m
ui-7d95c8ff49-fr74x        1/1     Running   4          29m

$ kubectl get svc|grep ui
ui              NodePort    10.103.25.143   <none>        9292:30124/TCP   9m

Browser: http://192.168.99.100:30124

Clean:

helm del --purge post
helm del --purge ui
helm del --purge mongodb


````
### Deploy with GitLab and Helm : TODO

```
Install GitLab or use gitlab chart to install 

$ helm init # initialize Helm
$ helm repo add gitlab https://charts.gitlab.io
$ helm install --name gitlab \
--namespace default \
--set baseIP=192.168.100.99 \
--set baseDomain=ci.devops.fun \
gitlab/gitlab-omnibus

Import project from github:

Setup deploy pipeline: add .gitlab-ci.yml to root of repo:

stages:
  - deploy

# deploy to staging environment (minikube)
deploy_staging:
  stage: deploy
  image: ibmcom/k8s-helm:v2.6.0
  before_script:
    - export KUBECONFIG=/etc/kubernetes/admin.conf 
    - helm init --client-only
  script:
    - helm upgrade  upgrade post ./post/charts/post --install --set image.tag=latest
    - helm upgrade  upgrade ui ./ui/charts/ui --install  --set image.tag=latest

 environment:
    name: staging
  only:
  - master
```


 ### Deploy with gitlab-runner
 
 ```
 
 sudo wget -O /usr/local/bin/gitlab-runner https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64
 sudo chmod +x /usr/local/bin/gitlab-runner


davar@home ~/LABS/GitLab-group-microservices-minikube-deploy/ui $ cat .gitlab-ci.yml
stages:
  - deploy

# deploy to staging environment
deploy_staging:
  stage: deploy
  image: davarski/k8s-helm:latest
  script:
    - export KUBECONFIG=/home/davar/.kube/config
    - helm init --client-only
    - helm  upgrade ui /ui/charts/ui --install  --set image.tag=latest 
  environment:
    name: staging
  only:
  - master
davar@home ~/LABS/GitLab-group-microservices-minikube-deploy/ui $ sudo gitlab-ci-multi-runner exec docker deploy_staging --docker-volumes /home/davar/.kube:/home/davar/.kube --docker-volumes /home/davar/.minikube:/home/davar/.minikube --docker-volumes ${PWD}:/ui
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
davar@home ~/LABS/GitLab-group-microservices-minikube-deploy/ui $ gitlab-runner exec docker deploy_staging --docker-volumes /home/davar/.kube:/home/davar/.kube --docker-volumes /home/davar/.minikube:/home/davar/.minikube --docker-volumes ${PWD}:/ui
Runtime platform                                    arch=amd64 os=linux pid=13116 revision=3afdaba6 version=11.5.0
WARNING: You most probably have uncommitted changes. 
WARNING: These changes will not be tested.         
Running with gitlab-runner 11.5.0 (3afdaba6)
Using Docker executor with image davarski/k8s-helm:latest ...
Pulling docker image davarski/k8s-helm:latest ...
Using docker image sha256:46f1b520ac9eb76235cd8f6bedc56dfcb8fc1a3017265abd16c20bd3b47f1ef6 for davarski/k8s-helm:latest ...
Running on runner--project-0-concurrent-0 via home...
Cloning repository...
Cloning into '/builds/project-0'...
done.
Checking out 6814ad1a as master...
Skipping Git submodules setup
$ export KUBECONFIG=/home/davar/.kube/config
$ helm init --client-only
Creating /root/.helm 
Creating /root/.helm/repository 
Creating /root/.helm/repository/cache 
Creating /root/.helm/repository/local 
Creating /root/.helm/plugins 
Creating /root/.helm/starters 
Creating /root/.helm/cache/archive 
Creating /root/.helm/repository/repositories.yaml 
Adding stable repo with URL: https://kubernetes-charts.storage.googleapis.com 
Adding local repo with URL: http://127.0.0.1:8879/charts 
$HELM_HOME has been configured at /root/.helm.
Not installing Tiller due to 'client-only' flag having been set
Happy Helming!
$ helm  upgrade ui /ui/charts/ui --install  --set image.tag=latest
Release "ui" does not exist. Installing it now.
NAME:   ui
LAST DEPLOYED: Sun Nov 25 23:34:07 2018
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Service
NAME  AGE
ui    2s

==> v1beta1/Deployment
ui  2s

==> v1/Pod(related)

NAME                 READY  STATUS   RESTARTS  AGE
ui-7d95c8ff49-nw4hd  0/1    Pending  0         1s
ui-7d95c8ff49-rnc2z  0/1    Pending  0         1s


NOTES:
1. Get the application URL by running these commands:
  export NODE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services ui)
  export NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT
  export SERVICE_IP=$(kubectl get svc --namespace default ui -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
  echo http://$SERVICE_IP:9292
  export POD_NAME=$(kubectl get pods --namespace default -l "app=ui,release=ui" -o jsonpath="{.items[0].metadata.name}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl port-forward $POD_NAME 8080:9292

Job succeeded
  
```  
  
