# eks-webinar-demo

1. Deploy the Quickstart (will take approx. 30 minutes - recording demo to edit out some of that time may help)
1. While that is happening explain how the quickstart and CDK/EKS work
1. Then show an example of containerising a Spring Boot app
1. Then show how to make the Kuberntes manifests for the app
1. Then show deploying it to the cluster and it getting a real ALB and HTTPS w/cert and proper public DNS name

## Deploy the Quickstart

Prereqs:
1. Fork https://github.com/jasonumiker/eks-quickstart
1. Generate a personal access token on GitHub - https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token
1. Edit cluster-codebuild/EKSCodeBuildStack.template.json to change Location to your GitHub repo/path
1. Run aws codebuild import-source-credentials --server-type GITHUB --auth-type PERSONAL_ACCESS_TOKEN --token <token_value> to provide your token to CodeBuild
1. Deploy cluster-codebuild/EKSCodeBuildStack.template.json via the AWS CloudFormation Console in the region you'd like the cluster

During the Demo:
1. Go to the CodeBuild console, click on the Build project that starts with EKSCodeBuild, and then click the Start build button.
1. Explain that merging changes into the right path on the right branch will trigger a build automatically - which is GitOps for the cluster
1. Show the `buildspec.yml` to explain what CodeBuild is doing
1. Go into the build and tail the log showing that `cdk deploy` is happening

## Explain the CDK and what is happening

1. Open the eks_cluster.py file in VS Code
1. Show that the parameters have been exposed as strings and booleans up top
1. Show how it can bridge the gap between AWS and Kubernetes with the managed Elasticsearch, IAM/IRSA & fluent-bit Helm chart

## Local Docker Demo - Containerising Spring Boot App

This will build a spring boot demo app into a container

Prereqs:
* Do the `docker build -t spring .` before the demo to pre-cache the maven step which takes quite awhile
* Run `docker build -t root-in-docker-vm .` in root-in-docker-vm folder


* Show the contents of `~/eks-webinar-demo/top-spring-boot-docker/demo/src/main/java/com/example/demo/DemoApplication.java`
    * Explain that we're about to go from source code to container - building this code within Docker
* `cd ~/eks-webinar-demo/top-spring-boot-docker/demo`
* Show the Dockerfile in an editor and explain what it's doing
* `docker build -t spring .`
    * Explain that we just did the build with no maven/JDK required on machine (or can support many different versions on the same machine easily)!
    * Explain that the maven build also caches all the maven stuff that was pulled down so subsequent builds are fast like you just saw
        * Show `spring-build-time-differences.txt` for the time difference (0s instead of 408s)
* `docker run --rm -p 8080:8080 --name spring spring`
    * Show the container running on our laptop - go to http://localhost:8080
    * ctrl-c to exit (which deletes the container due to our --rm)


## Local Kubernetes Demo

This will show making the Kubernetes manifest files and running the app on the laptop using the built-in Kubernetes that is part of Docker Desktop

We do this while we're waiting for our EKS cluster and add-ons to come up:

* `kubectl create deployment spring --image=spring`
    * Explain how this command created us a deployment rather than us having to type the YAML ourselves
* `kubectl get replicasets`
    * Explain how Deployments manage ReplicaSets which manage Pods
* `kubectl get pods`
    * Explain how this Pod can't pull the image because it is just on our laptop and we have not put it in a registry yet
* `kubectl edit deployment spring`
    * Change the `imagePullPolicy` to `Never` - and it'll then use our local image
    * `kubectl get pods` again to show it worked!
* Explain our service is running but we need to create a Service to access it over the network
    * `kubectl expose deployment spring --port 8080 --target-port 8080 --type NodePort`
    * `kubectl get services`
    * Go to `http://localhost:<nodeport>` you see in the output
* Now let's get the YAML files ready for deploying to our EKS and maybe storing in our git repo:
    * `kubectl create deployment spring --image=spring --dry-run=client -o yaml`
    * `kubectl create deployment spring --image=spring --dry-run=client -o yaml > spring-deployment.yaml`
    * `kubectl expose deployment spring --port 8080 --target-port 8080 --dry-run=client -o yaml`
    * `kubectl expose deployment spring --port 8080 --target-port 8080 --dry-run=client -o yaml > spring-service.yaml`

## Pushing our image up to ECR

While in production we'd have a CodeBuild build this and push it to ECR here we'll show manually creating an ECR repo and pusing it up and then referencing it in our EKS manifests.

TODO: Add steps here

## Deploying our image to EKS

Cut back to our EKS cluster being up and running.

* Edit `spring-deployment.yaml` and put in the ECR repo for the image
* `kubectl apply -f spring-deployment.yaml`
* `kubectl apply -f spring-service.yaml`
* Explain that to expose the service via an ALB we'll use a Kubernetes Ingress
    * You can't use kubectl to create these but we've prepared one at `spring-ingress-aws.yaml`
    * Show/explain the contents of that file
    * As part of that reiterate we've set up the ALB LB Controller and External DNS 
* `kubectl apply -f spring-ingress-aws.yaml`
* Give the ALB a couple minutes to come up (edit this out) then go to https://spring.jasonumiker.com and show it all working