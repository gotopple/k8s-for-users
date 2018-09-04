# Day 2: Jobs, Persistent State, Configuration, Secrets, and Service Accounts

## Jobs, ConfigMaps, and Secrets

Kubernetes is able to orchestrate all sort of resource types. These include batch jobs. The next few exercises will use Jobs and CronJobs to get familiar with advanced configuration resources including ConfigMaps and Secrets.

Jobs and CronJobs are part of the Kubernetes `batch` API. They describe single or replicated workloads that should be run once and to completion. CronJobs describe a Job Template from which Jobs will be created on some specified schedule.

Like other resources that result in the creation of Pods, the bulk of the effort to work with Jobs is in defining the attributes under `job.spec.template.spec`, but if you wanted to configure characteristics of the Job behavior you'd set one of the properties on `job.spec`.

### Exercise 1: Launch a Traceroute Batch Job

Create a Job resource that will use the `alpine:3.8` image to run `traceroute google.com`. Then use the `kubectl logs -f` command to follow the logs of the resulting Pod. 

The current API version for Job is `batch/v1`.

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

If you get stuck, there are several related examples under `./ex2/solution`.

When you're ready to run your new Jobs use `kubectl apply` to create all of the resources, then use `kubectl get jobs`, `kubectl get pods`, and `kubectl logs <POD_NAME>` to see the results. When the Jobs are done they will move to a completed state.

### Exercise 3: Getting ConfigMap Data in a Volume

Kubernetes uses the term Volume to describe just about everything that can be mounted into a container file system tree. The `emptyDir` type that you used yesterday creates an empty directory on the cluster node and mounts that into the Pod at the specified point. That type shares the Pod life cycle. 

In this exercise use the `configMap` field on a volume spec to mount ConfigMap data into a container.

Create a ConfigMap resource named `not-secret-config` with three keys in data:

| Key | Value |
|:-----|:-------------:|
|username|Jeff|
|password|TeachesThisClass|
|endpoint|http://localhost:8080/login|

Create a Job named, `volume-mount-explorer` with a single container named `explorer` that uses the `alpine:3.8` image and the command, `["/bin/sh", "-c", "ls -al /etc/not-secret; cat /etc/not-secret/username"]`. The container should create a `volumeMount` that mounts a volume named `not-secret-config` at `/etc/not-secret`. The Pod `volumes` field should list a single volume that references the `not-secret-config` ConfigMap.

When you're ready apply both the `not-secret-config` and `volume-mount-explorer` resources and get the logs for the resulting Pod.

The logs should contain results similar to the following:

```
total 12
drwxrwxrwx    3 root     root          4096 Sep  4 00:19 .
drwxr-xr-x    1 root     root          4096 Sep  4 00:19 ..
drwxr-xr-x    2 root     root          4096 Sep  4 00:19 ..2018_09_04_00_19_08.435754939
lrwxrwxrwx    1 root     root            31 Sep  4 00:19 ..data -> ..2018_09_04_00_19_08.435754939
lrwxrwxrwx    1 root     root            15 Sep  4 00:19 endpoint -> ..data/endpoint
lrwxrwxrwx    1 root     root            15 Sep  4 00:19 password -> ..data/password
lrwxrwxrwx    1 root     root            15 Sep  4 00:19 username -> ..data/username
Jeff
```

There is quite a bit going on here. You might have expected to find realized files for each of the data properties of the ConfigMap, but it looks like those files are symlinks. When a ConfigMap already being consumed in a volume is updated, projected keys are eventually updated as well. Kubelet (the clustering component and local orchestrator) is checking whether the mounted ConfigMap is fresh on every periodic sync. What has been realized in the volume is a current snapshot of the data and a set of symlinks to the "current version."

If multiple actors have access to change the values in a ConfigMap resources then they might make for effective information sharing devices. But, it is important to note that **Kubernetes will update the values in a mounted ConfigMap, but it will not take any specific action to notify the running software that configuration has changed**. This means that it is still up to the software to detect and reload the configuration.

The last line of the logs should display the content of the `username` property in the ConfigMap. In this case it says, `Jeff`.

You can verify this quickly using `describe` to inspect the ConfigMap:

```
kubectl describe configmap not-secret-config
```

ConfigMaps do not make any attempts to hide content. While you can apply RBAC to these resources to control who and what might access the data, secret material should be handled more purposefully.

### Exercise 4: Handling Secret Data and Mounting it as a Volume

Kubernetes secrets have a complicated history. The platform relies heavily on extensibility patterns and features like network security, database encryption, RBAC, and cryptographic operations are all provided by components that interact with the API or a plugin interface. Until late 2017 Kubernetes secrets were stored just like every other resource, in plaintext in etcd (the key-value storage engine). To make matters worse, network traffic between the Kubernetes API servers and etcd, or between etcd cluster nodes was not encrypted by default. While some Kubernetes distributions claimed to include more secure secret storage facilities, it wasn't until late 2017 and into 2018 that Kubernetes developed the EncryptionProvider and the associated EncryptionConfig resource. Configuring that resource is beyond the scope of this workshop, but know that its release and adoption by production clusters make using Kubernetes secrets a best practice.

