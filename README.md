## Distributed Load Testing Using Kubernetes

[![Docker Repository on Quay](https://quay.io/repository/honestbee/locust/status "Docker Repository on Quay")](https://quay.io/repository/honestbee/locust)

This tutorial demonstrates how to conduct distributed load testing using [Kubernetes](http://kubernetes.io) and includes a sample web application, Docker image, and Kubernetes controllers/services. For more background refer to the [Distributed Load Testing Using Kubernetes](http://cloud.google.com/solutions/distributed-load-testing-using-kubernetes) solution paper.

## Prerequisites

* Google Cloud Platform account
* Install and setup [Google Cloud SDK](https://cloud.google.com/sdk/)
* Install and setup [Helm](https://github.com/kubernetes/helm)

**Note:** when installing the Google Cloud SDK you will need to enable the following additional components:

* `App Engine Command Line Interface (Preview)`
* `App Engine SDK for Python and PHP`
* `Compute Engine Command Line Interface`
* `Developer Preview gcloud Commands`
* `gcloud Alpha Commands`
* `gcloud app Python Extensions`
* `kubectl`

Before continuing, you can also set your preferred zone and project:

    $ gcloud config set compute/zone ZONE
    $ gcloud config set project PROJECT-ID

## Deploy Web Application

The `sample-webapp` folder contains a simple Google App Engine Python application as the "system under test". To deploy the application to your project use the `gcloud preview app deploy` command.

    $ gcloud preview app deploy sample-webapp/app.yaml --project=PROJECT-ID --set-default

**Note:** you will need the URL of the deployed sample web application when deploying the `locust-master` and `locust-worker` controllers.

### Update Controller Docker Image (Optional)

The `locust-master` and `locust-worker` Deployments are set to use the pre-built `locust` Docker image, which has been uploaded to [Quay Registry](https://quay.io/repository/honestbee/locust) and is available at `quay.io/honestbee/locust`. If you are interested in making changes and publishing a new Docker image, refer to the following steps.

First, [install Docker](https://docs.docker.com/installation/#installation) on your platform. Once Docker is installed and you've made changes to the `Dockerfile`, you can build, tag, and upload the image using the following steps:

    $ docker build -t USERNAME/locust .
    $ docker tag USERNAME/locust gcr.io/PROJECT-ID/locust
    $ gcloud preview docker --project PROJECT-ID push gcr.io/PROJECT-ID/locust

**Note:** you are not required to use the Google Container Registry. If you'd like to publish your images to the [Docker Hub](https://hub.docker.com) please refer to the steps in [Working with Docker Hub](https://docs.docker.com/userguide/dockerrepos/).

Once the Docker image has been rebuilt and uploaded to the registry you will need to specify your new image location for the helm chart. Specifically, the `image.repository` field of the helm chart which Docker image to use.

If you uploaded your Docker image to the Google Container Registry:

    image: gcr.io/PROJECT-ID/locust:latest

If you uploaded your Docker image to the Docker Hub:

    image: USERNAME/locust:latest

**Note:** the image location includes the `latest` tag so that the image is pulled down every time a new Pod is launched. To use a Kubernetes-cached copy of the image, remove `:latest` from the image location.

### Deploy Kubernetes Cluster

First create the [Google Container Engine](http://cloud.google.com/container-engine) cluster using the `gcloud` command as shown below. 

**Note:** This command defaults to creating a three node Kubernetes cluster (not counting the master) using the `n1-standard-1` machine type. Refer to the [`gcloud alpha container clusters create`](https://cloud.google.com/sdk/gcloud/reference/alpha/container/clusters/create) documentation information on specifying a different cluster configuration.

    $ gcloud alpha container clusters create CLUSTER-NAME

After a few minutes, you'll have a working Kubernetes cluster with three nodes (not counting the Kubernetes master). Next, configure your system to use the `kubectl` command:

    $ kubectl config use-context gke_PROJECT-ID_ZONE_CLUSTER-NAME

**Note:** the output from the previous `gcloud` cluster create command will contain the specific `kubectl config` command to execute for your platform/project.

## Deploy Controllers and Services

Using [Helm](github.com/kubernetes/helm) to deploy locust, pass in your sample web application URL via the `master.config.target-host` key:

```
helm install stable/locust -n locust-nymph --set master.config.target-host=http://PROJECT-ID.appspot.com
```
This step will expose the Pod with an internal DNS name (`locust-master`) and ports `8089`, `5557`, and `5558`. As part of this step, the `type: LoadBalancer` directive in `locust-master-service.yaml` will tell Google Container Engine to create a Google Compute Engine forwarding-rule from a publicly avaialble IP address to the `locust-master` Pod. To view the newly created forwarding-rule, execute the following:

    $ gcloud compute forwarding-rules list 

### Scaling locust-worker

The `locust-worker-controller` is set to deploy 1 `locust-worker` Pods, to confirm they were deployed run the following:

    $ kubectl get pods -l name=locust,role=worker

To scale the number of `locust-worker` Pods, issue a replication controller `scale` command.

    $ kubectl scale --replicas=20 deployment locust-worker

To confirm that the Pods have launched and are ready, get the list of `locust-worker` Pods:

    $ kubectl get pods -l name=locust,role=worker

**Note:** depending on the desired number of `locust-worker` Pods, the Kubernetes cluster may need to be launched with more than 3 compute engine nodes and may also need a machine type more powerful than n1-standard-1. Refer to the [`gcloud alpha container clusters create`](https://cloud.google.com/sdk/gcloud/reference/alpha/container/clusters/create) documentation for more information.

### Setup Firewall Rules

The final step in deploying these controllers and services is to allow traffic from your publicly accessible forwarding-rule IP address to the appropriate Container Engine instances.

The only traffic we need to allow externally is to the Locust web interface, running on the `locust-master` Pod at port `8089`. First, get the target tags for the nodes in your Kubernetes cluster using the output from `kubectl get nodes`:

    $ kubectl get nodes
    NAME                        LABELS                                             STATUS
    gke-ws-0e365264-node-4pdw   kubernetes.io/hostname=gke-ws-0e365264-node-4pdw   Ready
    gke-ws-0e365264-node-jdcz   kubernetes.io/hostname=gke-ws-0e365264-node-jdcz   Ready
    gke-ws-0e365264-node-kp3d   kubernetes.io/hostname=gke-ws-0e365264-node-kp3d   Ready

The target tag is the node name prefix up to `-node` and is formatted as `gke-CLUSTER-NAME-[...]-node`. For example, if your node name is `gke-mycluster-12345678-node-abcd`, the target tag would be `gke-mycluster-12345678-node`. 

Now to create the firewall rule, execute the following:

    $ gcloud compute firewall-rules create FIREWALL-RULE-NAME --allow=tcp:8089 --target-tags gke-CLUSTER-NAME-[...]-node

## Execute Tests

To execute the Locust tests, navigate to the IP address of your forwarding-rule (see above) and port `8089` and enter the number of clients to spawn and the client hatch rate then start the simulation.

## Deployment Cleanup

To teardown the workload simulation cluster, use the following steps. First, delete the Kubernetes cluster:

    $ gcloud alpha container clusters delete CLUSTER-NAME

Next, delete the forwarding rule that forwards traffic into the cluster.

    $ gcloud compute forwarding-rules delete FORWARDING-RULE-NAME

Finally, delete the firewall rule that allows incoming traffic to the cluster.

    $ gcloud compute firewall-rules delete FIREWALL-RULE-NAME

To delete the sample web application, visit the [Google Cloud Console](https://console.developers.google.com).

## License

This code is Apache 2.0 licensed and more information can be found in `LICENSE`. For information on licenses for third party software and libraries, refer to the `docker-image/licenses` directory.
