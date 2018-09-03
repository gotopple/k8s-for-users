# Workload Resources and Workflows with Kubernetes

Most people know Kubernetes as a container or service orchestration and resiliency platform. These exercises will help you work through those specific Kubernetes features and the related resources. 

This morning you're going to learn with a classic Docker example application. The voting app uses a microservice architecture with four services written in different languages, and two databases (Redis and Postgres). Two services, `vote` and `result` serve web pages that allow visitors to vote and view polling results respectively. A third service is a Redis pub-sub worker that digests voting messages and updates a table in Postgres.

The following exercises will walk you through service deployment and life cycle management using Kubernetes resources and declarative workflows.

If you're not familiar with YAML then you might want to pull up this [YAML Primer](../YAML_Primer.md) in another tab.

## Pods and Services

Kubernetes is a database for modeling operations resources. Those resources fall into categories like "workload," "load balancing," "configuration," "storage," "cluster," and "metadata." These exercises introduce the Pod. A pod is the smallest schedulable workload resource in Kubernetes. Unlike other orchestrators and a Kubernetes pod can have one or more containers. 

All of the containers in a pod are configured such that they can interact like processes on a common machine. They share NET, IPC, UTS namespaces as well as a single loopback and network interface. Containers in a pod can all communicate with each other on localhost, and they compete for the same port ranges on the Ethernet interface. So, you won't be able to have two containers listen on port 80 within the same pod even if they are in different containers.

Pod resources specify discrete workloads, not templates. A single pod resource will be realized into a single set of the described containers. It does not make sense to talk about "scaling" a pod. You are certainly free to create a bunch of copies of a pod (by literally creating multiple pod definitions), but there are better ways to accomplish this.

### Exercise 1A: Explain Pod and Service

Before you can get started working with Pods you need to know about their structure and associated behaviors. You could pull up the Kubernetes API reference documentation but the documentation is also built into Kubernetes itself. The `explain` subcommand will describe any resource or resource property for any version of that resource supported by the current platform.

Run `kubectl explain pod` to see the top-level structure of a Pod. The `Status` property is special in Kubernetes and is read-only. When you're defining a Pod with YAML each of the listed properties (with the exception of Status) should be specified. You should explain the `pod.spec` and `pod.spec.containers` to see what properties are available.

Kubernetes defaults to showing the earliest version of a resource available on the platform. You must specify the `--api-version` flag to lookup any specific version of a resource.

Use `kubectl explain --api-version v1 pod`, `kubectl explain --api-version v1 pod.spec` and `kubectl explain --api-version v1 pod.spec.containers` to write a **v1 Pod** that runs Redis in a container. The Pod should be configured as follows:

* Pod name: `redis`
* Pod label key: `app` and value: `redis`
* Single container named `redis`, using the `redis:alpine` image.

### Exercise 1B: Introducing Namespaces

Before you launch your Pod take a moment to validate your answer against the provided solution. Compare your answer with an example look at `ex1/solution/redis-pod.yaml`. 

You might notice the inclusion of a metadata field that you likely did not include in your answer: `namespace`.

Kubernetes provides Namespaces to isolate collections of related resources. A Namespace is another resource type and is created, and otherwise managed the same way as any other Kubernetes resource. By default resources are created in the `default` Namespace. That is where most of your work today will be created. 

All resource names must be unique within any give Namespace. That is to say you cannot have two Pods named, `redis` in the same Namespace. So we want to keep the provided solutions separate from your workspace. To do so we've defined a new Namespace called, `workshop-day1-solutions`. You can see that definition in `ex1/solution/custom-ns.yaml`.

Before moving forward launch the provided solutions by running `kubectl apply -f ex1/solution`. You'll use this to verify your work in the next exercise.

### Exercise 2: Launch and a Redis Pod

Next you're going to start your Redis pod. The `kubectl apply` subcommand uses declarative tooling to analyze the delta between the state on the platform and the new desired state. You'll use this subcommand and specify the file containing your Pod definition. Do that now.

Once your Pod has been created you should be able to see it and any other Pods running on your platform. Do that now by running `kubectl get pod`. 

The output should look something like:

```
NAME      READY     STATUS    RESTARTS   AGE
redis     1/1       Running   0          2m
```

This is a list of the Pods running in the default Namespace. Note that 1/1 is in a Ready state. For reference you can compare that list with the Pods running in the `workshop-day1-solutions` Namespace by running `kubectl get po --namespace workshop-day1-solutions`.

So, redis is running "somewhere" but how do you access it? How do you know anything about it? You can get deep details about any Kubernetes resource (including your Pods) with the `kubectl describe` subcommand. Like other kubectl commands the first argument is always the resource type that you're inspecting and the remainder is a variadic list of names that you want to operate on. 

