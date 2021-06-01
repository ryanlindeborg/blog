---
layout: post
title: "Scaling Kubernetes with Custom Metrics"
date: 2021-05-31 16:00:00 -0800
permalink: /scaling-kubernetes-with-custom-metrics/
---

<img src="/assets/img/kubernetes-logo.svg" alt="Scaling Kubernetes with custom metrics" width="250" align="center" />

## Start thinking about scaling early
As you build out your production Kubernetes infrastructure, you will want to start thinking early about how you can maintain scalability of your system as traffic and load increases. How do you do this?

Well, you can technically scale up your ReplicaSets manually with the “kubectl autoscale” command or by manually updating your yaml configuration files, but this method of scaling isn’t itself scalable. You don’t want to have to manually increase the number of application pods whenever you get a surge in traffic. Instead, you should employ an autoscaling mechanism to do this pod management for you.

There are various ways you can autoscale Kubernetes deployments: horizontal pod autoscaling, vertical pod autoscaling, and cluster autoscaling. The Vertical Pod Autoscaler (VPA) will try to dynamically allocate pod resource constraints within certain bounds. For example, you can specify a minimum and maximum for CPU or memory. The cluster autoscaler is used to manage the number of nodes in a cluster so that pods are efficiently allocated on different nodes, and so that there are enough nodes to service the resource requirements of the pods being requested in a Deployment. The Horizontal Pod Autoscaler (HPA) dynamically scales the number of applications pods that are spun up based on a desired metric value, per this formula:

```
desiredReplicas = ceil[currentReplicas * ( currentMetricValue / desiredMetricValue )]
```

We’ll focus on the horizontal pod autoscaler in this post.

&nbsp;

## Using existing Kubernetes/cloud provider metrics

Using the Kubernetes autoscaling/v2beta2 API, we can scale off a set of metrics that the metrics.k8s.io API will allow us to query on. To do this, we define a declarative yaml file for our HPA like the following:

```
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
name: php-apache
spec:
scaleTargetRef:
apiVersion: apps/v1
kind: Deployment
name: php-apache
minReplicas: 1
maxReplicas: 10
metrics:
- type: Resource
  resource:
  name: cpu
  target:
  type: Utilization
  averageUtilization: 50
```

In this example, an Apache deployment is scaled via the CPU utilization metric. The target utilization is 50%, meaning Kubernetes will scale up pods (to a maximum of 10 pods) if CPU utilization is greater than 50%, and scale down pods (to a minimum of 1 pod) if CPU utilization is less than 50%.

If you or your company uses AWS, there are some metrics out of the box based on AWS services that you might want to leverage, like SQS queue length or S3 bucket size bytes or number of objects. However, your scaling use case may require more flexibility. What if these metrics are not the right ones to scale on?

&nbsp;

## Custom Metrics to the Rescue
Kubernetes allows us to scale off any custom metric we can store, as long as Kubernetes has an adapter to read from that metric store (you can write your own adapter if it doesn’t exist already). So, basically, we get to decide with full flexibility when Kubernetes will decide if more pods are needed to meet our application needs.

Let’s look at how to implement this using AWS Cloudwatch (AWS metric store) and Amazon EKS (Elastic Kubernetes Service – AWS-managed Kubernetes service). First, we will send PUT requests to AWS Cloudwatch via the Python boto3 client, specifying the metric name, value, units, and associated timestamp:

``` python
import boto3
from datetime import datetime

client = boto3.client('cloudwatch')

cloudwatch_response = client.put_metric_data(
    Namespace='TestMetricNamespace',
    MetricData=[
        {
        'MetricName': 'TestMetric',
        'Timestamp': datetime.utcnow(),
        'Value': 40,
        'Unit': 'Megabytes',
        }
    ]
)
```

That metric value is now stored in Cloudwatch and can be queried by Kubernetes to determine if any autoscaling actions need to be taken. Next, we will define a Kubernetes ExternalMetric object:

extMetricCustom.yaml:
```
apiVersion: metrics.aws/v1alpha1
kind: ExternalMetric
metadata:
name: test-custom-metric
spec:
name: test-custom-metric
resource:
resource: "deployment"
queries:
- id: autoscaling_test
metricStat:
metric:
namespace: "TestMetricNamespace"
metricName: " TestMetric "
period: 60
stat: Average
unit: Megabytes
returnData: true
```

We specify the metric name, namespace, and how we want this metric queried. In this example, we are querying for the average value of this metric over the last 60 seconds. Now, we can define the HPA:

hpaCustomMetric.yaml:
```
kind: HorizontalPodAutoscaler
apiVersion: autoscaling/v2beta1
metadata:
name: test-autoscaler
spec:
scaleTargetRef:
apiVersion: apps/v1beta1
kind: Deployment
name: metric-autoscaler
minReplicas: 1
maxReplicas: 4
metrics:
- type: External
  external:
  metricName: test-custom-metric
  targetAverageValue: 2
```

Here, we configure the autoscaler to a Deployment object, and direct it to modulate the number of pods based on our previously defined custom metric and a target value. Note that the ```metricName``` property in the HPA yaml must match the ```name``` property specified in the ExternalMetric yaml.

Once we apply these yamls to our Kubernetes resources, we can now assess the state of our HPA with ```kubectl get hpa``` to see how the current metric value compares to the target value, and the corresponding replicas that have been created as a result.

```
NAME              REFERENCE                      TARGETS     MINPODS   MAXPODS   REPLICAS   AGE
test-autoscaler   Deployment/metric-autoscaler   0/2 (avg)   1         4         1          161m
```

Google Kubernetes Engine (GKE) and Azure lets you do similar custom metric autoscaling with their managed Kubernetes services.

&nbsp;

## Going cloud-agnostic (if you require the flexibility)
Maybe your company has committed to one cloud provider – they have built their infrastructure assuming capabilities of that cloud provider, and systems talk to teach other easily because it assumes a certain API contract. Then tying the autoscaling to that cloud provider’s metric store may make sense. If not, you can achieve this same custom metric autoscaling functionality with a third-party metric store, like Prometheus or Datadog. There are various Kubernetes HPA adapters that you can employ to use the metric store of your choice.

With custom metric autoscaling, there are all sorts of creative ways you can start to scale your Kubernetes deployments – the rest is up to you!

&nbsp;

**References**:
1. <https://www.openshift.com/blog/kubernetes-autoscaling-3-common-methods-explained>
2. <https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/>
3. <https://github.com/awslabs/k8s-cloudwatch-adapter>
4. <https://github.com/kubernetes/metrics/blob/master/IMPLEMENTATIONS.md#custom-metrics-api>
5. <https://aws.amazon.com/blogs/compute/scaling-kubernetes-deployments-with-amazon-cloudwatch-metrics/>
