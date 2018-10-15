---
title: "Supporting Canary Deployments for Fission Functions"
draft: false
weight: 49
---

This tutorial will walk you through setting up a canary config to deploy a new version of a function on your cluster with minimal risk in a way that it gradually serves user traffic starting from 0% going all the way to 100% eventually.

## Setup & pre-requisites

This feature is dependent on Prometheus metrics to check the health of the new version of the function. Hence, Prometheus is listed as a dependency for fission chart. 
TBD : 
1. Write about how to enable canary feature once new code gets in
2. Write about how users can opt for prometheus installation along with fission or just ensure prometheus is already deployed on the cluster and it's service is accessible from fission controller.

### Canary Config parameters

A Canary Config has the following parameters :

``` 
duration: Specifies how frequently the amount of user traffic to be handled by the new version of the functions needs to be incremented for the new version of function

failurethreshold: Specifies the threshold in percentage above which the new version of a function is declared unhealthy

funcn: Specifies the name of the latest version of the function

funcn-1: Specifies the name of the current stable version of the function

trigger: Specifies the name of the http trigger object 

weightincrement: Specifies the percentage increase of user traffic towards the new version of the function

failureType: Specifies the parameter for checking the health of the new version of a function. For now, the only supported type is `status-code` which is the http status code. So if a function returns a status code other than 200, its considered to be unhealthy.  
```

Taking an example to clarify these parameters - let's suppose that the current stable version of a function that's deployed is fn-a-v1 and that we want to deploy the latest version of the function which is fn-a-v2. Let's also suppose that we want to increment the traffic towards the new version in steps of 30% every 1m with a failure threshold of 10%. 
For such a scenario, you can configure the canary config as given below.

```bash
apiVersion: fission.io/v1
kind: CanaryConfig
metadata:
  name: canary-1
  namespace: default
spec:
  duration: 1m
  failureType: status-code
  failurethreshold: 10
  funcn: fn-a-v2
  funcn-1: fn-a-v1
  trigger: route-fn-a
  weightincrement: 30
```

What happens is that every 1m, the percentage of failed requests to fn-a-v2 gets calculated from prometheus metrics. If it is under the configured failure threshold of 10%, then the percentage traffic to fn-a-v2 gets incremented by 30% and this cycle repeats until - either the failure threshold has reached at which point the deployment is rolled back, or, fn-v2 is receiving 100% of the user traffic.   

### Steps to setup a canary config

1. Create an environment :

```bash
fission env create --name nodejs --image fission/node-env
```

2. Create fission functions :

```bash
fission fn create --name fn-a-v1 --code hello.js --env nodejs
fission fn create --name fn-a-v2 --code hello2.js --env nodejs
```

3. Create an http trigger to these functions :

```bash
fission route create --name route-fn-a --function fn-a-v1 --weight 100 --function fn-a-v2 --weight 0
```

4. Create a canary config :

```bash
fission canary-config create --name canary-1 --funcN fn-a-v2 --funcN-1 fn-a-v1 --trigger route-fn-a --increment-step 30 --increment-interval 1m --failure-threshold 10
```

### Steps to verify the status of a canary deployment

```bash
fission canary-config get --name canary-1
```

This prints the status of the canary deployment of the new version of the function. 
1. The status is "Pending" if the canary deployment is in progress.
2. The status is "Succeeded" if the new version of the function is receiving 100% of the user traffic.
3. The status is "Failed" if the failure threshold reached for the new version of the function and as a result 100% of the traffic gets routed to the old version of the function(rollback).
4. The status is "Aborted" if there were some failures during the canary deployment.

### Note

The `scrape_interval` for Prometheus server is 1m by default. If the "duration" parameter needs to be less than 1m, the `scrape_interval` parameter needs to configured to a much lower value.
This can be done by updating the config map for prometheus server. Just updating the config map is enough, prometheus server need not be restarted. 
 