```
kubectl describe po redis
```

In the output from describe you'll see associated metadata, relationships with other resources, the set of associated containers, and an assigned IP address (this is the ClusterIP which we'll cover later). This is the IP address for the Pod. However you'll also note that it is a private network IP address that is not likely routable from your browser or terminal. If you want to test this redis instance you're going to need to get creative.

You might be familiar with Docker already and be used to the idea of execing a command in a running container. Well Kubernetes is built on top of Docker or other similar container technology. You can see the redis container in your Pod by running `docker ps`.

You could exec a command in the redis container, but in production you'd only be able to do that from the cluster node where the container is running. That won't scale well. Instead use the `kubectl exec` subcommand. It is similar in form and function but since Pods can contain multiple containers it does require a bit of refinement. Use the following command to define and increment a counter in your redis instance:

```
kubectl exec redis -i -t -- redis-cli incr my-training-counter
```

Repeat that command a few times to watch the counter increase.

By default the exec subcommand will use the first container defined in the Pod spec. If you need to exec on a different container you should specify both the Pod and container name:

```
kubectl exec redis -c redis -i -t -- redis-cli incr my-training-counter
```

If you need to work with a Pod in a different namespace you just need to specify that namespace:

```
kubectl exec redis -c redis -i -t --namespace workshop-day1-solutions -- redis-cli incr my-training-counter
```

When you're finished having fun with your redis instance tear down your Pod with the `kubectl delete` subcommand. Remember, you specify the resources to delete by pointing to the resource definition files (the `-f` flag).

### Exercise 3: Compose to Kubernetes Conversion with Pods and Services

Now lets start working with a real application. It is time to create resources to launch the voting application in Kubernetes. 

This software in this exercise involves multiple processes that use each other over the network. Since this example was originally designed for use in Docker you're going to need to use a few tricks to make it work. 

You're going to create two pods. For now it might be easier to put them both in the same file. To do so separate the resource definitions with the YAML document separator `---` on its own line. The two pods should be defined as follows:

##### Pod 1: Voter

Should be named `voter` and have labels `app: voter` and `version: v1`. The pod spec should include the following container definitions:

| Container name  | Port | Image |
| --------------- |:----:| -----:|
| redis | 6379 | `redis:alpine` |
| vote | 80 | `dockersamples/examplevotingapp_vote:before` |

When software in this pod goes to resolve the name, "redis" it should get "127.0.0.1" as the result.

##### Pod 2: Results

Should be named `results` and have labels `app: results` and `version: v1`. This time use a volume to isolate the database state. The pod spec should include the following volume definitions:

