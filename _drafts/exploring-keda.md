---
layout: post
title: Exploring KEDA
subtitle: A brief look at KEDA using Kafka triggers
gh-repo: Cottonglow/kafka-integration-with-keda
gh-badge: [star, fork, follow]
tags: [keda, kafka, kubernetes, research]
comments: false
published: true
---

With KEDA releasing version 1, it is a good time to have a quick look at what it is and what it does!

In this post I will investigate the basics of KEDA using a Kafka trigger and look at what the properties mean and how it affects the scaling of your pods.

All the sample code used in this post is also available in this <a href="https://github.com/Cottonglow/kafka-integration-with-keda">GitHub repository</a>.

# KEDA 

So what actually is KEDA? KEDA, Kubernetes-based Event Driven Autoscaler, is an MIT licensed open source project from Microsoft and Red Hat that aims to provide better scaling options for your event-driven architectures on Kubernetes.

Let's have a look at what this means. 
Currently on Kubernetes, the HPA (Horizontal Pod Autoscaler) only reacts to resource-based metrics such as CPU or memory usage or custom metrics. From my understanding, for event-driven applications where there could suddenly be a stream of data, this could be quite slow to scale up. Never mind scaling back down once the data stream is lessening and removing the extra pods.  
I imagine paying for those unneeded resources all the time wouldn't be too fun!

KEDA is more proactive. It monitors your event source and feeds this data back to the HPA resource. This way, KEDA can scale any container based on the number of events that need to be processed, before the CPU or memory usage goes up.
You can also explicitly set which deployments KEDA should scale for you. So, you can tell it to only scale a specific application, e.g. the consumer.

As KEDA seems to be able to just slot in to your cluster, it seems quite flexible on how you want to use it. You don't need to do a code change and you don't need to change your other containers. It only needs to be able to look at your event source and the deployment(s) you are interested in scaling.

That felt like a lot of words! Let's have a look at this diagram for a high-level view of what KEDA does.

![KEDA](/img/exploring-keda/Keda.PNG)

KEDA monitors your event source and regularly checks if there are any events. When needed, KEDA will then activate or deactivate your pod depending on whether there are any events. KEDA also exposes metric data to the HPA which handles the scaling to and from 1.

This sounds straightforward to me! Let's have a closer look at KEDA now.

## Deploying KEDA

The instructions for deploying KEDA are very simple and can be found on <a href="https://keda.sh/deploy/"> KEDA's deploy page </a>.

There are two ways to deploy KEDA into your Kubernetes cluster:

1. Helm
1. Deploy yaml 

So what gets deployed? By the looks of it, the deployment contains the KEDA operator, roles and role bindings and these custom resources:

* ScaledObject

  The ScaledObject maps an event source to the deployment that you want to scale.
* TriggerAuthentication

  If required, this resource contains the authentication configuration needed for monitoring the event source.

The scaled object controller also creates the HPA for you.

## ScaledObject Properties

Let's take a closer look at the ScaledObject.

This is a code snippet of the one I used in my sample repository.

{% highlight yml linenos %}
apiVersion: keda.k8s.io/v1alpha1
kind: ScaledObject
metadata:
  name: consumer-scaler
  labels:
    deploymentName: consumer-service
spec:
  scaleTargetRef:
    deploymentName: consumer-service
  pollingInterval: 1
  cooldownPeriod:  60
  minReplicaCount: 0
  maxReplicaCount: 10
  triggers:
    - type: kafka
      metadata:
        topic: messages
        brokerList: kafka-cluster-kafka-bootstrap.keda-sample:9092
        consumerGroup: testSample
        lagThreshold: '3'
{% endhighlight %}

{: .box-warning}
The ScaledObject and the deployment referenced in `deploymentName` need to be 
in the same namespace.

So let's look at each property in the spec section and see what they are used for.

```yml
scaleTargetRef:
    deploymentName: consumer-service
```
This is the reference to the deployment that you want to scale. In this example, I have a `consumer-service` app that I want to scale depending on the number of events coming through to Kafka.

```yml
pollingInterval: 1 # Default is 30
```
The polling interval is in seconds. This is the interval in which KEDA checks the triggers for the queue length or the stream lag.

```yml
cooldownPeriod:  60 # Default is 300
```
The cooldown period is also in seconds and it is the period of time to wait after the last trigger activated before scaling back down to 0.

But what does activated mean and when is this? Having a look at the code and the docs, activated seems to be when KEDA last checked the event source and found that there were events, this sets the trigger to active.  
The next time KEDA looks at the event source and finds it empty, then the trigger is set to inactive and then kicks off the cool down period before scaling down to 0.

This could be interesting to balance with the polling interval to make sure it doesn't scale down too fast before the events are done being consumed!

```yml
minReplicaCount: 0 # Default is 0
```
This is the minimum number of replicas that KEDA will scale a deployment down to. 

```yml
maxReplicaCount: 10 # Default is 100
```
This is the maximum number of replicas that KEDA will scale up to.

```yml
triggers:
    - type: kafka
```
This is the list of triggers to use to activate the scaling. In this example, I use Kafka as my event source.

# Kafka Trigger

Although KEDA supports multiple types of event sources, we will be looking at using the Kafka scaler in this post.

