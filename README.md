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

Gitlab CI has its own container registry, but in this example we use Docker Hub,  you’ll need to make sure the CONTAINER_IMAGE variable is set according to your Docker Hub account name repository for each image is created. For each project define variables: Settings -> CI/CD. Define CI_REGISTRY_USER and CI_REGISTRY_PASSWORD variables to allow logging to Docker Hub. After define projects and setup CI/CD pipeline env variables execute:

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