| Name  | EmptyDir |
| --------------- |:----:|
| db-data | `{}`` | 

And the following container definitions:

| Container name  | Port | VolumeMounts | Image |
| --------------- |:----:|:------------:| -----:|
| db | 5432 | `name: db-data` `mountpoint: /var/lib/postgresql/data` | `postgres:9.4` |
| worker | NA |  | `dockersamples/examplevotingapp_worker` |
| result | 80 |  | `dockersamples/examplevotingapp_result:before` |

When software in this pod goes to resolve the name, "db" it should get "127.0.0.1" as the result.

*Hint:* you might find a property on the pod.spec that allows you to manage custom host names in the pod.

The five services are split in this way because the vote and results container both have software that binds to port 80. Those applications cannot live in the same pod. The act of voting and result publishing is bound by an asynchronous data processing pipeline and in this case the redis instance is a synchronous dependency for the voting application, but not the results application. By splitting the architecture into these two pods we can be sure that each component will function correctly without the other.

After you've created the pod definitions you need to provide the service discovery glue so that they can talk to each other and so you can start voting.

##### Services

You should know that every pod gets an IP address, but pods are not automatically discoverable via DNS. Network service discovery and load balancing functionality are provided by Service resources. 

A Service resource is little more than a description of a reverse proxy. Kubernetes will publish specific DNS records for every Service resource, and realize proxies in according to the Service spec. Take a moment to checkout the documentation for the `service` resource and its `spec` property.

```
kubectl explain service
kubectl explain service.spec
```

Take note of three important `service.spec` fields: type, selector, and ports.

There are several Service types. Some types are for internal service definitions, others are for integration with broader systems like a cloud environment or managed DNS system. The default type is `ClusterIP` which creates an internal IP address that is serviced by a reverse-proxy component called, `kube-proxy.` ClusterIP provides lightweight internal network load balancing (layer 4).

Since you're running these examples on your local machine and a single-node Kubernetes cluster it will be easiest to use the `NodePort` service type for services that you want to access outside of Kubernetes. That type behaves similar to Docker port publishing and works well for local testing. 

Create Service resource definitions according to the following table:

| Service name  | Publish Port | Internal (y/n) | Selector |
| ------------- |:------------:|:--------------:|---------:|
| redis | 6379/TCP | y | `app: voter` |
| vote | 30000/TCP | n | `app: voter` |
| result | 30001/TCP | n | `app: results` |

Remember to separate your resource definitions with the YAML record separator, `---`.

Once you're done startup your app with the `kubectl apply` command.

##### Trying it out

Poll the resources a few times with `kubectl get po` and `kubectl get svc`. When all of the resources are ready you should be able to load up the voting and results apps in your web browser by navigating to [localhost:30000](http://localhost:30000) and [localhost:30001](http://localhost:30001) respectively. 

Votes cast in one window should be reflected in the other shortly.

If you're running into trouble you can checkout a solution under `ex3/solution` or just start up the solution stack with `kubectl apply -f ex3/solution`. That stack uses ports 31000 and 31001 respectively.

Don't forget to tear down the solutions when you're done with `kubectl delete`.

## Resilience and Automated Deployment Control 

Pods are fine, but working with them is a bit like working with raw containers in Docker. They provide a great language for describing workloads, but they do little to help us abstract workflows. Upgrading a service using raw Pod resources would involve manually creating new Pod versions, updating some label like `version` and then updating Service selectors to route to the new and old pods, and then manually take down old versions. Yuck. Processes like that are formulaic and should be automated whenever possible. Leaving them un-automated only introduces opportunities for human error. Kubernetes provides those automations (and many more powerful ones) but adopting them means working with higher-level resource types. 

The next resource you'll work with is called a ReplicaSet. 

### Exercise 4: Resiliency with ReplicaSets

ReplicaSets provide replica and resilience controls. They create Pod resources on your behalf according to the template and other properties you provide in the ReplicaSet spec. If a Pod under management of a ReplicaSet is deleted or fails then the ReplicaSet will take care of replacing it and maintaining the realized desired state of the system.

**Note**: there is another resource type called a ReplicationController that is deprecated. New users and workloads should use ReplicaSets or other higher-level resources.

Now create a new file to describe your stack in terms of ReplicaSets and Services instead of raw Pod resources. Call the new file `voting-rs.yaml`. Include all of the ReplicaSet and Service definitions in that same file.

The ReplicaSet resource type is defined in the `apps/v1` API version. The `apps` API tree is full of higher-level abstractions.

You're going to want to create two ReplicaSets, one for each type of Pod in the previous exercise. Use `kubectl explain replicaset --api-version apps/v1` and `kubectl explain replicaset.spec --api-version apps/v1` to get the structure.

Once you're finished with your new resource definitions start your resources with the `kubectl apply` subcommand. If you get stuck or want to check your solution you can find the example solution under `ex4/solution`. You can launch that solution by running `kubectl apply -f ex4/solution`

When they've reached a full ready state open the voting application in your web browser: [localhost:30000](http://localhost:30000). Cast a vote and note the "container ID" at the bottom of the page. This is the hostname of the pod where the request was served. Note that you did not specify that name anywhere. It was created by the ReplicaSet. Go lookup the running Pod and ReplicaSet resources.

```
kubectl get rs
kubectl get po
```

Use `kubectl describe rs` to see the list of events associated with the voter ReplicaSet. You should see a line like near the bottom:

```
Normal  SuccessfulCreate  1m    replicaset-controller  Created pod: voter-f9599
```

Events are just resources in Kubernetes. This is a list of all the events created by the replicaset-controller component in relation to this ReplicaSet resource. Now put the ReplicaSet resilience to the test. Delete the pod that served the last request. Using the Pod name from the event above you'd issue a command like, `kubectl delete po voter-f9599`.

Deleting the Pod is one type of failure from the perspective of the replicaset-controller. Since the ReplicaSet resource remains in the system and now the realized state is our of spec from the desired state it should create a replacement Pod immediately.

You should describe the ReplicaSet resource again and examine the Event list. You'll see that there is a new event describing the creation of a new Pod. If you then reload the voting page you'll notice that the app still works and is being served by the replacement Pod.

You've witnessed both the resilience and dynamic service backend routing mechanisms provided by Kubernetes controllers. Before moving on, take a moment to vote Cats vs Dogs. Then load the results page [localhost:30001](http://localhost:30001) and verify that the vote was recorded.

The problem with containers, Pods, ReplicaSets, etc is that they are primarily concerned with runtime context and consistent disposable processes. Persistent state needs to be modeled. The results Pod uses a volume, but a Kubernetes volume is scoped to a Pod. Volumes are cleaned up when Pods are cleaned up and they are not shared across Pods. Watch what happens when you delete your results Pod. Do that now and reload the results page. You should note the vote goes back to 50/50.

Pods are replaced all the time as part of natural life cycle events like deployments or cluster node replacement. Before you learn how to deal with persistent state you should be familiar with some life cycle workflows.

### Exercise 5: A Life Cycle Adventure

Performing a rolling deployment by hand on any platform usually goes something like:

* Verify that some minimum percentage of the service replicas are in a healthy state
* Identify one or some batch of replicas to replace
* Remove those replicas from the reverse proxy backend pool
* Stop those replicas
* Start a new replica of with the updated configuration for each stopped replica
* Verify that each new replica enters a running or ready state
* Add the new replias to the reverse proxy backend pool
* Monitor service health for some period, repeat from step 1 until all replicas have been replaced
* Mark the deployment as complete and clean up old replicas

ReplicaSets maintain a desired number of Pods, but don't provide any deployment workflow. To get an appreciation for that workflow you're going to deploy an update to your ReplicaSet definitions from the last example.

Product management issued you a ticket and they want to roll out a new poll. They'd like to stop asking people about Cats vs. Dogs and focus more on the enterprise software crowd. They want to poll Java vs. .NET. Luckily, that software has already been written and is available simply by updating the image tag.

**With your solution from exercise 4 running**, update your definition file and change the images used for the voter and results containers from the `:before` tag to the `:after` tag. When you've updated both Pod definitions you're going to trigger a deployment using the `kubectl apply` command.

Once the changes have been applied reload [localhost:30000](http://localhost:30000) and [localhost:30001](http://localhost:30001). Did you see where the new poll came up? No? What if you refresh again? Maybe wait a few seconds and try again. No? You might notice that the vote page is being served by the same Pod as before. It is almost like no changes were made to the running software.

Well if you inspect the ReplicaSet configurations with `kubectl describe` you'll note that the Pod template does specify the `:after` image. But that there is only a single Pod creation event in the history. It looks like you'll need to manually trigger the replacement. 

Delete the Pod that was created by the voter ReplicaSet, check the description again and verify that a new Pod was created and reload the page. Viola. In this case the Service definition you used for the voter and redis services only select backends based on the `app` label. As updated version come into service they are automatically included in the Service definition so no manual proxy adjustment was required.

Delete the result Pod to flip that side of the app as well.

This was a dangerous flip operation. You could have done a more seamless migration here by creating new ReplicaSet definitions with an increased `version` label and carefully adjusting the `replias` property on each. But either way manipulating these resources manually is error prone.

**Before moving on** cleanup all of the pods, replica sets, and services that you created for this exercise.

### Exercise 6: Deployments

Kubernetes provides another resource called a `Deployment`. Deployments describe a workload (a pod template and a desired number of replicas) as well as parameters for performing the actual deployment and rollback orchestration.

You can use Deployment resources to describe rolling updates, or recreate deployments and tune the typical parameters associated with those deployment methods. See details by running `kubectl explain --api-version apps/v1 deployment.spec`, and `kubectl explain --api-version apps/v1 deployment.spec.strategy`.

Instead of writing your own Deployment spec take a moment to examine the provided solutions under `ex6/v1`. Consider the new `voter` Deployment resource.

```
apiVersion: apps/v1