{% highlight yml linenos %}
triggers:
    - type: kafka
      metadata:
        topic: messages
        brokerList: kafka-cluster-kafka-bootstrap.keda-sample:9092
        consumerGroup: testSample
        lagThreshold: '3'
{% endhighlight %}

## Kafka Trigger Properties

```yml
topic: messages
```
This is the name of the topic that you want to check the events in.

```yml
brokerList: kafka-cluster-kafka-bootstrap.keda-sample:9092
```
Here you can list the brokers that KEDA should monitor on. Looking at the code, this seems to be a comma separated list.

```yml
consumerGroup: testSample
```
This is the name of the consumer group.

```yml
lagThreshold: '3' # Default is 10
```
This one actually took me a while to figure out, but that is probably down to my inexperience in this area!  
From the documents this is described as how much the event stream is lagging. So, I thought it was something with time.

In reality, the lag refers to the number of records that haven't been read yet by the consumer.  
KEDA checks against the total number of records in each of the partitions and the last consumed record. After some calculations, this is used to identify how much it should scale the deployments.

{: .box-warning}
For Kafka, the number of partitions in your topic affects how KEDA handles the scaling as it will **not** scale beyond the number of partitions you requested for your topic.

# Example
So, what does this look like in practice? In the sample repository you can find a very simple consumer service using Kafka as the event source. We will be using this to experiment with KEDA.

If you want to try along as you read, you can find the instructions to start up the services in the README.

Let's start!  

The repository contains the Kafka and Zookeeper servers, a basic consumer service that simply outputs the messages from the Kafka topic and our KEDA scaler.

Here is how the keda-sample namespace looks like before KEDA is started:

{% highlight bash linenos %}
$ kubectl get all -n keda-sample
NAME                                                 READY   STATUS    RESTARTS   AGE
pod/consumer-service-5887df99d7-hgcnc                1/1     Running   0          15s
pod/kafka-cluster-entity-operator-784dbf5d5f-nkqz2   3/3     Running   0          29s
pod/kafka-cluster-kafka-0                            2/2     Running   0          54s
pod/kafka-cluster-zookeeper-0                        2/2     Running   0          78s

NAME                                             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/consumer-service                         ClusterIP   10.98.226.160    <none>        8090/TCP                     15s
service/kafka-cluster-kafka-0                    NodePort    10.104.101.0     <none>        9094:32000/TCP               54s
service/kafka-cluster-kafka-bootstrap            ClusterIP   10.106.255.132   <none>        9091/TCP,9092/TCP,9093/TCP   54s
service/kafka-cluster-kafka-brokers              ClusterIP   None             <none>        9091/TCP,9092/TCP,9093/TCP   54s
service/kafka-cluster-kafka-external-bootstrap   NodePort    10.97.47.72      <none>        9094:32100/TCP               54s
service/kafka-cluster-zookeeper-client           ClusterIP   10.100.96.220    <none>        2181/TCP                     78s
service/kafka-cluster-zookeeper-nodes            ClusterIP   None             <none>        2181/TCP,2888/TCP,3888/TCP   78s

NAME                                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/consumer-service                1/1     1            1           15s
deployment.apps/kafka-cluster-entity-operator   1/1     1            1           29s

NAME                                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/consumer-service-5887df99d7                1         1         1       15s
replicaset.apps/kafka-cluster-entity-operator-784dbf5d5f   1         1         1       29s

NAME                                       READY   AGE
statefulset.apps/kafka-cluster-kafka       1/1     54s
statefulset.apps/kafka-cluster-zookeeper   1/1     78s
{% endhighlight %}

You can see that there is one pod for the consumer-service currently active.

So, what happens after you start up the KEDA scaler?

{% highlight bash linenos %}
$ kubectl get all -n keda-sample
NAME                                                 READY   STATUS    RESTARTS   AGE
pod/kafka-cluster-entity-operator-784dbf5d5f-nkqz2   3/3     Running   0          12m
pod/kafka-cluster-kafka-0                            2/2     Running   0          13m
pod/kafka-cluster-zookeeper-0                        2/2     Running   0          13m

...

NAME                                                            REFERENCE                     TARGETS             MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/keda-hpa-consumer-service   Deployment/consumer-service   <unknown>/3 (avg)   1         10        0          10s
{% endhighlight %}

You can see that the HPA spun up and the consumer-service pod disappeared.

# Jobs
KEDA doesn't just scale deployments, but it can also scale your Kubernetes jobs.

Although I haven't tried this out, it sounds quite interesting! Instead of having many events processed in your deployment and scaling up and down based on the number of messages needing to be consumed, KEDA can spin up a job for each message in the event source.  
Once a job completes its single message, it will terminate.

You can configure how many parallel jobs should be run at a time as well, similar to the maximum number of replicas you want in a deployment.

{: .box-note}
KEDA offers this as a solution to handling long running executions as the job only terminates once the message is completed as opposed to deployments which terminate based on a timer.

# Resources

* <a href="https://keda.sh"> KEDA website </a>
* <a href="https://github.com/kedacore/keda"> KEDA GitHub<a>
* <a href="https://kafka.apache.org/"> Kafka website </a>