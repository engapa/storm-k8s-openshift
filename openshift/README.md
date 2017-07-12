# storm-openshift

Follow these steps in order to deploy a storm cluster within openshift

First of all you must decide the openshift project to deploy the cluster into.

If your openshift account has the admin role or self-provisioner then you will be able to create one:

```sh
oc login -u <your_user> -p <your_account> <openshift_endpoint>
oc new-project <project_name> \
    --description="<description>" --display-name="<display_name>"
oc project <new_project>
```

> NOTE: This is an optional step, always you could use an existing project.
If you haven't any of these roles then contact openshift cluster administrator to request one project.

It isn't required to clone this repository, use this env variable to use the https url instead of local files:

```sh
export STORM_PREFIX_RESOURCES="https://raw.githubusercontent.com/engapa/storm-k8s-openshift/master/openshift/"
```

## Zookeeper

Follow instructions at [here](https://github.com/engapa/zookeeper-k8s-openshift/tree/master/openshift) to get a required Zookeeper installation.

```sh
$ oc process -f https://raw.githubusercontent.com/engapa/zookeeper-k8s-openshift/master/openshift/zookeeper.yaml \
 -p ZOO_REPLICAS=2 | oc create -f -
service "zk-svc" created
statefulset "zk" created
```

Now we have a zookeeper cluster ready for storm. The zookeeper nodes will be available through this internal hostname list: `zk-0.zk-svc, zk-1.zk-svc`

## Using pacemaker

Pacemaker is a storm daemon designed to process heartbeats from workers.
This deployment is optional but is a good idea to use it to keep in memory the worker heartbeats instead of write them on disk.

You have more details at: http://storm.apache.org/releases/1.1.0/Pacemaker.html

Create one/many storm pacemaker pod/s:

```sh
$ oc process -f ${STORM_PREFIX_REOURCES}storm-pacemaker.yaml \
  -p NAME=storm-pacemaker \
  -p PORT=6699
  -p REPLICAS=1 | oc create -f -
configmap "storm-pacemaker-config" created
service "storm-pacemaker" created
statefulset "storm-pacemaker" created
```

When you are going to create masters (nimbus) and workers (supervisor),
you should add this section in suitable ConfigMaps (variable CONFIG_FILE_CONTENS,
by default these lines are commented):

```yaml
pacemaker.servers: ["<NAME>-<i>.<NAME>.<PROJECT>.svc.cluster.local"]
pacemaker.port: 6699
storm.cluster.state.store: "org.apache.storm.pacemaker.pacemaker_state_factory"
```

where each server name in `pacemaker.servers` variable follows this format:
`<NAME>-<i>.<NAME>.<PROJECT>.svc.cluster.local`,
where:\
-**NAME**: The value of the NAME param. In this example would be 'storm-pacemaker-0.storm-pacemaker.storm.svc.cluster.local'\
-**i** : The ordinal or index of the node. In this case we have 1 replicas, the unique value is 0.\
-**PROJECT**: The name of the project you are. In this case 'storm'.

## Storm masters

The masters or nimbus processes of the cluster can be created by typing this command:

```sh
$ oc process -f ${STORM_PREFIX_REOURCES}storm-nimbus.yaml \
  -p NAME=storm-nimbus \
  -p REPLICAS=2 \
  -p ZK_SERVERS="\"zk-0.zk\", \"zk-1.zk\"" | oc create -f -
configmap "storm-nimbus-config" created
service "storm-nimbus" created
statefulset "storm-nimbus" created
```

The complete hostname for each node is:
`<NAME>-<i>.<NAME>.<PROJECT>.svc.cluster.local`,
where:\
-**NAME**: The value of the NAME param. In this example is 'storm-nimbus'\
-**i** : The ordinal or index of the node. In this case we have 2 replicas, values are [0,1]\
-**PROJECT**: The name of the project you are. In this case 'storm'.

## Storm workers

```sh
$ oc process -f ${STORM_PREFIX_REOURCES}storm-worker.yaml \
  -p NAME=storm-worker \
  -p NIMBUS_SEEDS="\"storm-nimbus-0.storm-nimbus.storm.svc.cluster.local\", \"storm-nimbus-1.storm-nimbus.storm.svc.cluster.local\"" \
  -p ZK_SERVERS="\"zk-0.zk-svc\", \"zk-1.zk-svc\"" | oc create -f -
configmap "storm-worker-config" created
service "storm-worker" created
deploymentconfig "storm-worker" created
horizontalpodautoscaler "storm-worker" created
```

> NOTE: By default the MIN_REPLICAS and MAX_REPLICAS parameter values are `1`, change these values for other number of replicas.
If you want to manage the number of replicas every time, note that these values must be equal, in other case the horizontal pod autoscaler takes the control
 (change the value of CPU_SCALE_TARGET if needed, default is 80 percent).

## Storm UI

```sh
$ oc process -f ${STORM_PREFIX_REOURCES}storm-monitor.yaml \
  -p NAME=storm-monitor \
  -p NIMBUS_SEEDS="\"storm-nimbus-0.storm-nimbus.storm.svc.cluster.local\", \"storm-nimbus-1.storm-nimbus.storm.svc.cluster.local\"" \
  -p ROUTE_SERVICE_DOMAIN=svc.domain.com | oc create -f -
configmap "storm-monitor-config" created
service "storm-monitor" created
deploymentconfig "storm-monitor" created
route "storm-monitor-ui" created
```

-**NIMBUS_SEEDS**: The nimbus seed list, check the number of replicas and their complete hostnames in the bellow above section.\
-**ROUTE_SERVICE_DOMAIN**: This is the service domain of the openshift router, the public domain for services.


## Customizing configuration

Each resource of the storm cluster has a config map which is written in a file on a pod volume,
 and the volume is mounted in a container directory.

```sh
$ oc get cm
NAME                   DATA      AGE
storm-monitor-config   1         3m
storm-nimbus-config    1         6m
storm-worker-config    1         2m
```

These config maps contains a key 'storm.yaml' which is used for all
storm processes launched inside all containers.

To edit contents of the config map just type:

```sh
$ oc edit cm storm-monitor-config
...
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
data:
  storm.yaml: |
    # These properties will be generated automatically by provided params
    # storm.zookeeper.servers
    # nimbus.thrift.port
    # nimbus.seeds
    # ui.port
    # logviewer.port
    # storm.log.dir
    # storm.local.dir
    # Add any other properties here
    ui.host: 0.0.0.0
    ui.childopts: "-Xmx768m"
    ui.actions.enabled: true
    --> type your properties or changes here <--
...
```

Be careful, if another container is created you would have containers with different configuration and the cluster may be no consistent.

When a change in config maps occurs the containers aren't recreated by themselves so you must do it manually.
Note that all containers are managed by DeploymentConfig and StatefulSets,
this means that if any container is removed then it will be recreated automatically.

Contents for these files could have been changed in provision time, the suitable parameter is `CONFIG_FILE_CONTENS`.
Please note in these case that we notify you generated properties from param values.

## Launch a topology

### Manual mode

Enter into nimbus leader or worker container and launch an example :

```sh
$ oc rsh ui
$ storm --config /conf/storm.yaml jar examples.jar
```

### By template

The idea is to use a deploy job which launches the topology by providing the nested parameter.

> **TODO**: In progress

## Cleanup

Remove all resources:

```sh
$ oc delete all,statefulset,pvc -l app=<ZK_NAME>
$ oc delete all,statefulset,cm -l app=<STORM_NIMBUS_NAME>
$ oc delete all,dc,cm -l app=<STORM_MONITOR_NAME>
$ oc delete all,dc,cm,hpa -l app=<STORM_WORKER_NAME>
```

And optionally:
```sh
$ oc delete template -l template=storm-nimbus
$ oc delete template -l template=storm-monitor
$ oc delete template -l template=storm-worker
$ oc delete project storm
```