# Note that this is a Deployment resource, but that the spec only
# includes the desired replica count and pod template. No deployment
# options are included. This Deployment reosurce will use the defaults.
kind: Deployment
metadata:
  name: voter
  namespace: workshop-day1-solutions
spec:
  # The described pods are stateful and only route votes to the
  # pod-local redis instance. Running multiple replicas would
  # create a federated set with no linkage.
  replicas: 1
  selector:
    matchLabels:
      app: voter
  template:
    metadata:
      labels:
        app: voter
        version: v1
    spec:
      # This is a nice little hack that will tell the applications
      # in this pod to look to localhost when resolving  the redis
      # or vote names (instead of doing a full DNS lookup).
      hostAliases:
       - ip: "127.0.0.1"
         hostnames:
          - "vote"
          - "redis"
      containers:
       - name: vote
         image: dockersamples/examplevotingapp_vote:before
         ports:
          - containerPort: 80   # purely informational
       - name: redis
         image: redis:alpine
         ports:
          - containerPort: 6379 # purely informational
```

Create the logical service definitions, and then apply the resource definitions under `v1`:

```
kubectl apply -f ./ex6/services.yaml
kubectl apply -f ./ex6/v1
```

Specifying the namespace for every command is going to be painful. Instead change your kubectl config to set the current namespace to `workshop-day1-solutions` with the following command (if you're not using Docker for Desktop then you'll need to lookup your current context name - see the instructor):

```
kubectl config set-context docker-for-desktop --namespace workshop-day1-solutions
```

Once you've launched the stack take a moment to examine what you've launched. Get the Deployment resources and describe one of them:

```
kubectl get deploy
kubectl describe deploy voter # Describe the voter Deployment
```

The description data for the `voter` Deployment will have a section that describes the current replica status and rollout configuration. 

```
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
```

Deployment resources are watched by a controller that manages ReplicaSet resources. Those ReplicaSets are watched by the ReplicaSet Controller who manages Pods. You can list and describe those ReplicaSet and Pod resources directly:

```
kubectl get rs
kubectl get po
```

With the application running [go vote](http://localhost:31000) and [view the results](http://localhost:31001). The running poll should be the original version: Cats v Dogs.

Now, to perform an update using Deployment resources we need only change the Deployment resource definitions. Inspect the updated definitions under `./ex6/v2` and apply the changes: `kubectl apply -f ./ex6/v2`

The apply command will determine which resources have changed and update the resources appropriately. The Kubernetes controller responsible for watching Deployment resources will see that those resources have changed and execute the deployment workflow as configured.

At this point you should be able to refresh [the vote page](http://localhost:31000) and [the results page](http://localhost:31001) as see that the poll has switched to Java v .NET (if it hasn't yet give it a few moments and refresh again). 

If you describe the `voter` Deployment again you'll be able to see the events that were triggered by the change.

```
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  52s   deployment-controller  Scaled up replica set voter-cf8cc565b to 1
  Normal  ScalingReplicaSet  45s   deployment-controller  Scaled down replica set voter-79dbcf7549 to 0
