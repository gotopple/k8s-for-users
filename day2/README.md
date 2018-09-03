# Day 2: Jobs, Persistent State, Configuration, Secrets, and Service Accounts

## Jobs, ConfigMaps, and Secrets

Kubernetes is able to orchestrate all sort of resource types. These include batch jobs. The next few exercises will use Jobs to get familiar with advanced configuration resources including ConfigMaps and Secrets.

### Exercise 1: Launch a Traceroute Batch Job

Create a Job resource that will use the `alpine:3.8` image to run `traceroute google.com`. Then use the `kubectl logs -f` command to follow the logs of the resulting Pod. Like other resources that result in the creation of Pods, the bulk of the work is in defining the attributes under `job.spec.template.spec`, but if you wanted to configure characteristics of the Job behavior you'd set one of the properties on `job.spec`.

The default Pod restartPolicy `Always` is not valid for Jobs. All Jobs should terminate so choose between `Never` and `OnFailure`.

After you've launched your traceroute Job you can check its status using `describe` alternatively `get jobs` will state if the job was successful. A successful job will not be restarted and its Pod will not be automatically cleaned up. Jobs with multiple Pods that execute in sequence or in parallel will all be listed separately.

### Exercise 2: Building a Set of Jobs that use Shared Configuration

The only thing 

Create a ConfigMap resource to contain the shared configuration:

| Name | Data: target | Data: ttw | Data: max-hops |
|:-----|:-------------|:----------|:---------------|
|target-google|google.com|"4"|"15"|

Create 2 Job resources that consume that ConfigMap. Configuring a container to consume ConfigMap resources as environment variables requires the use of the `pod.spec.containers.env.valueFrom` property on each `pod.spec.containers.env` sequence entry. The containers in these Jobs require the `target`, `ttw`, and `max-hops` keys in the ConfigMap to be used in environment variables named, `TARGET`, `TTW`, and `MAX-HOPS` respectively. 

The rest of the Pod spec should be configured using these details:

| Name | Image | Command |
|:-----|:-------------|:----------|
|ping-google|alpine:3.8|`["/bin/sh", "-c", "ping -W $(TTW) -w 30 $(TARGET)"]`|
|traceroute-google|alpine:3.8|`["/bin/sh", "-c", "traceroute -w $(TTW) -m $(MAX_HOPS) $(TARGET)"]`|

If you get stuck, there are several examples under `./ex2/solution`.

When you're ready to run your new Jobs use `kubectl apply` to create all of the resources, then use `kubectl get jobs`, `kubectl get pods`, and `kubectl logs <POD_NAME>` to see the results. When the Jobs are done they will move to a completed state.

### Exercise 3: Using Secret Data



## Managing Persistent State



## Service Accounts

