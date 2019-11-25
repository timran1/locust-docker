## Locust for Kubernetes

[![CircleCI](https://circleci.com/gh/greenbird/locust.svg?style=svg)](https://circleci.com/gh/greenbird/locust)

This reporitory is forked from the tutorial from Google on how to conduct distributed load testing using [Kubernetes](http://kubernetes.io).
We have removed the sample webapp and will only be focusing on maintaining an up to date Locust image for use in Helm Charts and other Kubernetes deployments.

In addition to the groundwork laid down by Google, we've merged updates from the [honestbee](https://github.com/honestbee/distributed-load-testing) fork and built from there.

For instructions on how to use the Helm Chart to deploy to Kubernetes, we refer to [the chart](https://github.com/helm/charts/tree/master/stable/locust). 
And for more background on this topic we refer to the [Distributed Load Testing Using Kubernetes](http://cloud.google.com/solutions/distributed-load-testing-using-kubernetes) solution paper.

**Note:** the image location includes the `latest` tag so that the image is pulled down every time a new Pod is launched. To use a Kubernetes-cached copy of the image, remove `:latest` from the image location.
### Testing Locust locally

Locust can be tested locally using `docker-compose` as follows:

1. Start master and worker:

   ```
   docker-compose -p locust up -d
   ```

1. Verify master and worker logs:

   ```
   docker-compose -p locust logs
   ```

1. Open web console:

   ```
   open localhost:8089
   ```

1. Scale up workers:

   ```
   docker-compose -p locust scale worker=4
   ```

Shutting down local test.

```
docker-compose -p locust stop
```


## License

This code is Apache 2.0 licensed and more information can be found in `LICENSE`. For information on licenses for third party software and libraries, refer to the `docker-image/licenses` directory.
