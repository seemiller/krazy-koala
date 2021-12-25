---
title: "Rails From the Closet to Kubernetes - Scaling"
date: 2021-12-25T16:16:14-07:00
draft: true
---

# Rails - From the Closet To Kubernetes - Scaling

Part 2. Continuing the story from [Deploying Rails from the Closet to Kubernetes - Deploymnet](https://example.com). Way back in 2007, our application, Pivotal Tracker, was deployed to a bare metal server in a closet in the office. This single server hosted the Rails application and the MySQL database. This situation was obviously not ideal, at least not to today's standards. One server, and a noisy one I came to later find out, in a closet. Before we put a label on it, it was sometimes powered off by those who couldn't take the noise and who did not know it was hosting Tracker. Those few times it was powered off, we scaled to, well, zero!

Thinking about it, scaling to zero isn't such a bad thing. Why pay for computing resources if you don't need to? On the flip side, the reverse is also true. If more resources are needed, wouldn't it be great if more servers could be spawned to handle the increased load? Back in the day with only one server, while we could technically scale to zero, obviously not in an ideal fashion, we had no means to easily scale up either. Scaling up meant upgrading hardware, or trying to add additional servers. The additional servers would then require a load balancer, which is even more hardware and complexity - [not gonna happen](https://www.youtube.com/watch?v=4QHHGHve_N0). Our easy out for the time was that Tracker was still primarily an internal application. So we didn't need to scale much.

Fast-forward a couple of years to 2010, we got serious again about moving Tracker to a SaaS model. This meant time to move out of the server closet and off that single server. Time to start scaling up. Our first move was to EngineYard, which at the time was having some scaling problems of their own. We experienced slow databases, frequent network problems and outright outages. EngineYard worked with us to remediate our problems, but it was death by a thousand cuts, and we needed to move.

We shortly found a home in a hybrid Cloud VM/bare metal world at BlueBox. All of our production virtual machines ran on our own 4, dedicated bare metal servers. No noisy neighbors to worry about expect for ourselves. Our database had loads of CPU, tons of memory and SSD RAID Array. The network was blazing fast too. BlueBox was our home for 5 years.

Having these 4 servers meant we could scale up, and get some redundancy in the process. So on each bare metal server, we ran virtual machines for:

- Rails application (Nginx/Passenger)
- Ordered Delayed Job process
- Unordered Delayed Job process
- Solr Cloud node for search functionality

There were also 2 dedicated servers for Memcached, a utility VM for random maintenance work, and a couple of bastion servers that all of our hardware was secured behind.

BlueBox was also the first time and place were we had an environment other than production. An entire bare metal server hosted virtual machines for a Staging environment. And we also spun up demo and beta environments too. It was great finally having a place to preview code, find bugs snd performance test outside of prod! The performance of Pivotal Tracker at BlueBox was fantastic.

The forced move to Amazon Web Services and CloudFoundry was quite a departure from what we had at BlueBox. In my opinion, CloudFoundry was way overkill for what we needed. The infrastructure required to just to run a CloudFoundry instance dwarfed what we had at BlueBox. The ulterior motive for running on a CloudFoundry instance was of course to dog food it. So we rolled with it since it was helping out the company with that need.

We quickly discovered that in order to keep the same level of performance that we had at BlueBox, we needed more virtual machines. A lot more. Twenty to thirty, instead of 4, to handle our peak loads. That's quite a bit of computing resources. Did we need that all the time? Of course not, just for the busy parts of the week. Did we have a means to reduce our VM count when traffic was light on the weekends? Not really, unless we manually reduced the virtual machine count. But then we'd have to manually increased it for Monday mornings. So, we settled on just running a (for us) large number of virtual machines, and paying more than double per month from our old BlueBox environment. This helped speed a move to the more cost-effective Google Cloud Platform.

While our bill did go down with the GCP move, we kept tweaking our deployments. By this point, we had a pretty robust deployment pipeline set up and running with [Concourse](https://concourse-ci.org/). One of our long range goals was to get to a continuous deployment, with no downtime deploys. To do this, we  implemented a [Blue/Green](https://martinfowler.com/bliki/BlueGreenDeployment.html) deploy.

The Blue/Green deployment was a godsend. It allowed us to roll out code whenever we wanted without impacting our customers. On the surface it was great, but underneath it had one big problem - resources. Remember that need for 20-30 VMs? Well, now we needed resources to handle 40-60 virtual machines. The Blue/Green deployment required us to fully deploy all `Green` VM's before switching the network route away from `Blue`. We also needed to keep `Blue` around in case of the need to rollback. And why keep it around instead of just redeploy? Our deployment times on CloudFoundry took in the range of 20-30 minutes. If there was a problem, we needed to move quick, and a 20+ minute deploy was no good. In the end, half of the pool of resources allocated to VM's for the web tier sat idle.

So how can moving to Kubernetes help us with scaling our deployments? First, the Kubernetes deployment itself. We can specify the number of replicas to run, and we can very easily scale it up or down with a `kubernetes scale deployment/...` command. Deployments under Kubernetes offer a rollout history. This history allows us to jump back to any known good version, or just quickly undo the last. Kubernetes is also able to more efficiently start up and terminate pods in the deployments, allowing for quicker switches between blue/green deploys. In the end - much, much quicker deploy times with the built-in ability to quickly rollback or manually scale our app up or down.

Let's talk about that ability to scale up or down. Having that ability is great, but it's manual. Wouldn't it be great if Kubernetes could just scale for us automatically? Kubernetes has something called the [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/). Sounds like it might do what we want, but it's still in beta. If I was to start Pivotal Tracker all over again now, I would look to something like [Knative](https://knative.dev/docs/). Knative has the ability to scale the pods running the application based on their load. The higher the load and requests, the more pods to handle them. In the quiet times, the pods can be scaled down completely, with the application not even running. If only something like Knative was available back in the day.

Knative also supports revisions. Revisions are basically the same as a deployment rollout history. As you modify and change your application, Knative will track the changes so that you can see your history and even rollback if needed. Revisions are also helpful when rolling out new containers as it lets you perform blue/green deploys. You can even configure canary builds with traffic throttling.

A Rails app is an ideal application for deployment and scaling with Knative. The [criteria](https://knative.dev/docs/serving/convert-deployment-to-knative-service/#determine-if-your-workload-is-a-good-fit-for-knative) for this is:

- All actions initiated by an HTTP request.
- Stateless - Rails applications don't necessarily need to track state locally, that's what the database is for!
- Do not require local storage - Rails applications can get all configuration from ConfigMap and Secret.

In this post I want to show how to set up a Rails application running on Kubernetes with Knative serving demonstrating the scale to zero feature. I've already created a Rails application that we can use for this, it's the Rails Blog application that is detailed in the [Ruby on Rails - Getting Started](https://guides.rubyonrails.org/v6.0/getting_started.html) guide. I've already followed the guide and committed it to a GitHub [repository](https://github.com/seemiller/rails-blog). This demonstration will be building off of the Tanzu Community Edition cluster that was deployed to AWS in [Part 1](https://example.com).

## Setup and Installation

If you haven't already cloned that repo, do that now, or use your own.

```shell
git clone git@github.com:seemiller/rails-blog.git
cd rails-blog
```

Before we install Knative, we need to first install [Contour](https://projectcontour.io/) as a prerequisite. Contour is an open source [ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) provider. I'm using still using Tanzu Community Edition, so I'll install the Contour package.

```shell
tanzu package install contour \
  --package-name contour.community.tanzu.vmware.com \
  --version 1.18.1
```

Since I am going to use a [custom domain](https://knative.dev/docs/serving/using-a-custom-domain/), `k8squid.com`, I'll need to create a configuration file for the knative-serving package and configure the Route53 DNS.

Create the configuration file.

```shell
cat knative-values.yaml <<EOF
domain:
  type: real
  name: k8squid.com
```

Get the external IP of the Envoy Load Balancer. On Route 53's Hosted Zone page, confgure the `A` record for your domain to use this IP.

```shell
> kubectl get service --namespace projectcontour

NAME      TYPE           CLUSTER-IP       EXTERNAL-IP                                                              PORT(S)                      AGE
contour   ClusterIP      100.66.15.181    <none>                                                                   8001/TCP                     3d7h
envoy     LoadBalancer   100.69.190.187   a7d4a55e...67870.us-east-1.elb.amazonaws.com   80:30949/TCP,443:32571/TCP   3d7h
```

With our prerequisites out of the way, we can install the Knative package.

```shell
tanzu package install knative-serving \
  --package-name knative-serving.community.tanzu.vmware.com \
  --version 1.0.0
  --values-file knative-serving.yaml
```

In another terminal, watch the knative-serving namespace to see the pods come up.

```shell
kubectl get pods --namespace knative-serving --watch
```

I still have my Rails blog application deployed from the previous walk-through. I'll need to remove that deployment, ingress and service before deploying the application with Knative.

```shell
kubectl delete --filename kubernetes/rails-deployment.yaml
kubectl delete --filename kubernetes/rails-ingress.yaml
kubectl delete --filename kubernetes/rails-service.yaml
```

## Knative Service Deployment

With the original deployment cleaned up, we can deploy the application using Knative. For that we need a Knative service manifest. It does seem a bit odd to be deploying our application in a service instead of a deployment, but that's just how it is. This is basically our original deployment, now in a Knative manifest.

```shell
> cat kubernetes/rails-knative.yaml
```
```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: rails-blog
  namespace: blog
spec:
  template:
    metadata:
      name: rails-blog-0001
    spec:
      containers:
        - name: rails
          image: ttl.sh/seemiller/rails-blog:1d
          ports:
          - containerPort: 3000
          env:
          - name: DATABASE_URL
            valueFrom:
              secretKeyRef:
                name: rails-secrets
                key: database-url
          - name: SECRET_KEY_BASE
            valueFrom:
              secretKeyRef:
                name: rails-secrets
                key: secret-key-base
```

> Notice that the `.spec.template.metadata.name` field has a `-0001` suffix. This is the revision of the service. The revision can be used to track the different versions of your application.

Have 2 terminal windows open - 1 to deploy the service and another to watch pods.

```shell
# second terminal
kubectl get pods --namespace=blog --watch
```

```shell
# primary terminal
kubectl apply --filename=kubernetes/rails-knative-service.yaml
```

Notice that Knative will deploy 2 pods. Wait a few minutes and notice that the pods are terminated. Let's get the Knative service to see how to make a request to our Rails application. Notice that the URL for our application is the name of the service, the namespace, and finally our custom domain name.

```shell
> kubectl get ksvc --namespace blog

NAME         URL                                       LATESTCREATED     LATESTREADY       READY   REASON
rails-blog   http://rails-blog.blog.k8squid.com        rails-blog-0001   rails-blog-0001   True
```

Hit that URL with a browser or a curl command. Wait 30 seconds and see the pods terminate. Repeat your requests to the application and watch the pods come back to life!

## Conclusion

In my opinion, having a service that is able to spin up an app on demand and spin it down when idle is pretty amazing. Instead of wasting computing resources on idle web servers, the system can balance itself out and be available for other things. Add in the ability to manage revisions and perform blue/green/canary deploys and you've got a pretty solid deployment service. Knative could have gone a lng way in helping us solve our scaling issues and perhaps simplified our deployment pipeline. If Knative serving is at all intriguing, I would recommend a review of their official [documentation](https://knative.dev/docs/serving/) or take a look at the [Knative: Operator’s Handbook](https://knative.tips/pod-config/volumes/) for other examples.
