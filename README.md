# microservises app: bluepost

This is an example of a microservices application **Gitlab CI**

$  git clone https://github.com/adavarski/gitlab-group-microservices
$ cd gitlab-group-microservices
$ rm -rf .git

In Gitlab UI, I’ll create a new group for my bluepost application

Gitlab CI groups allow to group related projects. In our example, bluepost group will contain the microservices that the bluepost application consists of.

And create a new project in Gitlab CI web UI for each component of bluepost application

Describe a CI/CD pipeline for each project
Each component of the bluepost application is contained in its own repository and has its own CI/CD pipeline defined in a .gitlab-ci-yml file (which has a special meaning for Gitlab CI) stored in the root of each of the component’s directory.

For each project define variables: Settings -> CI/CD. Define CI_REGISTRY_USER and CI_REGISTRY_PASSWORD variables to allow logging to Docker Hub.