Next convert the `not-secret-config` ConfigMap resource from the previous exercise to a `Secret` (in the `v1` apiVersion). Name the new Secret resource, `pretty-secret-config`. The structure of a Secret is similar to a ConfigMap, but the values in the `data` map must be base64 encoded. I've provided the base64 encoded values for you below:

| Key | Value |
|:-----|:-------------:|
|username|SmVmZg==|
|password|VGVhY2hlc1RoaXNDbGFzcw==|
|endpoint|aHR0cDovL2xvY2FsaG9zdDo4MDgwL2xvZ2lu|

Go ahead and use `kubectl apply -f <file containing pretty-secret-config>` to create the secret. Once it has been created run `kubectl describe secret pretty-secret-config` to verify that the contents are not printed to the terminal. *Note: I still don't like that it leaks content metadata. But that doesn't really matter if you have access to read it, or create Pods (as you'll see next).*

Now create a new CronJob that creates Jobs that mount `pretty-secret-config` as a volume and use the material.

CronJobs are a beta resource in Kubernetes 1.10 so the `apiVersion` that you should use is `batch/v1beta1`. Get the API details with `kbuectl explain CronJob --api-version batch/v1beta1`. The top-level fields are fairly standard, but the `spec` is worth investigating: `kubectl explain CronJob.spec --api-version batch/v1beta1`.

Create a new CronJob named `login-heartbeat` that executes a Job every minute. The Jobs should:

* use the `alpine:3.8` image
* mount the secret material from `pretty-secret-config` at `/run/secret/`
* use the command: `["/bin/sh", "-c", "echo $(cat /run/secret/username) has logged into $(cat /run/secret/endpoint)"]`

*Hints*:
* Use the `secret` and it's child, `secretName` to specify a Secret as a source for a volume.
* Be careful not to get lost in the "spec/template soup." The correct structure should nest as follows: `spec > jobTemplate > spec > template > spec > containers`.
* There is a sample solution at `./ex4/solution/login-heartbeat.yaml`

When you're ready, launch your CronJob and wait for the current minute to finish. When it does the cronjob-controller will create a new Job based on the template you provided, which will create a new Pod. Check those out `kubectl get jobs; kubectl get pods`.

Once the job has completed view the logs to see that the secret material is exposed as plaintext in the logs. This means two things:

* Any actor with access to create Pods can specify Pods that need access to a secret and reveal the secret material.
* Kubernetes cannot protect secrets from being exposed by insecure applications.

## Managing Persistent State: StatefulSets

Yesterday you worked with a voting application that made use of a Postgres database, but every time you updated the Pod spec that contained that database the state was lost. Kubernetes models persistent state and the storage with a collection of resources. Some of those resources are part of standard configuration that ships with the Kubernetes distro, others might be added by cluster administrators. Users should be concerned with managing only a few: StatefulSets, PersistentVolumes, and PersistentVolumeClaims. This section will introduce the basic tooling for working with persistent data.

A StatefulSet, "manages the deployment and scaling of a set of Pods, and provides guarantees about the ordering and uniqueness of these Pods... Unlike a Deployment, a StatefulSet maintains a sticky identity for each of their Pods." In addition to a Pod spec, StatefulSets have a `volumeClaimTemplate` field which is used to describe the PersistentVolumeClaims required by each Pod.

A [PersistantVolumeClaim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) is a request for storage by a user/Pod. The claim describes properties of the storage and from what StorageClass it should come.

A [StorageClass](https://kubernetes.io/docs/concepts/storage/storage-classes/) is a resource controlled by a cluster admin that describe types of storage, or storage "profiles.". "Different classes might map to quality-of-service levels, or to backup policies, or to arbitrary policies determined by the cluster administrators." You can see the StorageClasses available on your machine by running `kubectl get storageclass` and using `kubectl describe` to examine any of the resources. On Docker for Desktop you should see the following:

```
NAME                 PROVISIONER          AGE
hostpath (default)   docker.io/hostpath   19d
```

A [PersistentVolume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistent-volumes) is a description of realized storage. They are either provisioned ahead of time by an administrator, or dynamically by a PersistentVolumeProvisioner. In the exercises below the statefulset controller will work with the PersistentVolumeProvisioner for the `hostpath` StorageClass. 

### Exercise 5: Adding Persistent Storage to the Voting App Data Pipeline

__ kubectl apply -f solution-ns.yaml
__ kubectl apply -f data.yaml
__ kubectl apply -f results.yaml
__ kubectl apply -f voter.yaml

__ go vote and view the results

__ update db to postgres:9.4-alpine
__ kubectl apply -f data-v2.yaml
__ the results app crashes without retrying database connections so we have to trigger a redeployment, just kill the pod and let the ReplicaSet re-launch it
__ kubectl delete <the resutls pod>

__ reload the results page - validate that the old vote is still accounted for

__ try deleting the db container
__ delete the results container as well

__ reload the results page - validate that the old vote is still accounted for

## Service Accounts, and Roles in Kubernetes RBAC

