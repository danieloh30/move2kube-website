---
layout: default
title: "Migrating and deploying Cloud Foundry apps to Kubernetes"
permalink: /tutorials/cloud-foundry/
parent: Tutorials
nav_order: 4
---

# Migrating and deploying Cloud Foundry applications to Kubernetes

## Description

This document takes us through the steps that will install `Move2Kube` and use Move2Kube's 3 step process (collect, plan and transform) to create deployment artifacts for Cloud Foundry apps. Here, we are going to use the data from [samples/cloud-foundry](https://github.com/konveyor/move2kube-demos/tree/main/samples/cloud-foundry).

### TLDR

```console
$ move2kube transform -s cloud-foundry
```

Move2Kube will automatically analyse all the artifacts in the cloud-foundry directory and transform and create all artifacts required for deploying the application in Kubernetes.

## Prerequisites

1. A source directory which contains the source code files and/or the manifest.yml file of a Cloud Foundry application.

   A sample of this is present in the [move2kube-demos](https://github.com/konveyor/move2kube-demos) repository. In this tutorial, we will be using the `cloud-foundry` sample from this repository.
    ```console
    $ curl https://move2kube.konveyor.io/scripts/download.sh | bash -s -- -d samples/cloud-foundry -r move2kube-demos
    ```

   Let's see the structure inside the `./cloud-foundry` directory. The `cloud-foundry` directory contains the source code files and the manifest.yml file.

   ```console
  $ tree cloud-foundry
  cloud-foundry/
  ├── cfnodejsapp
  │   ├── main.js
  │   ├── manifest.yml
  │   ├── package-lock.json
  │   └── package.json
  └── m2k_collect
      └── cf
          └── cfapps.yaml
   ```

1. Install [Move2Kube](/installation)

1. Install a container runtime: [Docker](https://www.docker.com/get-started) or [Podman](https://podman.io/getting-started/installation)

1. Install [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl)

   To verify that dependencies were correctly installed you can run the below commands.
   ```console
   $ move2kube version
   ```
   ```console
   $ docker version
   ```
   or
   ```console
   $ podman info
   ```
   ```console
   $ kubectl version
   ```

1. Install the [Cloud Foundry CLI](https://docs.cloudfoundry.org/cf-cli/install-go-cli.html)

1. To demonstrate how to use Move2Kube to migrate a Cloud Foundry (CF) application to Kubernetes, we will be using the source code inside the `cloud-foundry/cfnodejsapp` directory. If you want to try out Move2Kube on your CF application, then in place of our sample `cloud-foundry` directory, you should provide the correct path of the source directory (containing the source code and/or manifest files) of your CF application to Move2Kube during the *plan* phase.

1. Optional: We will deploy a simple nodejs application into CF. If you have a running CF app already you may use that instead. Provision a CF app with the name `cfnodejsapp` using your cloud provider (Ex: [IBM Cloud](https://cloud.ibm.com/)).
   1. Make note of the API endpoint (API endpoints for the IBM Cloud Foundry service can be found [here](https://cloud.ibm.com/docs/cloud-foundry-public?topic=cloud-foundry-public-endpoints)).
   1. Login to CF using
       ```console
       $ cf login -a <YOUR CF API endpoint>
       ```
   1. Run the below commands to deploy the sample application to Cloud Foundry. The source code of the sample application is present inside `./cloud-foundry/cfnodejsapp` folder.

       ```console
       $ cf push -f ./cloud-foundry/cfnodejsapp -p ./cloud-foundry/cfnodejsapp
       ```

   1. You can visit the URL of the application (you can get this by running `cf apps`) to see it running.


## Steps to generate target artifacts

Now that we have a running CF app we can transform it using Move2Kube. We will be using the three stage process (*collect*, *plan* and *transform*) for the transformation. Run these steps from the directory where `cloud-foundry` directory is present:

1. **Optional**: This step is required only if you are interested in metadata such as environment variables from a running instance. If you don't have a running app, you can use the m2k_collect directoy that comes with the sample.
 We will first collect some data about your running CF application. Here we will limit the collect to only cloud foundry information using the `-a cf` annotation flag.  

    ```console
    $ cf login -a <YOUR CF API endpoint>
    $ move2kube collect -a cf
    INFO[0000] Begin collection                             
    INFO[0000] [*collector.CfAppsCollector] Begin collection 
    INFO[0013] [*collector.CfAppsCollector] Done            
    INFO[0013] [*collector.CfServicesCollector] Begin collection 
    INFO[0027] [*collector.CfServicesCollector] Done        
    INFO[0027] Collection done                              
    INFO[0027] Collect Output in [/Users/username/m2k_collect]. Copy this directory into the source directory to be used for planning.
    ```

    * The data we collected will be stored in a new directory called `./m2k_collect`.

    ```console
    $ ls m2k_collect
    cf
    ```

    * The `./m2k_collect/cf` directory contains the yaml file which has the runtime information of the particular application that you are trying to transform. There is information about the buildpacks that are supported, the memory, the number of instances and the ports that are supported. If there are environment variables, it collects that information too.

    * Move the `./m2k_collect/cf` directory into the source directory `./cloud-foundry`.

    ```console
    $ mv m2k_collect cloud-foundry/
    ```

2. Then we create a *plan* on how to transform your app to run on Kubernetes. In the *plan* phase, it is going to combine the runtime metadata (if present) with source artifacts and come up with a *plan* for us. Here, we provide to Move2Kube the path to the source directory (containing the source code and/or manifest files of CF application) using the `-s` flag.

    ```console
    $ move2kube plan -s cloud-foundry
    INFO[0000] Configuration loading done                   
    INFO[0000] Planning Transformation - Base Directory     
    INFO[0000] [CloudFoundry] Planning transformation       
    INFO[0000] Identified 1 named services and 0 to-be-named services 
    INFO[0000] [CloudFoundry] Done                          
    INFO[0000] [ComposeAnalyser] Planning transformation    
    INFO[0000] [ComposeAnalyser] Done                       
    INFO[0000] [DockerfileDetector] Planning transformation 
    INFO[0000] [DockerfileDetector] Done                    
    INFO[0000] [Base Directory] Identified 1 named services and 0 to-be-named services 
    INFO[0000] Transformation planning - Base Directory done 
    INFO[0000] Planning Transformation - Directory Walk     
    INFO[0000] Identified 1 named services and 0 to-be-named services in cfnodejsapp 
    INFO[0000] Transformation planning - Directory Walk done 
    INFO[0000] [Directory Walk] Identified 1 named services and 0 to-be-named services 
    INFO[0000] [Named Services] Identified 1 named services 
    INFO[0000] No of services identified : 1               
    INFO[0000] Plan can be found at [/Users/username/m2k.plan].
    ```

    * It has created a *m2k.plan* which is essentially a yaml file. Let's see what is inside the *plan* file.

    ```console
    $ cat m2k.plan
    ```
    <details markdown="block">
    <summary markdown="block">
    ```yaml
    # click to see the full plan yaml
    apiVersion: move2kube.konveyor.io/v1alpha1
    kind: Plan
    .......
    ```
    </summary>
    ```yaml
      apiVersion: move2kube.konveyor.io/v1alpha1
      kind: Plan
      metadata:
        name: myproject
      spec:
        sourceDir: cloud-foundry
        services:
          cfnodejsapp:
            - transformerName: CloudFoundry
              paths:
                CfManifest:
                  - cfnodejsapp/manifest.yml
                CfRunningManifest:
                  - m2k_collect/cf/cfapps.yaml
                ServiceDirPath:
                  - cfnodejsapp
              configs:
                CloudFoundryService:
                  serviceName: cfnodejsapp
                ContainerizationOptions:
                  - Nodejs-Dockerfile
            - transformerName: Nodejs-Dockerfile
              paths:
                ServiceDirPath:
                  - cfnodejsapp
        transformers:
          Buildconfig: m2kassets/built-in/transformers/kubernetes/buildconfig/buildconfig.yaml
          CloudFoundry: m2kassets/built-in/transformers/cloudfoundry/cloudfoundry.yaml
          ClusterSelector: m2kassets/built-in/transformers/kubernetes/clusterselector/clusterselector.yaml
          ComposeAnalyser: m2kassets/built-in/transformers/compose/composeanalyser/composeanalyser.yaml
          ComposeGenerator: m2kassets/built-in/transformers/compose/composegenerator/composegenerator.yaml
          ContainerImagesPushScriptGenerator: m2kassets/built-in/transformers/containerimage/containerimagespushscript/containerimagespushscript.yaml
          DockerfileDetector: m2kassets/built-in/transformers/dockerfile/dockerfiledetector/dockerfiledetector.yaml
          DockerfileImageBuildScript: m2kassets/built-in/transformers/dockerfile/dockerimagebuildscript/dockerfilebuildscriptgenerator.yaml
          DockerfileParser: m2kassets/built-in/transformers/dockerfile/dockerfileparser/dockerfileparser.yaml
          DotNetCore-Dockerfile: m2kassets/built-in/transformers/dockerfilegenerator/dotnetcore/dotnetcore.yaml
          EarAnalyser: m2kassets/built-in/transformers/dockerfilegenerator/java/earanalyser/ear.yaml
          EarRouter: m2kassets/built-in/transformers/dockerfilegenerator/java/earrouter/earrouter.yaml
          Golang-Dockerfile: m2kassets/built-in/transformers/dockerfilegenerator/golang/golang.yaml
          Gradle: m2kassets/built-in/transformers/dockerfilegenerator/java/gradle/gradle.yaml
          Jar: m2kassets/built-in/transformers/dockerfilegenerator/java/jar/jar.yaml
          Jboss: m2kassets/built-in/transformers/dockerfilegenerator/java/jboss/jboss.yaml
          Knative: m2kassets/built-in/transformers/kubernetes/knative/knative.yaml
          Kubernetes: m2kassets/built-in/transformers/kubernetes/kubernetes/kubernetes.yaml
          KubernetesVersionChanger: m2kassets/built-in/transformers/kubernetes/kubernetesversionchanger/kubernetesversionchanger.yaml
          Liberty: m2kassets/built-in/transformers/dockerfilegenerator/java/liberty/liberty.yaml
          Maven: m2kassets/built-in/transformers/dockerfilegenerator/java/maven/maven.yaml
          Nodejs-Dockerfile: m2kassets/built-in/transformers/dockerfilegenerator/nodejs/nodejs.yaml
          PHP-Dockerfile: m2kassets/built-in/transformers/dockerfilegenerator/php/php.yaml
          Parameterizer: m2kassets/built-in/transformers/kubernetes/parameterizer/parameterizer.yaml
          Python-Dockerfile: m2kassets/built-in/transformers/dockerfilegenerator/python/python.yaml
          ReadMeGenerator: m2kassets/built-in/transformers/readmegenerator/readmegenerator.yaml
          Ruby-Dockerfile: m2kassets/built-in/transformers/dockerfilegenerator/ruby/ruby.yaml
          Rust-Dockerfile: m2kassets/built-in/transformers/dockerfilegenerator/rust/rust.yaml
          Tekton: m2kassets/built-in/transformers/kubernetes/tekton/tekton.yaml
          Tomcat: m2kassets/built-in/transformers/dockerfilegenerator/java/tomcat/tomcat.yaml
          WarAnalyser: m2kassets/built-in/transformers/dockerfilegenerator/java/waranalyser/war.yaml
          WarRouter: m2kassets/built-in/transformers/dockerfilegenerator/java/warrouter/warrouter.yaml
          WinConsoleApp-Dockerfile: m2kassets/built-in/transformers/dockerfilegenerator/windows/winconsole/winconsole.yaml
          WinSLWebApp-Dockerfile: m2kassets/built-in/transformers/dockerfilegenerator/windows/winsilverlightweb/winsilverlightweb.yaml
          WinWebApp-Dockerfile: m2kassets/built-in/transformers/dockerfilegenerator/windows/winweb/winweb.yaml
          ZuulAnalyser: m2kassets/built-in/transformers/dockerfilegenerator/java/zuul/zuulanalyser.yaml
    ```
    </details>

    * In the plan, you can see that Move2Kube has detected the `cfnodejsapp` service, which is the name of our sample CF application from it's manifest.yml.
    * And the plan file is saying that the application can be transformed using two transformers `CloudFoundry` or `Nodejs-Dockerfile`.
    * It can use the source artifacts `manifest.yaml` and also the runtime information from `cfapps.yaml` and combine all of them and do the transformation. 

3. Let's invoke `move2kube transform` on this *plan*.

    ```console
    $ move2kube transform
    INFO[0000] Detected a plan file at path /Users/username/m2k.plan. Will transform using this plan. 
    INFO[0000] Starting Plan Transformation                 
    ? Select all transformer types that you are interested in:
    ID: move2kube.transformers.types
    Hints:
    [Services that don't support any of the transformer types you are interested in will be ignored.]
      [Use arrows to move, space to select, <right> to all, <left> to none, type to filter]
      > [✓]  Buildconfig
        [✓]  CloudFoundry
        [✓]  ClusterSelector
        [✓]  ComposeAnalyser
        [✓]  ComposeGenerator
        [✓]  ContainerImagesPushScriptGenerator
        [✓]  DockerfileDetector
        [✓]  DockerfileParser
        [✓]  DotNetCore-Dockerfile
        [✓]  EarAnalyser
        [✓]  EarRouter
        [✓]  Golang-Dockerfile
        [✓]  Gradle
        [✓]  Jar
        [✓]  Jboss
        [✓]  Knative
        [✓]  Kubernetes
        [✓]  KubernetesVersionChanger
        [✓]  Liberty
        [✓]  Maven
        [✓]  Nodejs-Dockerfile
        [✓]  PHP-Dockerfile
        [✓]  Parameterizer
        [✓]  Python-Dockerfile
        [✓]  ReadMeGenerator
        [✓]  Ruby-Dockerfile
        [✓]  Rust-Dockerfile
        [✓]  Tekton
        [✓]  Tomcat
        [✓]  WarAnalyser
        [✓]  WarRouter
        [✓]  WinConsoleApp-Dockerfile
        [✓]  WinSLWebApp-Dockerfile
        [✓]  WinWebApp-Dockerfile
        [✓]  ZuulAnalyser
    ```

    * Let's go ahead with the default answer by pressing `return` or `enter` key.

    ```console
    Buildconfig, CloudFoundry, ClusterSelector, ComposeAnalyser, ComposeGenerator, ContainerImagesPushScriptGenerator, DockerfileDetector, DockerfileImageBuildScript, DockerfileParser, DotNetCore-Dockerfile, EarAnalyser, EarRouter, Golang-Dockerfile, Gradle, Jar, Jboss, Knative, Kubernetes, KubernetesVersionChanger, Liberty, Maven, Nodejs-Dockerfile, PHP-Dockerfile, Parameterizer, Python-Dockerfile, ReadMeGenerator, Ruby-Dockerfile, Rust-Dockerfile, Tekton, Tomcat, WarAnalyser, WarRouter, WinConsoleApp-Dockerfile, WinSLWebApp-Dockerfile, WinWebApp-Dockerfile, ZuulAnalyser
    ? Select all services that are needed:
    ID: move2kube.services.[].enable
    Hints:
    [The services unselected here will be ignored.]
      [Use arrows to move, space to select, <right> to all, <left> to none, type to filter]
    > [✓]  cfnodejsapp
    ```

    * Here, we go ahead with the `cfnodejsapp` service.

    ```console
     cfnodejsapp
    INFO[0555] Iteration 1                                  
    INFO[0555] Iteration 2 - 1 artifacts to process         
    INFO[0555] Transformer CloudFoundry processing 1 artifacts 
    INFO[0555] Transformer CloudFoundry Done                
    INFO[0555] Created 0 pathMappings and 1 artifacts. Total Path Mappings : 0. Total Artifacts : 1. 
    INFO[0555] Iteration 3 - 1 artifacts to process         
    INFO[0555] Transformer Nodejs-Dockerfile processing 1 artifacts 
    ? Select port to be exposed for the service cfnodejsapp :
    ID: move2kube.services.cfnodejsapp.port
    Hints:
    [Select Other if you want to expose the service cfnodejsapp to some other port]
      [Use arrows to move, type to filter]
    > 8080
      Other (specify custom option)
    ```

    * Select the port on which you want to expose the `cfnodejsapp` service.

    ```console
     8080
    INFO[0675] Transformer Nodejs-Dockerfile Done           
    INFO[0675] Created 2 pathMappings and 2 artifacts. Total Path Mappings : 2. Total Artifacts : 2. 
    INFO[0675] Iteration 4 - 2 artifacts to process         
    INFO[0675] Transformer DockerfileImageBuildScript processing 2 artifacts 
    ? Select the container runtime to use :
    ID: move2kube.containerruntime
    Hints:
    [The container runtime selected will be used in the scripts]
      [Use arrows to move, type to filter]
    > docker
      podman
    ```

    * Select the container runtime you want to use.

    ```console
     docker
    INFO[0744] Transformer DockerfileImageBuildScript Done  
    INFO[0744] Transformer DockerfileParser processing 1 artifacts 
    INFO[0744] Transformer ZuulAnalyser processing 2 artifacts 
    INFO[0744] Transformer ZuulAnalyser Done                
    INFO[0744] Transformer DockerfileParser Done            
    INFO[0744] Created 1 pathMappings and 4 artifacts. Total Path Mappings : 3. Total Artifacts : 4. 
    INFO[0744] Iteration 5 - 4 artifacts to process         
    INFO[0744] Transformer ClusterSelector processing 2 artifacts 
    ? Choose the cluster type:
    ID: move2kube.target.clustertype
    Hints:
    [Choose the cluster type you would like to target]
      [Use arrows to move, type to filter]
    > Openshift
      AWS-EKS
      Azure-AKS
      GCP-GKE
      IBM-IKS
      IBM-Openshift
      Kubernetes
    ```

    * Now, it asks to select the cluster type you want to deploy to. Here, we select the `Openshift` cluster type.

    ```console
     Openshift
    INFO[0842] Transformer ClusterSelector Done             
    INFO[0842] Transformer Buildconfig processing 2 artifacts 
    ? What kind of service/ingress to create for cfnodejsapp's 8080 port?
    ID: move2kube.services."cfnodejsapp"."8080".servicetype
    Hints:
    [Choose Ingress if you want a ingress/route resource to be created]
      [Use arrows to move, type to filter]
      Ingress
      LoadBalancer
      NodePort
    > ClusterIP
      Don't create service
    ```

    * Select the `ClusterIP` to create a service in OpenShift cluster.

    ```console
     ClusterIP
    ? Provide the minimum number of replicas each service should have
    ID: move2kube.minreplicas
    Hints:
    [If the value is 0 pods won't be started by default]
    (2)
    ```

    * Let's go ahead with the default answer again, which means `2` replicas for each service.

    ```console
    2
    ? Enter the URL of the image registry : 
    Hints:
    [You can always change it later by changing the yamls.]
      [Use arrows to move, type to filter]
      Other (specify custom option)
      index.docker.io
    > quay.io
      us.icr.io
    ```

    * Then it asks to select the registry where your images are hosted. Here, we are selecting `quay.io`. Select 'Other' if your registry name is not here.

    ```console
     quay.io
    ? Enter the namespace where the new images should be pushed : 
    ID: move2kube.target.imageregistry.namespace
    Hints:
    [Ex : myproject]
    (myproject) username
    ```

    * Input the username or org. that you want to push an image. For example, if you enter **danieloh30** as the username, the image path will be **quay.io/danieloh30/cfnodejsapp**.

    ```console
     danieloh30
    ? [quay.io] What type of container registry login do you want to use?
    ID: move2kube.target.imageregistry.logintype
    Hints:
    [Docker login from config mode, will use the default config from your local machine.]
      [Use arrows to move, type to filter]
      Use existing pull secret
    > No authentication
      UserName/Password                 
    ```

    * Select `No authentication` for the container registry login type.

    ```console
     No authentication
    INFO[2439] Generating Buildconfig pipeline for CI/CD    
    INFO[2439] Generating Buildconfig pipeline for CI/CD    
    INFO[2439] Transformer Buildconfig Done                 
    INFO[2439] Transformer ComposeGenerator processing 2 artifacts 
    INFO[2439] Transformer ComposeGenerator Done            
    INFO[2439] Transformer ContainerImagesPushScriptGenerator processing 2 artifacts 
    INFO[2439] Transformer ContainerImagesPushScriptGenerator Done 
    INFO[2439] Transformer ClusterSelector processing 2 artifacts 
    INFO[2439] Transformer ClusterSelector Done             
    INFO[2439] Transformer Knative processing 2 artifacts   
    INFO[2439] Transformer Knative Done                     
    INFO[2439] Transformer ClusterSelector processing 2 artifacts 
    INFO[2439] Transformer ClusterSelector Done             
    INFO[2439] Transformer Kubernetes processing 2 artifacts 
    INFO[2439] Transformer Kubernetes Done                  
    INFO[2439] Transformer ClusterSelector processing 2 artifacts 
    INFO[2439] Transformer ClusterSelector Done             
    INFO[2439] Transformer Tekton processing 2 artifacts    
    ? Provide the ingress host domain
    ID: move2kube.target.ingress.host
    Hints:
    [Ingress host domain is part of service URL]
    (myproject.com)
    ```

    * It is now asking for the ingress hosting domain. It can be grabbed from the cluster you are going to deploy to. However, you don't need to use the Ingress because the OpenShift cluster allows you to create a `Route` to access the application based on FQDN domain. Go ahead with the default answer by pressing `return` or `enter` key.

    ```console
    ID: move2kube.target.ingress.host
    Hints:
    [Ingress host domain is part of service URL]
    myproject.com
    INFO[2780] Transformer Tekton Done                      
    INFO[2780] Created 35 pathMappings and 9 artifacts. Total Path Mappings : 38. Total Artifacts : 8. 
    INFO[2780] Iteration 6 - 9 artifacts to process         
    INFO[2780] Transformer Parameterizer processing 5 artifacts 
    INFO[2780] Transformer Parameterizer Done               
    INFO[2780] Transformer ReadMeGenerator processing 6 artifacts 
    INFO[2780] Transformer ReadMeGenerator Done             
    INFO[2780] Plan Transformation done                     
    INFO[2780] Transformed target artifacts can be found at [/Users/username/myproject].
    ```

Finally, the transformation is successful and the target artifacts can be found inside the `./myproject` directory. The structure of the *./myproject* directory can be seen by executing the below command.

<details markdown="block">
<summary markdown="block">
```console
# click to see the output
$  tree myproject
```

</summary>

```console
$  tree myproject
myproject
├── Readme.md
├── deploy
│   ├── cicd
│   │   ├── buildconfig
│   │   │   ├── myproject-clone-build-push-cfnodejsapp-buildconfig.yaml
│   │   │   ├── myproject-git-repo-generic-secret.yaml
│   │   │   └── myproject-web-hook-cfnodejsapp-secret.yaml
│   │   ├── buildconfig-parameterized
│   │   │   ├── helm-chart
│   │   │   │   └── myproject
│   │   │   │       ├── Chart.yaml
│   │   │   │       └── templates
│   │   │   │           ├── myproject-clone-build-push-cfnodejsapp-buildconfig.yaml
│   │   │   │           ├── myproject-git-repo-generic-secret.yaml
│   │   │   │           └── myproject-web-hook-cfnodejsapp-secret.yaml
│   │   │   ├── kustomize
│   │   │   │   └── base
│   │   │   │       ├── kustomization.yaml
│   │   │   │       ├── myproject-clone-build-push-cfnodejsapp-buildconfig.yaml
│   │   │   │       ├── myproject-git-repo-generic-secret.yaml
│   │   │   │       └── myproject-web-hook-cfnodejsapp-secret.yaml
│   │   │   └── openshift-template
│   │   │       └── template.yaml
│   │   ├── tekton
│   │   │   ├── cfnodejsapp-vcapasenv-secret.yaml
│   │   │   ├── myproject-clone-build-push-pipeline.yaml
│   │   │   ├── myproject-clone-push-serviceaccount.yaml
│   │   │   ├── myproject-git-event-triggerbinding.yaml
│   │   │   ├── myproject-git-repo-eventlistener.yaml
│   │   │   ├── myproject-git-repo-route.yaml
│   │   │   ├── myproject-image-registry-secret.yaml
│   │   │   ├── myproject-run-clone-build-push-triggertemplate.yaml
│   │   │   ├── myproject-tekton-triggers-admin-role.yaml
│   │   │   ├── myproject-tekton-triggers-admin-rolebinding.yaml
│   │   │   └── myproject-tekton-triggers-admin-serviceaccount.yaml
│   │   └── tekton-parameterized
│   │       ├── helm-chart
│   │       │   └── myproject
│   │       │       ├── Chart.yaml
│   │       │       └── templates
│   │       │           ├── cfnodejsapp-vcapasenv-secret.yaml
│   │       │           ├── myproject-clone-build-push-pipeline.yaml
│   │       │           ├── myproject-clone-push-serviceaccount.yaml
│   │       │           ├── myproject-git-event-triggerbinding.yaml
│   │       │           ├── myproject-git-repo-eventlistener.yaml
│   │       │           ├── myproject-git-repo-route.yaml
│   │       │           ├── myproject-image-registry-secret.yaml
│   │       │           ├── myproject-run-clone-build-push-triggertemplate.yaml
│   │       │           ├── myproject-tekton-triggers-admin-role.yaml
│   │       │           ├── myproject-tekton-triggers-admin-rolebinding.yaml
│   │       │           └── myproject-tekton-triggers-admin-serviceaccount.yaml
│   │       ├── kustomize
│   │       │   └── base
│   │       │       ├── cfnodejsapp-vcapasenv-secret.yaml
│   │       │       ├── kustomization.yaml
│   │       │       ├── myproject-clone-build-push-pipeline.yaml
│   │       │       ├── myproject-clone-push-serviceaccount.yaml
│   │       │       ├── myproject-git-event-triggerbinding.yaml
│   │       │       ├── myproject-git-repo-eventlistener.yaml
│   │       │       ├── myproject-git-repo-route.yaml
│   │       │       ├── myproject-image-registry-secret.yaml
│   │       │       ├── myproject-run-clone-build-push-triggertemplate.yaml
│   │       │       ├── myproject-tekton-triggers-admin-role.yaml
│   │       │       ├── myproject-tekton-triggers-admin-rolebinding.yaml
│   │       │       └── myproject-tekton-triggers-admin-serviceaccount.yaml
│   │       └── openshift-template
│   │           └── template.yaml
│   ├── compose
│   │   └── docker-compose.yaml
│   ├── knative
│   │   └── cfnodejsapp-service.yaml
│   ├── knative-parameterized
│   │   ├── helm-chart
│   │   │   └── myproject
│   │   │       ├── Chart.yaml
│   │   │       └── templates
│   │   │           └── cfnodejsapp-service.yaml
│   │   ├── kustomize
│   │   │   └── base
│   │   │       ├── cfnodejsapp-service.yaml
│   │   │       └── kustomization.yaml
│   │   └── openshift-template
│   │       └── template.yaml
│   ├── yamls
│   │   ├── cfnodejsapp-deployment.yaml
│   │   ├── cfnodejsapp-latest-imagestream.yaml
│   │   ├── cfnodejsapp-service.yaml
│   │   └── cfnodejsapp-vcapasenv-secret.yaml
│   └── yamls-parameterized
│       ├── helm-chart
│       │   └── myproject
│       │       ├── Chart.yaml
│       │       ├── templates
│       │       │   ├── cfnodejsapp-deployment.yaml
│       │       │   ├── cfnodejsapp-latest-imagestream.yaml
│       │       │   ├── cfnodejsapp-service.yaml
│       │       │   └── cfnodejsapp-vcapasenv-secret.yaml
│       │       ├── values-dev.yaml
│       │       ├── values-prod.yaml
│       │       └── values-staging.yaml
│       ├── kustomize
│       │   ├── base
│       │   │   ├── cfnodejsapp-deployment.yaml
│       │   │   ├── cfnodejsapp-latest-imagestream.yaml
│       │   │   ├── cfnodejsapp-service.yaml
│       │   │   ├── cfnodejsapp-vcapasenv-secret.yaml
│       │   │   └── kustomization.yaml
│       │   └── overlays
│       │       ├── dev
│       │       │   ├── apps-v1-deployment-cfnodejsapp.yaml
│       │       │   └── kustomization.yaml
│       │       ├── prod
│       │       │   ├── apps-v1-deployment-cfnodejsapp.yaml
│       │       │   └── kustomization.yaml
│       │       └── staging
│       │           ├── apps-v1-deployment-cfnodejsapp.yaml
│       │           └── kustomization.yaml
│       └── openshift-template
│           ├── parameters-dev.yaml
│           ├── parameters-prod.yaml
│           ├── parameters-staging.yaml
│           └── template.yaml
├── scripts
│   ├── builddockerimages.bat
│   ├── builddockerimages.sh
│   ├── pushimages.bat
│   └── pushimages.sh
└── source
    ├── cfnodejsapp
    │   ├── Dockerfile
    │   ├── main.js
    │   ├── manifest.yml
    │   ├── package-lock.json
    │   └── package.json
    └── m2k_collect
        └── cf
            └── cfapps.yaml

44 directories, 93 files
```
</details>

So, Move2Kube has created all the deployment artifacts which are present inside the *./myproject* directory.

## Deploying the application to Kubernetes with the generated target artifacts

1. Let's get inside the *./myproject* directory.

     ```console
     $ cd myproject/

     $ ls
     Readme.md deploy    scripts   source
     ```

2. Next we run the *builddockerimages.sh* script inside the `./myproject/scripts` directory. This step may take some time to complete.

    ```console
    $ cd scripts
    ```

    ```console
    $ ./builddockerimages.sh
    [+] Building 1.9s (8/8) FINISHED                                                                                                                       
    => [internal] load build definition from Dockerfile                                                                                              0.1s
    => => transferring dockerfile: 747B                                                                                                              0.0s
    => [internal] load .dockerignore                                                                                                                 0.0s
    => => transferring context: 2B                                                                                                                   0.0s
    => [internal] load metadata for registry.access.redhat.com/ubi8/nodejs-12:latest                                                                 1.7s
    => [internal] load build context                                                                                                                 0.0s
    => => transferring context: 868B                                                                                                                 0.0s
    => [1/3] FROM registry.access.redhat.com/ubi8/nodejs-12@sha256:1208ace959a40906e0e0e753b5ed5621c052a5a115e333d70ca8fa5e5c0dc0ca                  0.0s
    => CACHED [2/3] COPY . .                                                                                                                         0.0s
    => CACHED [3/3] RUN npm install                                                                                                                  0.0s
    => exporting to image                                                                                                                            0.0s
    => => exporting layers                                                                                                                           0.0s
    => => writing image sha256:2d560a9a7b40f2d29447b57b619d55a5dd103ae5dd626cd517791f4ae8e61cbb                                                      0.0s
    => => naming to docker.io/library/cfnodejsapp                                                                                                    0.0s
    /Users/username/myproject
    done
    ```

3. Now using the *pushimages.sh* script we can push our applications images to the registry that we specified during the *transform* phase. For this step, you are required to log in to your Docker registry. To log in to `quay.io` run `docker login quay.io`. To log in to IBM Cloud `us.icr.io` registry refer [here](https://cloud.ibm.com/docs/Registry?topic=Registry-registry_access#registry_access_apikey_auth_docker).

    ```console
    $ ./pushimages.sh
    Using default tag: latest
    The push refers to repository [quay.io/danieloh30/cfnodejsapp]
    44682b6a7ca8: Layer already exists 
    4d994360499e: Layer already exists 
    31226b9a7823: Layer already exists 
    718375ad3f65: Layer already exists 
    e0b9a99c51af: Layer already exists 
    ef5d9ad2a541: Layer already exists 
    93749af418e7: Layer already exists 
    latest: digest: sha256:197c820069a3691c727db3ccf0d544601035fdca6dba7b96bae16463068a5307 size: 1789
    ```

    NOTE: If you have pushed the image repository to `quay.io`, then in the Repository's Visibility in [quay.io](https://quay.io) `cfnodejsapp` repository's Settings, select whether you want the repository `cfnodejsapp` to be public or private so that it can be properly accessed by the Kubernetes cluster.

4. Finally we are going to deploy the application with *oc apply* command using the yaml files which Move2Kube has created for us inside the `./myproject/deploy/yamls` directory.

    You'll use [Developer Sandbox for Red Hat OpenShift](https://developers.redhat.com/developer-sandbox), a shared multi-tenant OpenShift cluster that is pre-configured with a set of developer tools. To access the sandbox, visit [Get started in the Sandbox](https://developers.redhat.com/developer-sandbox/get-started) then sign it up for free. 

    You'll use `oc` command line tool over `kubectl` to create Kubernetes resources in the OpenShift cluster. If you haven't already installed the oc command line tool, read the article,[Where can I download the OpenShift command line tool?](https://developers.redhat.com/openshift/command-line-tools).

    Before you deploy the application, you need to log in to the OpenShift cluster on the sandbox. You can copy the `oc` login command with a valid token by the following screen flow:

    ![devsandbox](../../assets/images/samples/dev-sandbox/oclogin.png)

    When you paste the oc login command on your local terminal, the output should looke like:
 
    ```console
    $ oc login --token=sha256~_4yyHH7hAANsx7BchfB4xkclvdmWUAdt-efqZt0MVOQ --server=https://api.sandbox.x8i5.p1.openshiftapps.com:6443

    Logged into "https://api.sandbox.x8i5.p1.openshiftapps.com:6443" as "USERNAME" using the token provided.

    You have access to the following projects and can switch between them with 'oc project <projectname>':

      * USERNAME-dev
        USERNAME-stage

    Using project "doh-dev".
    ```

    **USERNAME** should be replaced with your own account. Be sure to log in to `USERNAME-dev` project.



    ```console
    $ cd ..

    $ oc apply -f deploy/yamls
    deployment.apps/cfnodejsapp created
    imagestream.image.openshift.io/cfnodejsapp-latest created
    service/cfnodejsapp created
    secret/cfnodejsapp-vcapasenv configured
    ```

    Now our application is accessible on the cluster. You can check the status of pods by running the command mentioned below.

    ```console
    $ oc get pods

    NAME                           READY   STATUS    RESTARTS   AGE
    cfnodejsapp-7f9ccb98d7-b97xn   1/1     Running   0          7s
    cfnodejsapp-7f9ccb98d7-zhdnh   1/1     Running   0          7s
    ```

    Create a `Route` to access the application by the FQDN URL.

    ```console
    $ oc expose svc/cfnodejsapp
    route.route.openshift.io/cfnodejsapp exposed
    ```

    You can get the route URL using the following oc command.

    ```console
    $ oc get route
    NAME          HOST/PORT                                                    PATH   SERVICES      PORT        TERMINATION   WILDCARD
    cfnodejsapp   cfnodejsapp-doh-dev.apps.sandbox.x8i5.p1.openshiftapps.com          cfnodejsapp   port-8080                 None
    ```

    Access the application using the `curl` command line tool.

    ```console
    $ curl cfnodejsapp-doh-dev.apps.sandbox.x8i5.p1.openshiftapps.com
    <h1>This is a node server</h1>
    ```

    You can also find out if the application is deployed well in Topology view on the OpenShift cluster. Let's add a NodeJs label to the application pod using the following `oc` command:

    ```console
    $ oc label deployment/cfnodejsapp app.openshift.io/runtime=nodejs --overwrite 
    deployment.apps/cfnodejsapp labeled
    ```

    Go to the OpenShift web console. Then, navigate to the `Topology` view in _Develop perspective_. You will see the `NodeJs` icon as below. 

    ![devsandbox](../../assets/images/samples/dev-sandbox/topology.png)

    Click on `Open URL` icon. You will see the application page as below.

    ![devsandbox](../../assets/images/samples/dev-sandbox/route.png)

## Conclusion

So, that is a simple way where you were able to combine multiple information like runtime information, source information and cluster information, and do a holistic transformation of your Cloud Foundry app to Kubernetes using the Developer Sandbox for Red Hat OpenShift. Find more information about the developer sandbox [here](https://developers.redhat.com/developer-sandbox/get-started).
