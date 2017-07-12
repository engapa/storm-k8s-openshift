# storm-k8s-openshift

[Storm](http://storm.apache.org/) on kubernetes and openshift

The main cluster components are:

- zookeeper: the cluster requires a zookeeper in order to manage the states of the different parts. Take a look at [zookeeper project]()
- storm: nimbus (master), supervisors (workers), logviewer, ui, ...

## kubernetes and/or openshift
These are the resources which are going to be deployed:

- ConfigMap: The common way to provide the configuration of each process of our storm cluster is by supplying a ConfigMap.
The keys of this ConfigMap are the name of the configuration files, and values are the content of those files.
- StatefulSet: A special deployment with permanent identity of each replica.
- DeploymentConfig (openshift) or Deployment (kubernetes): Define how to deploy pods.
- Horizontal Pod Autoscaler: Change the number of replicas according to CPU usage.

### kubernetes

If you are thinking about a storm deployment on kubernetes [these resources could help you](kubernetes).

### openshift

In this case the main resources are [openshift templates](openshift)

## docker-compose

If you just want to use docker compose we've left some resources in [compose directory](compose)