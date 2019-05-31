# Install a Solace PubSub+ Software Message Broker onto a Pivotal Container Service (PKS) cluster

## Purpose of this repository

This repository explains how to install a Solace PubSub+ Software Message Broker in various configurations onto a [Pivotal Container Service (PKS)](//cloud.vmware.com/pivotal-container-service ) cluster.

The recommended Solace PubSub+ Software Message Broker version is 9.1 or later.

For deploying Solace PubSub+ Software Message Broker in a generic Kubernetes environment, refer to the [Solace Kubernetes Quickstart project](//github.com/SolaceProducts/solace-kubernetes-quickstart ).

## Description of the Solace PubSub+ Software Message Broker

The Solace PubSub+ software message broker meets the needs of big data, cloud migration, and Internet-of-Things initiatives, and enables microservices and event-driven architecture. Capabilities include topic-based publish/subscribe, request/reply, message queues/queueing, and data streaming for IoT devices and mobile/web apps. The message broker supports open APIs and standard protocols including AMQP, JMS, MQTT, REST, and WebSocket. Moreover, it can be deployed in on-premise datacenters, natively within private and public clouds, and across complex hybrid cloud environments.

Solace PubSub+ software message brokers can be deployed in either a 3-node High-Availability (HA) cluster, or as a single-node non-HA deployment. For simple test environments that need only to validate application functionality, a single instance will suffice. Note that in production, or any environment where message loss cannot be tolerated, an HA cluster is required.

## How to deploy a message broker onto PKS

In this quick start we go through the steps to set up a message broker either as a single-node instance (default settings), or in a 3-node HA cluster.

### Step 1: Access to PKS

Perform any prerequisites to access PKS v1.4 or later in your target environment. For specific details, refer to your PKS platform's documentation.

Tasks may include:

* Getting access to a platform which supports PKS, such as VMware Enterprise PKS
* Install Kubernetes [`kubectl`](//kubernetes.io/docs/tasks/tools/install-kubectl/ ) tool.
* Install the PKS CLI client and log in.
* Create a PKS cluster (for CPU and memory requirements of your Solace message broker target deployment configuration, refer to the [Deployment Configurations](#message-broker-deployment-configurations) section)
* Configure any necessary environment settings and install certificates
* Fetch the credentials for the PKS cluster
* Perform any necessary setup and configure access if using a specific Docker image registry, such as Harbor
* Perform any necessary setup and configure access if using a specific Helm chart repository, such as Harbor

Verify access to your PKS cluster and the available nodes by running `kubectl get nodes -o wide` from your environment.

### Step 2: Deploy Helm package manager

We recommend using the [Kubernetes Helm](//github.com/kubernetes/helm/blob/master/README.md ) tool to manage the deployment.

This GitHub repo provides a helper script which will install, setup and configure Helm in your environment if not already there.

Clone this repo and execute the `setup_helm` script:
```sh
mkdir ~/workspace; cd ~/workspace
git clone //github.com/SolaceProducts/solace-pks.git
cd solace-pks    # repo root directory
./scripts/setup_helm.sh
```

### Step 3 (Optional): Load the Solace Docker image to a Docker image registry

**Hint:** You may skip the rest of this step if not using a specific Docker image registry (Harbor). The free PubSub+ Standard Edition is available from the [public Docker Hub registry](//hub.docker.com/r/solace/solace-pubsub-standard/tags/ ), in this case the docker registry reference to use will be `solace/solace-pubsub-standard:<TagName>`.

To get the Solace message broker Docker image, go to the Solace Developer Portal and download the Solace PubSub+ software message broker as a **docker** image or obtain your version from Solace Support.

| PubSub+ Standard<br/>Docker Image | PubSub+ Enterprise Evaluation Edition<br/>Docker Image
| :---: | :---: |
| Free, up to 1k simultaneous connections,<br/>up to 10k messages per second | 90-day trial version, unlimited |
| [Download Standard docker image](http://dev.solace.com/downloads/ ) | [Download Evaluation docker image](http://dev.solace.com/downloads#eval ) |

To load the Docker image tar archive file into a docker registry, follow the steps specific to the registry you are using.

This is a general example:
* [`docker`](//docs.docker.com/get-started/ ) is required
* First load the image to the local docker registry:
```
sudo docker load -i <solace-pubsub-XYZ-docker>.tar.gz
# Verify image has been loaded, note "IMAGE ID"
sudo docker images
```
* Login to the target registry
`sudo docker login <target-registry> ...`
* Tag the image with the targeted name and tag
`sudo docker tag <image-id> <target-registry>/<path>/<image-name>:<tag>`
* Push the image to the target registry
`sudo docker push <target-registry>/<path>/<image-name>:<tag>`

Note that additional steps may be required if using signed images.


### Step 4: Deploy message broker Pods and Service to the cluster

A deployment is defined by a "Helm chart", which consists of templates and values. The values specify the particular configuration properties in the templates. The generic [Solace Kubernetes Quickstart project](//github.com/SolaceProducts/solace-kubernetes-quickstart#step-4 ) provides additional details about the templates used.

For the "solace" Helm chart the values are in the `values.yaml` file located in the `solace` directory:
```sh
cd ~/workspace/solace-pks/solace
more values.yaml
``` 

Update: @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
For a description of all value configuration properties, refer to section XYZ

When Helm is used to install a deployment the configuration properties can be set in several ways, in combination of the followings:

* By default, if no other values file is specified, settings from the `values.yaml` in the chart directory is used
```sh
# <chart-location>: directory path, url or Helm repo reference to the chart
helm install <chart-location>  
```
* Settings can be taken from a specified values file; multiple can be specified in a sequence with left-to-right overriding/complementing previous ones, beginning from `values.yaml` in the chart directory. A values file may define only a subset of values.
```sh
# <my-values-file>: directory path to a values filehelm
helm install -f <my-values-file> <chart-location>
```
* Explicitly overriding settings
```sh
# overrides the setting in values.yaml in the chart directory, can pass multiple
helm install <chart-location> --set <param1>=<value1>[,<param2>=<value2>]
```

Helm will generate a release name is not specified. Here is how to specify it:
```sh
# Helm will reference this deployment as "my-solace-ha-release"
helm install --name my-solace-ha-release <chart-location>
```

Check for dependencies before going ahead with a deployment: the Solace deployment may depend on the presence of Kubernetes objects, such as a StorageClass and ImagePullSecrets. These need to be created first if required.

#### Ensure a StorageClass is available

The Solace deployment uses disk storage for logging, configuration, guaranteed messaging and other purposes. The use of a persistent storage is recommended, otherwise data will be lost with the loss of a pod.

A [StorageClass](https://kubernetes.io/docs/concepts/storage/storage-classes/ ) is used to obtain persistent storage from outside the pod.

For a list of of available StorageClasses, execute
```sh
kubectl get storageclass
```

It is expected that there is at least one StorageClass available. By default, the "solace" chart assumes the name `standard`, adjust the `storage.useStorageClass` value if necessary.

Refer to your PKS environment's documentation if a StorageClass needs to be created or to understand the differences if there are multiple options.


#### Create and use ImagePullSecrets for signed images

ImagePullSecrets may be required if using signed images from a private Docker registry, including Harbor.

Here is an example of creating a secret. Refer to your registry's documentation for the specific details of use.

```sh
kubectl create secret docker-registry <pull-secret-name> --dockerserver=<target-registry-server> \
  --docker-username=<registry-user-name> --docker-password=<registry-user-password> \
  --docker-email=<registry-user-email>
```

Then set the `image.pullSecretName` value to `<pull-secret-name>`.

#### Use Helm to install the deployment

##### Single-node non-HA deployment

The default values in the `values.yaml` file in this repo configure a small single-node deployment (`redundancy: false`) with up to 100 connections (`size: prod100`).

```sh
cd ~/workspace/solace-pks/solace
# Use contents of values.yaml and override the admin password
helm install --name my-solace-nonha-release . --set solace.redundancy=false,solace.usernameAdminPassword=Ch@ngeMe!
# Wait until the pod is running and ready and the active message broker pod label is "active=true"
watch kubectl get pods --show-labels
```

##### HA deployment

The only difference to the non-HA deployment in the simple case is to set `solace.redundancy=true`.

```sh
cd ~/workspace/solace-pks/solace
# Use contents of values.yaml and override the admin password
helm install --name my-solace-ha-release . -f values.yaml --set solace.redundancy=true,solace.usernameAdminPassword=Ch@ngeMe!
# Wait until all pods running and ready and the active message broker pod label is "active=true"
watch kubectl get pods --show-labels
```

To modify a deployment, refer to the section [Upgrading/modifying the message broker cluster](#upgradingmodifying-the-message-broker-cluster). If you need to start over then refer to the section [Deleting a deployment](#deleting-a-deployment).

### Validate the Deployment

Now you can validate your deployment on the command line. In this example an HA cluster is deployed with po/XXX-XXX-solace-0 being the active message broker/pod. The notation XXX-XXX is used for the unique release name that Helm dynamically generates, e.g: "tinseled-lamb".

```sh
prompt:~$ kubectl get statefulsets,services,pods,pvc,pv
NAME                                 DESIRED   CURRENT   AGE
statefulsets/XXX-XXX-solace   3         3         3m
NAME                                  TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                                       AGE
svc/XXX-XXX-solace             LoadBalancer   10.15.249.186   35.202.131.158   22:32656/TCP,8080:32394/TCP,55555:31766/TCP   3m
svc/XXX-XXX-solace-discovery   ClusterIP      None            <none>           8080/TCP                                      3m
svc/kubernetes                        ClusterIP      10.15.240.1     <none>           443/TCP                                       6d
NAME                         READY     STATUS    RESTARTS   AGE
po/XXX-XXX-solace-0   1/1       Running   0          3m
po/XXX-XXX-solace-1   1/1       Running   0          3m
po/XXX-XXX-solace-2   1/1       Running   0          3m
NAME                               STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS              AGE
pvc/data-XXX-XXX-solace-0   Bound     pvc-74d9ceb3-d492-11e7-b95e-42010a800173   30Gi       RWO            XXX-XXX-standard   3m
pvc/data-XXX-XXX-solace-1   Bound     pvc-74dce76f-d492-11e7-b95e-42010a800173   30Gi       RWO            XXX-XXX-standard   3m
pvc/data-XXX-XXX-solace-2   Bound     pvc-74e12b36-d492-11e7-b95e-42010a800173   30Gi       RWO            XXX-XXX-standard   3m
NAME                                          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                                  STORAGECLASS              REASON    AGE
pv/pvc-74d9ceb3-d492-11e7-b95e-42010a800173   30Gi       RWO            Delete           Bound     default/data-XXX-XXX-solace-0   XXX-XXX-standard             3m
pv/pvc-74dce76f-d492-11e7-b95e-42010a800173   30Gi       RWO            Delete           Bound     default/data-XXX-XXX-solace-1   XXX-XXX-standard             3m
pv/pvc-74e12b36-d492-11e7-b95e-42010a800173   30Gi       RWO            Delete           Bound     default/data-XXX-XXX-solace-2   XXX-XXX-standard             3m


prompt:~$ kubectl describe service XXX-XX-solace
Name:                     XXX-XXX-solace
Namespace:                default
Labels:                   app=solace
                          chart=solace-0.3.0
                          heritage=Tiller
                          release=XXX-XXX
Annotations:              <none>
Selector:                 active=true,app=solace,release=XXX-XXX
Type:                     LoadBalancer
IP:                       10.55.246.5
LoadBalancer Ingress:     35.202.131.158
Port:                     ssh  22/TCP
TargetPort:               2222/TCP
NodePort:                 ssh  30828/TCP
Endpoints:                10.52.2.6:2222
:
:
```

Generally, all services including management and messaging are accessible through a Load Balancer. In the above example `35.202.131.158` is the Load Balancer's external Public IP to use.

## Available ports

@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
Refer to //docs.solace.com/Configuring-and-Managing/Default-Port-Numbers.htm

## Gaining admin access to the message broker

Refer to the [Management Tools section](//docs.solace.com/Management-Tools.htm ) of the online documentation to learn more about the available tools. The WebUI is the recommended simplest way to administer the message broker for common tasks.

### WebUI, SolAdmin and SEMP access

Use the Load Balancer's external Public IP at port 8080 to access these services.

### Solace CLI access

If you are using a single message broker and are used to working with a CLI message broker console access, you can SSH into the message broker as the `admin` user using the Load Balancer's external Public IP:

```sh

$ssh -p 22 admin@35.202.131.158
Solace PubSub+ Standard
Password:

Solace PubSub+ Standard Version 8.10.0.1057

The Solace PubSub+ Standard is proprietary software of
Solace Corporation. By accessing the Solace PubSub+ Standard
you are agreeing to the license terms and conditions located at
http://www.solace.com/license-software

Copyright 2004-2018 Solace Corporation. All rights reserved.

To purchase product support, please contact Solace at:
http://dev.solace.com/contact-us/

Operating Mode: Message Routing Node

XXX-XXX-solace-0>
```

If you are using an HA cluster, it is better to access the CLI through the Kubernets pod and not directly via SSH.

Note: SSH access to the pod has been configured at port 2222. For external access SSH has been configured to to be exposed at port 22 by the load balancer.

* Loopback to SSH directly on the pod

```sh
kubectl exec -it XXX-XXX-solace-0  -- bash -c "ssh -p 2222 admin@localhost"
```

* Loopback to SSH on your host with a port-forward map

```sh
kubectl port-forward XXX-XXX-solace-0 62222:2222 &
ssh -p 62222 admin@localhost
```

This can also be mapped to individual message brokers in the cluster via port-forward:

```
kubectl port-forward XXX-XXX-solace-0 8081:8080 &
kubectl port-forward XXX-XXX-solace-1 8082:8080 &
kubectl port-forward XXX-XXX-solace-2 8083:8080 &
```

For SSH access to individual message brokers use:

```sh
kubectl exec -it XXX-XXX-solace-<pod-ordinal> -- bash
```

## Viewing logs
Logs from the currently running container:

```sh
kubectl logs XXX-XXX-solace-0 -c solace
```

Logs from the previously terminated container:

```sh
kubectl logs XXX-XXX-solace-0 -c solace -p
```

## Testing data access to the message broker

To test data traffic though the newly created message broker instance, visit the Solace Developer Portal and and select your preferred programming language in [send and receive messages](http://dev.solace.com/get-started/send-receive-messages/). Under each language there is a Publish/Subscribe tutorial that will help you get started and provide the specific default port to use.

Use the external Public IP to access the cluster. If a port required for a protocol is not opened, refer to the next section on how to open it up by modifying the cluster.

## Upgrading/modifying the message broker cluster

To upgrade/modify the message broker cluster, make the required modifications to the chart in the `solace-kubernetes-quickstart/solace` directory as described next, then run the Helm tool from here. When passing multiple `-f <values-file>` to Helm, the override priority will be given to the last (right-most) file specified.

### Restoring Helm if not available

Before getting into the details of how to make changes to a deployment, it shall be noted that when using a new machine to access the deployment the Helm client may not be available or out of sync with the server. This can be the case when e.g. using cloud shell, which may be terminated any time.

To restore Helm, run the configure command with no parameter provided:

```
cd ~/workspace/solace-kubernetes-quickstart/solace
../scripts/configure.sh
```

Now Helm shall be available on your client, e.g: `helm list` shall no longer return an error message.

### Upgrading the cluster

To **upgrade** the version of the message broker running within a Kubernetes cluster:

- Add the new version of the message broker to your container registry.
- Create a simple upgrade.yaml file in solace-kubernetes-quickstart/solace directory, e.g.:

```sh
image:
  repository: <repo>/<project>/solace-pubsub-standard
  tag: NEW.VERSION.XXXXX
  pullPolicy: IfNotPresent
```
- Upgrade the Kubernetes release, this will not effect running instances

```sh
cd ~/workspace/solace-kubernetes-quickstart/solace
helm upgrade XXX-XXX . -f values.yaml -f upgrade.yaml
```

- Delete the pod(s) to force them to be recreated with the new release. 

```sh
kubectl delete po/XXX-XXX-solace-<pod-ordinal>
```
> Important: In an HA deployment, delete the pods in this order: 2,1,0 (i.e. Monitoring Node, Backup Messaging Node, Primary Messaging Node). Confirm that the message broker redundancy is up and reconciled before deleting each pod - this can be verified using the CLI `show redundancy` and `show config-sync` commands on the message broker, or by grepping the message broker container logs for `config-sync-check`.

### Modifying the cluster

Similarly, to **modify** other deployment parameters, e.g. to change the ports exposed via the loadbalancer, you need to upgrade the release with a new set of ports. In this example we will add the MQTT 1883 tcp port to the loadbalancer.

```
cd ~/workspace/solace-kubernetes-quickstart/solace
tee ./port-update.yaml <<-EOF   # create update file with following contents:
service:
  internal: false
  type: LoadBalancer
  externalPort:
    - port: 1883
      protocol: TCP
      name: mqtt
      targetport: 1883
    - port: 22
      protocol: TCP
      name: ssh
      targetport: 2222
    - port: 8080
      protocol: TCP
      name: semp
    - port: 55555
      protocol: TCP
      name: smf
    - port: 943
      protocol: TCP
      name: semptls
      targetport: 60943
    - port: 80
      protocol: TCP
      name: web
      targetport: 60080
    - port: 443
      protocol: TCP
      name: webtls
      targetport: 60443
  internalPort:
    - port: 2222
      protocol: TCP
    - port: 8080
      protocol: TCP
    - port: 55555
      protocol: TCP
    - port: 60943
      protocol: TCP
    - port: 60080
      protocol: TCP
    - port: 60443
      protocol: TCP
    - port: 1883
      protocol: TCP
EOF
helm upgrade  XXXX-XXXX . --values values.yaml --values port-update.yaml
```

## Deleting a deployment

Use Helm to delete a deployment, also called a release:

```
helm delete XXX-XXX
```

Check what has remained from the deployment, which should only return a single line with svc/kubernetes:

```
kubectl get statefulsets,services,pods,pvc,pv
NAME                           TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)         AGE
service/kubernetes             ClusterIP      XX.XX.XX.XX     <none>            443/TCP         XX
```

> Note: In some versions, Helm may not be able to clean up all the deployment artifacts, e.g.: pvc/ and pv/. If necessary, use `kubectl delete` to delete those.

## Message Broker Deployment Configurations

The `solace-kubernetes-quickstart/solace/values-examples` directory provides examples for `values.yaml` for several deployment configurations:

* `dev100-direct-noha` (default if no argument provided): for development purposes, supports up to 100 connections, non-HA, simple local non-persistent storage
* `prod1k-direct-noha`: production, up to 1000 connections, non-HA, simple local non-persistent storage
* `prod1k-direct-noha-existingVolume`: production, up to 1000 connections, non-HA, bind the PVC to an existing external volume in the network
* `prod1k-direct-noha-localDirectory`: production, up to 1000 connections, non-HA, bind the PVC to a local directory on the host node
* `prod1k-direct-noha-provisionPvc`: production, up to 1000 connections, non-HA, bind the PVC to a provisioned PersistentVolume (PV) in Kubernetes
* `prod1k-persist-ha-provisionPvc`: production, up to 1000 connections, HA, to bind the PVC to a provisioned PersistentVolume (PV) in Kubernetes
* `prod1k-persist-ha-nfs`: production, up to 1000 connections, HA, to dynamically bind the PVC to an NFS volume provided by an NFS server, exposed as storage class `nfs`. Note: "root_squash" configuration is supported on the NFS server.

Similar value-files can be defined extending above examples:

- To open up more service ports for external access, add new ports to the `externalPort` list. For a list of available services and default ports refer to [Software Message Broker Configuration Defaults](//docs.solace.com/Configuring-and-Managing/SW-Broker-Specific-Config/SW-Broker-Configuration-Defaults.htm) in the Solace customer documentation.

- It is also possible to configure the message broker deployment with different CPU and memory resources to support more connections per message broker, by changing the solace `size` in `values.yaml`. The Kubernetes host node resources must be also provisioned accordingly.

    * `dev100` (default): up to 100 connections, minimum requirements: 1 CPU, 1 GB memory
    * `prod100`: up to 100 connections, minimum requirements: 2 CPU, 2 GB memory
    * `prod1k`: up to 1,000 connections, minimum requirements: 2 CPU, 4 GB memory
    * `prod10k`: up to 10,000 connections, minimum requirements: 4 CPU, 12 GB memory
    * `prod100k`: up to 100,000 connections, minimum requirements: 8 CPU, 28 GB memory
    * `prod200k`: up to 200,000 connections, minimum requirements: 12 CPU, 56 GB memory

## Alternative installation: generating templates for Kubernetes Kubectl tool

This is for users not wishing to install the Helm server-side Tiller on the Kubernetes cluster.

This method will first generate installable Kubernetes templates from this project's Helm charts, then the templates can be installed using the Kubectl tool.

### Step 1: Generate Kubernetes templates for Solace message broker deployment

1) Clone this project:

```sh
git clone //github.com/SolaceProducts/solace-kubernetes-quickstart.git
cd solace-kubernetes-quickstart # This directory will be referenced as <project-root>
```

2) [Download](//github.com/helm/helm/releases/tag/v2.9.1 ) and install the Helm client locally.

We will assume that it has been installed to the `<project-root>/bin` directory.

3) Customize the Solace chart for your deployment

The Solace chart includes raw Kubernetes templates and a "values.yaml" file to customize them when the templates are generated.

The chart is located in the `solace` directory:

`cd <project-root>/solace`

a) Optionally replace the `<project-root>/solace/values.yaml` file with one of the prepared examples from the `<project-root>/solace/values-examples` directory. For details refer to the [Other Deployment Configurations section](#other-message-broker-deployment-configurations) in this document.

b) Then edit `<project-root>/solace/values.yaml` and replace following parameters:

SOLOS_CLOUD_PROVIDER: Current options are "gcp" or "aws" or leave it unchanged for unknown (note: specifying the provider will optimize volume provisioning for supported providers).
<br/>
SOLOS_IMAGE_REPO and SOLOS_IMAGE_TAG: use `solace/solace-pubsub-standard` and `latest` for the latest available or specify a [version from DockerHub](//hub.docker.com/r/solace/solace-pubsub-standard/tags/ ). For more options, refer to the [Solace PubSub+ message broker docker image section](#step-3-optional) in this document. 

c) Configure the Solace management password for `admin` user in `<project-root>/solace/templates/secret.yaml`:

SOLOS_ADMIN_PASSWORD: change it to the desired password, considering the [password rules](//docs.solace.com/Configuring-and-Managing/Configuring-Internal-CLI-User-Accounts.htm#Changing-CLI-User-Passwords ).

4) Generate the templates

```sh
cd <project-root>/solace
# Create location for the generated templates
mkdir generated-templates
# In next command replace myrelease to the desired release name
<project-root>/bin/helm template --name myrelease --values values.yaml --output-dir ./generated-templates .
```

The generated set of templates are now available in the `<project-root>/solace/generated-templates` directory.

### Step 2: Deploy the templates on the target system

Assumptions: `kubectl` is deployed and configured to point to your Kubernetes cluster

1) Optionally copy the `generated-templates` directory with contents if this is on a different host

2) Assign rights to current user to modify cluster-wide RBAC (required for creating clusterrole binding when deploying the Solace template podModRbac.yaml)

Example:

```sh
# Get current user - GCP example. Returns e.g.: myname@example.org
gcloud info | grep Account  
# Assign rigths - replace user
kubectl create clusterrolebinding myname-cluster-admin-binding \
  --clusterrole=cluster-admin \
  --user=myname@example.org
```

3) Initiate the deployment:

`kubectl apply --recursive -f ./generated-templates/solace`

Wait for the deployment to complete, which is then ready to use.

4) To delete deployment, execute:

`kubectl delete --recursive -f ./generated-templates/solace`


## Contributing

Please read [CONTRIBUTING.md](CONTRIBUTING.md) for details on our code of conduct, and the process for submitting pull requests to us.

## Authors

See the list of [contributors](//github.com/SolaceProducts/solace-kubernetes-quickstart/graphs/contributors) who participated in this project.

## License

This project is licensed under the Apache License, Version 2.0. - See the [LICENSE](LICENSE) file for details.

## Resources

For more information about Solace technology in general please visit these resources:

- The Solace Developer Portal website at: http://dev.solace.com
- Understanding [Solace technology.](http://dev.solace.com/tech/)
- Ask the [Solace community](http://dev.solace.com/community/).