```

The deployment-controller created a new ReplicaSet with an updated Pod template then scaled that up to 1 replica, and after it became ready, scaled down the old ReplicaSet.

This is much easier than doing all of this by hand. The Kubernetes extensibility model makes it possible to build all sort of higher-level abstractions to simplify operational procedures. Entire frameworks (like the Operator framework) exist to make building those controllers simple.

But despite this enhanced operational experience, you might have noticed that your previous votes were lost during the deployment. The system state is still scoped to a specific Pod and the database is deployed in the same Pod as the front-end applications.

Take this opportunity to refactor the Deployment based solution in `./ex6/v2` into three Deployment resources. 

| Deployment Name | Migrated Pods |
|-----------------|---------------|
| voting-result | voting-result |
| voter | voter |
| data-pipeline | worker, redis, db |

**Hints**:

* Make sure to remove the `hostAliases` from the `voter` and `voting-result` Pod templates.
* The data-pipeline Pod will still need `hostAliases` for `redis` and `db` so that the `worker` container can discover those inside the Pod.
* In the old architecture the `db` container was co-located with the `worker` and `results` containers so it never needed to be exposed as a Service. You'll need to build a new Service so that the application in the `results` container can reach it.
* Make sure to put your new resource definitions in the same namespace as the old ones, `workshop-day1-solutions`.

When you're ready, compare your work with the solutions under `./ex6/v3`. The `data.yaml` file will be particularly interesting. This solution includes Service resource definitions in the same files as the associated Deployment resources.

When you're ready, rollout the new changes by running: `kubectl apply -f ./ex6/v3`. And checkout the Deployment events: `kubectl describe deploy voter`. You should see and event list like the following:

```
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  19m   deployment-controller  Scaled up replica set voter-79dbcf7549 to 1
  Normal  ScalingReplicaSet  17m   deployment-controller  Scaled up replica set voter-cf8cc565b to 1
  Normal  ScalingReplicaSet  17m   deployment-controller  Scaled down replica set voter-79dbcf7549 to 0
  Normal  ScalingReplicaSet  15m   deployment-controller  Scaled up replica set voter-659db78f67 to 1
  Normal  ScalingReplicaSet  15m   deployment-controller  Scaled down replica set voter-cf8cc565b to 0
```

Again, you can see the deployment-controller orchestrating the ReplicaSet changes. Now you should be able to independently change the `voting-results` and `voter` Deployment resources without losing state in the `data-pipeline` Pods. This is ofcourse a fragile configuration. If the data-pipeline Pod is lost then the `emptyDir` volumes (and the state they contain) will be lost as well. You'll learn to work with Persistent State in tomorrow's session.

