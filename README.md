

This guide will focus on building a Jenkins Pipeline running in a `pod` on Kubernetes following security best practices. The result is a simple Pipeline that builds and pushes a container image and deploys it to a pod in a Kubernetes cluster. This guide focus on the integration of an existing Jenkins into Kubernetes using the Groovy syntax for writing pipelines.

## Assumptions

* You are familiar with [Groovy](https://jenkins.io/doc/pipeline/tour/hello-world/)
* You have a Kubernetes cluster running
* You have a standalone Jenkins server.

## Overview
We'll begin by configuring your existing Jenkins instance
with the Kubernetes plugin, allowing Jenkins to create dynamic
`Pod` workers to perform its jobs. The Kubernetes plugin requires
credentials to authenticate to the cluster, which we'll create
and configure as well.

Jenkins will run a job that creates an image using
a standard `Dockerfile`, pulling source code from a specified
repository and layering it on top of it. The resulting image
will then be pushed to an image registry using Kaniko.
`Kaniko` is a tool that allows you to build images
without requiring connectivity to a Docker daemon.
Using `Kaniko` allows us to fully containerize the image
building process, giving Jenkins the ability to build
the image end-to-end using Kubernetes.

## Configuring your Jenkins instance with the Kubernetes plugin
To get started automating deployments using Jenkins the first
step is to configure the Kubernetes plugin in Jenkins. This
plugin provides your Jenkins instance with the capability to
run agents dynamically in Kubernetes. New `Pod`s
will be created and tasked with completing a job and be 
automatically cleaned up after the job completes.
All of the output of the tasks performed by those `Pod`s
can be then reviewed in the Jenkins Job output.

To install the Kubernetes plugin in your Jenkins instance:
* Click Manage Jenkins -> Plugin Manager
* Filter for 'Kubernetes' and select "Kubernetes - This plugin
integrates Jenkins with Kubernetes".
* Click install without restart.

We now need to configure the Kubernetes plugin, but before we
do that we need to have a service account token so it can be
used to use to communicate with Kubernetes.

In your cluster, you'll need to use a service account with 
permissions for Jenkins to perform builds and deployments so
we'll do that next.

## Creating a Role and roleBinding for Jenkins to use

In this pipeline, we will use Jenkins to deploy to different
`namespaces`. In order to comply with security best practices,
it is preferred to create a `Role` and `RoleBinding` in the 
destination `namespaces` and not giving Jenkins a `ClusterRole`
and `ClusterRoleBinding`. 

Also, we'll only give Jenkins the permissions necessary
for it to be able to perform his tasks and not more.

We will write a Jenkins pipeline that deploys in two 
different `namespaces`. For the first `stage` of the pipeline
the `default` `serviceAccount` will be used to build an image in the
`jenkins` `namespace`.

For the second stage, the application will be deployed to the
`test` `namespace`. For this to work, a `Role` and a `RoleBinding` 
will need to be created to allow the Jenkins "default" `serviceAccount`
to create the `deployment` using the resulting image from the first stage.

In your cluster, using `kubectl` while logged in as an admin
create the following `Role` and `RoleBinding` in the `test` namespace:

```Role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: test
  name: jenkins-deployment-control
rules:
- apiGroups: ["extensions", "apps",""]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```
 
```RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins-cross-namespace
  namespace: test
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: jenkins-deployment-control
subjects:
- kind: ServiceAccount
  name: default
  namespace: jenkins
```

If you save the above contents in a YAML file you can easily
create the resources using:
```
kubectl create -f role-and-binding.yaml
```

## Configure the Kubernetes cloud in Jenkins
Now that we have a service account for Jenkins to use, we can get
its token and configure the Kubernetes plugin to use it.

To get the token of the `default` service account for the jenkins
namespace run the following:

```
# kubectl get secrets default-token-XXXXX -o jsonpath={.data.token}|base64 -d
```

The above should give you the decoded token which you can now use
to configure Jenkins. If you want to test it before doing so,
you can do so simply using curl:

```
# curl -H "Authorization: Bearer XXX{TOKEN}XXX --insecure https://10.96.0.1
```
Unless you see a `401 Unauthorized`, your token should be functional.

To configure the Kubernetes plugin with this cluster and token:
* Click "Manage Jenkins" -> Configure System
* At the bottom of the configuration page, click "Add new cloud"->"Kubernetes"
* Add the Kubernetes details:
    * Endpoint (ensure port 6443 is defined)
    * Server certificate can be blank if you'll disable certificate check
    * Add Jenkins Credentials as "Secret text". Use the token
    obtained previously.

![jenkins-k8s-account](/images/jenkins-k8s-account.png)

Additionally, since the TCP port for inbound agents is disabled in
Global Security settings, you'll need to enable it so that 
Jenkins agents running inside kubernetes can talk back to Jenkins:
* Click "Manage Jenkins" -> Configure Global Security
* Select "Random" under "TCP port for inbound agents".


## Creating docker credentials secret for Kaniko

Since we'll use a Kaniko container to build our image,
we need a way for it to be able to push the resulting
image to an image registry. In this guide we'll use Docker Hub.

We'll create a docker config.json file and save it as a
secret so it can be mounted in the Kaniko container during
the build process.

You can obtain a `config.json` by logging in to Docker Hub:
```
docker login
```

After you authenticate you'll be able to find a config.json file
in your home directory under $HOME/.docker/config.json. The file
looks similar to this:

```dockercred
{
        "auths": {
                "https://index.docker.io/v1/": {      
                        "auth": "YxxxxxxxxxxxxxxxxxxxxQ=="
                }
        },
        "HttpHeaders": {
                "User-Agent": "Docker-Client/18.09.7 (linux)"
        }
}
```

We'll use `kubectl create secret` specifing the path of the
`config.json` to create the secret:

```secretCreate
kubectl create secret generic dockercred --from-file=config.json -n jenkins
```

We should now be able to leverage this secret and mount it in the Kaniko
container to be used as registry credentials when pushing resulting images.

## Creating a Jenkins Pipeline
 
The following is a scripted pipeline written in groovy
representing our basic pipeline. Although Jenkins declarative
pipeline is easier to read and implement, in most cases it's
not as flexible for advanced use cases for Kubernetes.

The following Pipeline initializes a `Pod` with 4 containers
that will execute different stages of the pipeline.

Jenkins will automatically inject a Jenkins JNLP agent container
in our `Pod`, so we don't need to specify it.
 
```Jenkinsfile
def label = "my-pipeline-$BUILD_NUMBER"
 
podTemplate(label: label, containers: [
 containerTemplate(name: 'kubectl', image: 'lachlanevenson/k8s-kubectl:v1.8.8', command: 'cat', ttyEnabled: true),
 containerTemplate(name: 'kaniko', image: 'gcr.io/kaniko-project/executor:debug', command: '/busybox/cat', ttyEnabled: true),
 containerTemplate(name: 'multitool', image: 'praqma/network-multitool:latest', command: 'cat', ttyEnabled: true)
],
volumes: [
   secretVolume(mountPath: '/root/.docker/', secretName: 'dockercred')
]) {
 node(label) {
   stage('Stage 1: Build with Kaniko') {
     container('kaniko') {
       sh '/kaniko/executor --context=git://gitlab.com/account/repository.git \
               --destination=docker.io/product/service:1.$BUILD_NUMBER \
               --insecure \
               --skip-tls-verify  \
               -v=debug'
     }
   }
 
   stage('Stage 2: Promote to test') {
     container('kubectl') {
       sh  'kubectl apply -f https://raw.gitlab.com/account/kubernetes-resources/deployment.yaml -n test'
     }
   }

   stage('Stage 3: Simple test') {
     container('multitool') {
       sh  'curl webapp.test.svc.cluster.local'
     }
   }
 }
}
```


## Breakdown
 
First, declare a global variable using the keyword `def` to create a random label value for both the `pod` name and the `label`. This will make sure Jenkins
can initialize a new `pod` with a new name every run.
 
```variables
def label = "my-pipeline-${UUID.randomUUID().toString()}"
```

In the `podTemplate`, we can declare all the fields necessary
for our pod to run like a `containerTemplate`, volumes,
resources, etc.

Inside the `podTemplate` we specify the containers that
will form a pod. In our example, we'll use 3 containers. 

The first container is the Jenkins JNLP agent which
Jenkins injects by default and therefore doesn't need
to be explicitly defined here.

The second (and first in the example) is kubectl container. 
The kubectl container includes the `kubectl` command
which we'll need to interact with the different Kubernetes clusters
related to the pipeline. When running it from within the container
it will use the container's service account permissions.
If we were to interact with a remote cluster, a volume with
the respective kubeconfig would be needed to be mounted as a secret.

The third container in our example is the Kaniko container
image which will be used to build and push the container
image using the credentials given in the `volumes` block.

The fourth is a "network-multitool" container which provides
different networking tools. We'll use it to test the
deployed application.
 
```podTemplate
podTemplate(label: label, containers: [
 containerTemplate(name: 'kubectl', image: 'lachlanevenson/k8s-kubectl:v1.8.8', command: 'cat', ttyEnabled: true),
 containerTemplate(name: 'kaniko', image: 'gcr.io/kaniko-project/executor:debug', command: '/busybox/cat', ttyEnabled: true),
 containerTemplate(name: 'multitool', image: 'praqma/network-multitool:latest', command: 'cat', ttyEnabled: true)
],
volumes: [
   secretVolume(mountPath: '/root/.docker/', secretName: 'dockercred')
])
```

In Scripted Pipeline syntax, one or more `node` blocks do
the core work throughout the entire Pipeline. In the `node`
brackets, the label variable is used to specify the
`podTemplate` that will use to run the Pipeline stages.
More than one `node{}` can be used inside the Jenkinsfile. 
Each `node` consists of multiple `stages`, each specifies
the `container` that will be used to perform a task
part of the pipeline. 

```node
 node(label) {
   stage('Stage 1: Build with Kaniko') {
     container('kaniko') {
       sh '/kaniko/executor --context=git://gitlab.com/account/repository.git \
               --destination=docker.io/product/service:1.$BUILD_NUMBER \
               --insecure \
               --skip-tls-verify  \
               -v=debug'
     }
   }
 
   stage('Stage 2: Promote to test') {
     container('kubectl') {
      sh 'kubectl apply - fhttps://raw.gitlab.com/account/kubernetes-resources/deployment.yaml -n test -n test'
     }
   }

   stage('Stage 3: Simple test') {
     container('multitool') {
       sh  'curl webapp.test.svc.cluster.local'
     }
   }
 }
```

The following is a working example of the pipeline above

```examplePipeline
def label = "goweb-1.$BUILD_NUMBER-pipeline"
 
podTemplate(label: label, containers: [
 containerTemplate(name: 'kubectl', image: 'lachlanevenson/k8s-kubectl:v1.8.8', command: 'cat', ttyEnabled: true),
 containerTemplate(name: 'kaniko', image: 'gcr.io/kaniko-project/executor:debug', command: '/busybox/cat', ttyEnabled: true),
 containerTemplate(name: 'multitool', image: 'praqma/network-multitool:latest', command: 'cat', ttyEnabled: true)
],
volumes: [
   secretVolume(mountPath: '/root/.docker/', secretName: 'dockercred')
]) {
 node(label) {
   stage('Stage 1: Build with Kaniko') {
     container('kaniko') {
       sh '/kaniko/executor --context=git://github.com/ahirshberg/goPlay.git \
               --destination=docker.io/cics89/hirshcorp:latest \
               --insecure \
               --skip-tls-verify  \
               -v=debug'
     }
   }
 
   stage('Stage 2: Run kubectl again') {
     container('kubectl') {
       sh  'kubectl apply -f https://raw.githubusercontent.com/ahirshberg/CNR-examples/master/wepapp-deployment.yaml -n test'
     }
   }
   stage('Stage 3: test') {
     container('multitool') {
       sh  'curl webapp.test.svc.cluster.local'
     }
   }
 }
}
```
