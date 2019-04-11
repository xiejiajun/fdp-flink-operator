# Flink-operator


`CRD`-based approach for managing Flink clusters in Kubernetes and OpenShift.

This operator uses [abstract-operator](https://github.com/jvm-operators/abstract-operator) library.

##Building
The operator has 2 participating projects:
* model - json definition of the model and scala code to build java classes (the reason is that SBT does not have an equivalent to [jsonschema2pojo-maven-plugin](https://github.com/joelittlejohn/jsonschema2pojo) so it is implemented as a separate project)
* operator - the actual implementation of the operator

Before building you need to generate types by running [GenerateMOdel](model/src/main/scala/com/lightbend/operator/model/GenerateModel.scala)
Then make sure that [generated classes](model/target/generated-sources/jsonschema2pojo) are included in your code sources

NOw build can be done running command:
````
 sbt docker 
````
Docker build here leverages base image, that can be build using the following [docker file](./Dockerfile)

##Installation
To install operator use [Helm](helm/flink-operator) 

The following configurations is available for operator:
* Operator image information including repository - operator docker name (default - lightbend/fdp-flink-operator); tag - operator docker tag (default - 0.0.1) and pullPolicy - operator docker pull policy (default - always)
* Namespace to watch - three options supported are - empty list - namespace where the operator is installed; explicit list of namespaces, “*” - all namespace (default - “*”)
* ReconciliationInterval - how often (in seconds) the full reconciliation should be run (default is 180)
* Metrics - a boolean defining whether operator metrics is exposed to Prometheus (default - true)
* MetricsPort - port used by metrics http server (default - 8080)
* InternalJvmMetrics - a boolean defining whether operator's internal JVM metrics is available through Prometheus (default - true)
* Operator's resource requirements including memory requirement for an operator (default - 512Mi); cpu requirement for an operator (default - 1000m)

##Basic commands
To create a cluster run something similar to 
````
cat <<EOF | kubectl create -f -
apiVersion: lightbend.com/v1
kind: FlinkCluster
metadata:
  name: my-cluster
spec:
  flinkConfiguration:
    num_taskmanagers: "2"
    taskmanagers_slots: "2"
EOF
````
Additional parameters can be added. See [example](yaml/cluster_complete.yaml)

By default a Flink [session cluster](https://ci.apache.org/projects/flink/flink-docs-stable/ops/deployment/kubernetes.html#flink-session-cluster-on-kubernetes) will be created (default argument *taskmanager* will be generated in this case.
Alternatively you can explicitly specify *taskmanager* and any additional arguments in the master inputs.

If you want to run Flink [job cluster](https://ci.apache.org/projects/flink/flink-docs-stable/ops/deployment/kubernetes.html#flink-job-cluster-on-kubernetes) specify
*jobcluster* cluster as an input followed by the name of main class for a job and list of parameters. 
When using job cluster you can additionally specify the following [parameters](https://github.com/apache/flink/tree/release-1.7/flink-container/docker#deploying-via-docker-compose):
* PARALLELISM - Default parallelism with which to start the job (default: 1), for example *--parallelism <parallelism>* 
* SAVEPOINT_OPTIONS - Savepoint options to start the cluster with (default: none), for example *--fromSavepoint <SAVEPOINT_PATH> --allowNonRestoredState* 

For more information on parallelism and savepoint options contact [documentation](https://ci.apache.org/projects/flink/flink-docs-stable/ops/cli.html#usage)

**Note** *An operator assume that custom images are build using [this project](https://github.com/lightbend/fdp-flink-build). If you build your images differently, the commands for running applications will change* 

##Seeing what is running

To see running clusters run 
````
oc get FlinkCluster
NAME         AGE
my-cluster   13m
```` 

To get the information about specific cluster, run
````
oc describe FlinkCluster my-cluster
Name:         my-cluster
Namespace:    flink
Labels:       <none>
Annotations:  <none>
API Version:  lightbend.com/v1
Kind:         FlinkCluster
Metadata:
  Cluster Name:        
  Creation Timestamp:  2019-03-20T19:00:29Z
  Generation:          1
  Resource Version:    12312782
  Self Link:           /apis/lightbend.com/v1/namespaces/flink/flinkclusters/my-cluster
  UID:                 6e16a9f4-4b42-11e9-bb33-0643529e7baa
Spec:
  Flink Configuration:
    Num _ Taskmanagers:    2
    Taskmanagers _ Slots:  2
Events:                    <none>
````
You can also get information about all running clusters running the following:
````
oc describe FlinkCluster
Name:         my-cluster
Namespace:    flink
Labels:       <none>
Annotations:  <none>
API Version:  lightbend.com/v1
Kind:         FlinkCluster
Metadata:
  Cluster Name:        
  Creation Timestamp:  2019-03-20T19:00:29Z
  Generation:          1
  Resource Version:    12312782
  Self Link:           /apis/lightbend.com/v1/namespaces/flink/flinkclusters/my-cluster
  UID:                 6e16a9f4-4b42-11e9-bb33-0643529e7baa
Spec:
  Flink Configuration:
    Num _ Taskmanagers:    2
    Taskmanagers _ Slots:  2
Events:                    <none>
````
To modify the cluster run the following:
````
cat <<EOF | kubectl replace -f -
> apiVersion: lightbend.com/v1
> kind: FlinkCluster
> metadata:
>   name: my-cluster
> spec:
>   flinkConfiguration:
>     num_taskmanagers: "3"
>     taskmanagers_slots: "2"
> EOF
````

To delete the cluster run the following:
````
oc delete FlinkCluster my-cluster
````

**Note** *The above CRD commands are not global, They only show the resources in a namespace that you are in.* 

##Metrics
Prometheus support is enabled via Helm chart
To see all available metrics, go to Prometheus console/graph and enter the following query
````
{app_kubernetes_io_name="flink-operator"}
````
This will return the list of all metrics produced by the operator.
You should also be able to see operator and created clusters in the lightbend console
