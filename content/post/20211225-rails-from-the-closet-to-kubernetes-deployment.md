---
title: "Rails - From the Closet To Kubernetes - Deployment"
date: 2021-12-25T15:43:53-07:00
draft: true
---

# Rails - From the Closet To Kubernetes - Deployment

From mid-2007 until early 2020, I helped to develop and maintain a Ruby on Rails application, Pivotal Tracker. I got to see almost the entire lifecycle of this application. When I joined the Tracker team at Pivotal Labs in mid 2007, the app already had its core functionality, but it needed its edges rounded out. For our use as an internal project management application, those rough edges were ok, but we worked to round them out and make it a full-fledged product.

Pivotal Tracker was wildly popular within the company, and our clients loved using it too. Almost too much. A common question when a client engagement neared its end was "Can we continue to use Tracker?". Or when they told their friends about it and then the friends called up asking if they could use it. That was a difficult question to answer, as the answer was always no. Tracker was part of our secret sauce and was for internal use only or while there was an active engagement with a client. We also lacked a number of account management features, settings and permissions. Tracker performed its core project management mission admirably, but yeah, rough edges prevented us from allowing outside or public use.

So we set out to smooth those rough edges. Plans were made to add those missing account management features so that we could safely allow outside use without fear of someone accessing a client project. We also added a subscription service so that we could charge users for use of the application. Companies have to keep the lights on somehow right?

## In the Closet

The entire Pivotal Tracker application was running on a single server. In the closet. And a noisy server as we came to find out, but that's the subject for another story. That single server hosted the Rails application, Apache and the MySQL database. Background jobs, delayed jobs, and maintenance scripts also had to run there as well.

Running an application on a single server is easy. There's 1 server, 1 process, 1 database, 1 DNS record and route. Everything is simple. That is until the performance starts to degrade, or the server goes down. Adding another server complicates things. Suddenly you'll need more infrastructure and process to run the application - load balancers, network routers and switches. The database should have its own server. And yes, backups should be stored on another server or the company NAS. All this means more complexity and expense and people to keep it all running.

Building an application for internal use is easy. Launching it to the public is hard. Opening up Tracker to the public as a subscription service brought a lot of baggage. If we're taking people's money, there is now an expectation that we will actually deliver the service they're paying for. I think reality started to set in.

The idea to create a subscription service was scraped. Instead, the app would be opened to the public for free. This allowed us to sleep easy at night. And since no one was paying for the service, we technically didn't have to provide any sort of uptime expectations. I'll admit that it was a bit disappointing to not flip the switch on the subscription service, but in hindsight, it was genius. With the application now free, the public started to discover it by word of mouth, and it helped the company further establish itself and Pivotal Tracker's popularity.

Pivotal Tracker continued on for another 2 years on that single server. During this time I left the company, but I got pulled back in early 2010. Tracker continued to grow in popularity, and it was time to revisit that subscription service. Everything was still running on that one box, and that was a problem. We needed a better hosting solution. At this time, EngineYard was focusing on Ruby on Rails deployments, and a lot of our clients started to host with them. So, it seemed a natural fit.

## Out of the Closet

Migrating to EngineYard was pretty straight forward, except for an issue using a CDN. Just update the Capistrano deployment configuration and we're deploying to EngineYard. Migrating the database was a snap. Even after 5 years, the size of the database was small enough that we were able to perform a dump and load within a few hours of downtime. Wow! Downtime! When did you last take your application __all the way down__ for maintenance?

EngineYard seemed promising at first, but we never really settled in. We were constantly getting plagued by extremely long response times and seemingly random service outages.

So the long response times was a problem on both us and the new deployment at EngineYard. When our application loaded its project page, it queried the database for all stories, comments, commands and user memberships for the project. This operation is pretty expensive and we were constantly poking and tweaking at it to improve performance. Our long term solution for that problem was memcached, but that's another story. So we already knew about the performance issues with the project load, but why was it such a problem at EngineYard? Turns out our database was running on a shared database server that itself was a virtual machine. Part of me died that day. There's no way we would survive with a shared database.

There was still the issue of the seemingly random outages. Two to 4 times a week, for a minute or so, our application would just stop - wouldn't handle requests; open terminals to the virtual machines would just hang. Then everything would just start working again. This went on for a few weeks and was becoming quite a problem. We inquired with some client projects that were deploying to EngineYard so see if they were experiencing similar issues. No one was. Our Product Manager fortunately started to notice a pattern. See, he was also managing an engagement for an up and coming coupon company called Groupon. At the time, Groupon was also deploying to EngineYard. He noticed that every time Groupon did a deployment, our application would get hit with an outage. Do a little digging, and we find out that Groupon's footprint is considerable larger than ours, and our virtual machines are sharing the same bare metal and network gear as Groupon. Once again, virtual machines running on shared servers bites us.

## Hybrid Cloud

These experiences left a sour taste for infrastructure where our virtual machines co-mingled with other "noisy neighbors". Unfortunately, this meant we started the search for a new hosting provider again. It wasn't long before we found a shop out of Seattle, Washington, called BlueBox. I don't recall what specifically drove us to chose them, but they had a hosting model to our liking. We could create an environment that was a hybrid between virtual and bare-metal servers. Application and utility servers were all virtual. We could easily spin up others for memcached, delayed jobs, and easily create demo and staging environments. The bare metal servers hosting all of those virtual machines were ours - no more noisy neighbors. Much to my delight, the database was also back on bare-metal.

BlueBox was a fantastic place to call home. The servers were fast and reliable. Their support was top-notch too. They had a small staff, but frequently helped us out with some code or database optimizations when we got stuck. We basically settled in for 4 or 5 years. This comfort did have a bit of a downside - we missed out on the beginnings of the Cloud era, and all the innovations that came with it.

There were a few factors that contributed to our eventual move to Amazon Web Services. In 2013 Pivotal Labs was acquired by EMC. My TL;DR of the acquisition was this: A client, Greenplum, which was owned by EMC, had such a favorable experience with us and was so enamored with our process and culture, convinced EMC to buy us out. The real motivator behind this was that EMC had a struggling engineering project that needed help. That was CloudFoundry. Pivotal Labs took over development of CloudFoundry and made it what it is today. Other companies contributed heavily to CloudFoundry as well, such as IBM. In 2015, IBM bought BlueBox, our much beloved hosting provider. We also started having some issues with our database server, and BlueBox was hesitant to provision a new one. After months of questioning, it came out that BlueBox was being directed to shut down its traditional hosting services (that we were still using) and migrate all customers to IBM's flavor of CloudFoundry. This was certainly a problem. Pivotal Labs' flagship application running on a competitor's version of CloudFoundry? I think not. This spurred a migration effort, Project ExBox, to get out of IBM BlueBox and on to a Pivotal CloudFoundry instance running on AWS.

## AWS

The migration to AWS was arduous. We advertised 5 months, but in reality I think it was 9+. There were so many places in the code that needed tweaking to run on a platform like CloudFoundry. This meant massive amounts of `if... else...` statements littering the code. We had to overhaul our continuous integration setup. Institute pipelines for testing, deploying, creating containers, running migrations, backups. Crash course in containerization as everything needed to run in a container. BOSH became our new worst nightmare. We had to migrate our (now multi-100GB) database to RDS. New DNS and TLS certificates. Load and performance testing. It was either a DevOps Engineer's worst nightmare or dream job. It took our team months to make all the changes and verify our migration plan. Then one donut fueled morning in October 2016, we took Tracker down for maintenance and made the move.

From the moment that we moved to AWS, there was pressure to move off it. Turns out running a CloudFoundry instance on AWS was not cheap, something like $75-100K a month. That was a significant bump in our hosting costs. Unfortunately, it was the easiest place to run CloudFoundry. AWS seemed to be the happy path for CloudFoundry development, make it work there first. This expense had us playing games like using spot instances and shutting down our CI pipelines for the night and weekends. Personally, this was a good time as I was learning about AWS and running applications in the Cloud. So this was now more of a dream job instead of a nightmare.

## GCP

About this time, Pivotal started to partner with Google to make CloudFoundry compatible with Google Cloud Platform. This partnership hatched the plan to accelerate a move off of AWS. THe idea was that Pivotal Tracker could dog food CloudFoundry on GCP. And our new Google partnership came with some partner cost savings. So with just a few months since our move to AWS, we started another migration project.

The migration to GCP was considerably easier than the move to AWS. Since Tracker was now running on CloudFoundry, in theory, we were insulated from the underlying cloud platform and hardware. As long as CloudFoundry had the drivers and providers, Tracker should work just fine. For the most part, in theory translated into in real life. We only needed 1-2 months of engineering time to made changes specific to GCP and validate that Tracker would run. Toss in another month or 2 for load and performance testing and data migration. By the end of July 2017, Tracker was running on GCP.

We settled in with Google Cloud Platform quite well. Costs were under control. Servers, virtual machines and databases all provided increased performance. But the network! Google's worldwide fibre network was amazing! Latency, response times and basically anything network related saw significant increases in performance. The GCP move also came with the added partnership of our CloudOps team in Dublin, Ireland. CloudOps EU had agreed to take ownership of running the CloudFoundry foundation, allowing us to get back to focusing on just Tracker. Managing a CloudFoundry installation is a lot of work!. This was a real win, but not just for the Tracker Team. CloudOps was able to provide Pivotal with a live, production testbed for new releases and processes for managing a CloudFoundry installation. This arrangement provided an enormous amount of benefit to Pivotal Tracker, CloudFoundry, and all of our clients who benefited from the innovations that the CloudOps made while maintaining the foundation.

However, that arrangement also spelled the end of Pivotal Tracker running on CloudFoundry. This is also the final step in our journey to Kubernetes. In the intervening years, EMC was acquired by Dell, and then Pivotal was IPO'd back out on its own. However, we were still majority owned by Dell. I'm certainly not going to claim that I understand how corporations work when it comes to earnings, debt and taxes, but a decision was made to have VMware, also majority owned by Dell, acquire Pivotal. This had some bad news for us.

## Kubernetes

One of the outcomes of the VMware acquisition was the closure of the Dublin, Ireland office. This is where the CloudOps team was based. No office meant no CloudOps team. No CloudOps team meant a big question as to who would be managing our CloudFoundry installation. The situation degraded even quicker than we expected. Most of the CloudOps team was focused on figuring out their future, with or without VMware, so they we understandably distracted. And remember I mentioned that we were dog-fooding new releases of software for CloudFoundry? Well, a bug was introduced into a piece of networking software that messed with DNS resolution. The end result of this to us was that we could no longer deploy new releases. Some valiant attempts to rectify the situation were made by the remaining CloudOps team members but to no avail. The picture was clear to us though. The Tracker Team simply did not have the expertise to manage that CloudFoundry foundation and we needed to move, again.

At this point, in early 2020, our path was pretty clear. Kubernetes was eating CloudFoundry's lunch on a daily basis. All the momentum was pointing us to make the move to Kubernetes. I joked that I would only be a part of the migration if we moved to Azure, so that I could get the Trifecta of the big 3 cloud providers. Having everything already running on GCP was part of what made the move to Google Kubernetes Engine so easy - no database to migrate - no major DNS or networking changes - app already coded to run in the Cloud. The majority of the initial work was to change the pipelines to deploy to Kubernetes instead of CloudFoundry, and that work was pretty quick, maybe a couple weeks? I seem to recall going from zero Kubernetes to production Kubernetes deployments in about 6 weeks. The move to Kubernetes was my final work with Pivotal Tracker. From a server in the closet to Kubernetes on GCP.

I was really amazed at just how easy it was to use Kubernetes. And the community around Kubernetes astounded me as well. All of these services and software community built. That's what eventually led me to help open sourcing VMware's flavor of Kubernetes. To be a part of that community and to give back.

## Deploying a Rails App

So if I were to start this story all over again now, I would most definitely start it with Kubernetes. Not sure if I would stick with Rails, but at this point, it's what I know. So I'd like to show deploying a Ruby on Rails application to Kubernetes.

So let's dive in and see what we need to deploy a Ruby on Rails application on Kubernetes.

- Rails app
- Container
- Kubernetes cluster
- Database Deployment
- Application Deployment

In developing this guide, I used the following hardware, software, and services.

- Apple Macbook Pro
- Amazon Web Services
- ttl.sh
- MySQL 8.0.27
- Rails 6.1.4.1
- Ruby 3.0.2
- VMware Tanzu Community Edition

### Rails Application

To start, we need a Rails application. We can either create one from scratch, or use an existing application. To keep things accessible and simple, we'll just use the Blog application built in the [Ruby on Rails - Getting Started](https://guides.rubyonrails.org/v6.0/getting_started.html) guide. I've already followed the guide and committed it to a GitHub [repository](https://github.com/seemiller/rails-blog). So you can bring your own app, or just clone this one. I did deviate from the guide by using a MySQL database instead of SQLite. Having a separate database makes for a more thorough demonstration.

```shell
git clone git@github.com:seemiller/rails-blog.git
```

Or if you're creating a new application from scratch, here's the command I used to create the basic application. You'll be on your own to create a scaffold or other components.

```shell
rails new rails-blog --minimal --database mysql --webpack=react
```

### Container

Kubernetes is really nothing more than an extensible container orchestration platform. So we need to put our application into a container. We can do this with a Dockerfile. Change into the rails-blog directory and lets take a look at the Dockerfile.

```shell
cd rails-blog

cat Dockerfile
```

The Dockerfile should look like this.

```dockerfile
FROM ruby:3.0.2

RUN curl https://deb.nodesource.com/setup_12.x | bash
RUN curl https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add -
RUN echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list

RUN apt-get update && apt-get install -y build-essential \
                                         nodejs \
                                         yarn \
                                         vim \
                                         --no-install-recommends && \
    rm -rf /var/lib/apt/lists/*

ENV RAILS_ENV production
ENV RAILS_LOG_TO_STDOUT true

RUN mkdir /app
WORKDIR /app
COPY Gemfile ./
RUN gem install bundler
RUN bundle install
COPY . .

RUN rake assets:precompile

EXPOSE 3000
CMD ["rails", "server", "-b", "0.0.0.0"]
```

This is a pretty straight-forward Dockerfile. We start with the Ruby 3.0.2 base image and add in the needed things like node, yarn, and build tools. Some environment variables are set as well to indicate that this is our "production" environment and where to log Rails' output to. It is important to have Rails log to STDOUT as this is where Kubernetes expects all to output to go. Kubernetes can then pipe all this output into log monitoring services such as fluentd. Next we copy our application source code and run bundle to get our gems. In a full blown production pipeline this is a good place to "pre-bake" an image to speed things up. Assets are compiled and we expose port 3000 and give the command to run the server.

Build the image and push it up.

```shell
docker build --tag ttl.sh/seemiller/rails-blog:1d .

docker push ttl.sh/seemiller/rails-blog:1d
```

### Kubernetes Cluster

For my Kubernetes cluster, I'm going to use Tanzu Community Edition, from VMware. Guess I'm a bit biased as to the choice of this distribution, so I'll admit that I have made some contributions to this distro.

This post isn't about deploying Kubernetes. So to keep things on track, I'll refer you to the [Getting Started](https://tanzucommunityedition.io/docs/latest/getting-started/) documentation for deploying Tanzu Community Edition. I used a managed cluster on Amazon Web Services. You are of course free to use any distribution that you prefer or have easy, and possibly free, access to.

Once a cluster is up and running, create a namespace to contain the blog application. To help keep track of things, assign a label to the namespace as well.

```shell
kubectl create namespace blog
kubectl label namespace blog app=blog
```

An equivalent manifest file is located in the `kubernetes` directory that could be applied instead.

```shell
kubectl apply --filename kubernetes/namespace.yaml
```

Learn more about [namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) and [labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) in the official Kubernetes documentation.

### Database

If you have any experience at all with Rails, you know that you can't do anything without a database. Ok, you can serve static files, but there are better ways to do that than by running a Rails server. Personally, I'm not a huge fan of running a database on any type of virtual or containerized platform. In my opinion, a database should be installed on a big honkin' bare metal server with lots of CPU, lots of RAM and lots of disk. And if you can afford it, there's an identical machine setting next to it in the rack. Except this other server is on a different electrical circuit and network stack. If you can't have bare metal, opt for a service like AWS RDS or GCP CloudSQL. The managed database services will work pretty well for most use cases, and they come with bells and whistles like snapshots, backups, redundancies in different regions and zones. Pivotal Tracker started with MySQL running alongside the Rails application in the closet. At BlueBox we had [Percona](https://www.percona.com/software/mysql-database/percona-server) running on bare-metal, and it was awesome. AWS RDS and GCP CloudSQL rounded out our databases in the Cloud.

However, for this exercise, we'll deploy plain old MySQL to our cluster. Why MySQL and not Postgres or some other database? To be honest, it's what I've always used. Pivotal Tracker used it for years, and I used it before joining Pivotal Labs. It's what I know and am comfortable with. It might not be web scale, but it gets the job done. So that's what I'm going to deploy. It also helps that the Kubernetes documentation has a nice [example](https://kubernetes.io/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/) already worked up with MySQL, which I've drawn inspiration from.

In deploying our MySQL database, we need to declare three things to Kubernetes:

- Service
- Persistent Volume/Claim
- Deployment

Let's start with the service. The service will make the database application available to the other pods in the cluster. The selector `app: blog` will allow the pods in our future Rails deployment to map to the correct service. By specifying `clusterIP: None` we are disabling load-balancing and not allocating a cluster IP address. We will have to reference our service by DNS. MySQL's well known port is 3306, so we declare the service should use 3306.

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: blog
  labels:
    app: blog
spec:
  selector:
    app: blog
  clusterIP: None
  ports:
    - name: "mysql"
      port: 3306
      protocol: TCP
      targetPort: 3306
```

Let's create the service by applying the manifest.

```shell
kubectl apply --filename kubernetes/mysql-service.yaml
```

Learn more about [services](https://kubernetes.io/docs/concepts/services-networking/service/) in the official Kubernetes documentation.

### Persistent Volume Claim

A database needs a disk to write its data to. Otherwise, we might as well use /dev/null, which I hear is fast and web scale. Kubernetes provides disk via Persistent Volume Claims, or pvc's. Since this is just an example application, we don't need much, just a small disk that provides Read/Write access to a single node. We'll allocate the disk space here, and wire it up in the deployment. We will declare a persistant volume, and then a claim to that volume.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/var/lib/mysql"
  claimRef:
    name: mysql-pvc
    namespace: blog
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: blog
  labels:
    app: blog
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  volumeName: mysql-pv
```

> Note that we're using a hostPath in the Persistent Volume. This is really only good for single node clusters. If you really want this disk to work in a more production like environment, you'll want to use a different [type of persistent volume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#types-of-persistent-volumes) that aligns with your infrastructure.

Create the persistent volume claim by applying the manifest.

```shell
kubectl apply --filename kubernetes/mysql-storage.yaml
```

Learn more about [persistent volume/claims](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) in the official Kubernetes documentation.

### Deployment

The Deployment tells Kubernetes about the containerized application that you want to run. You can specify how many pods to run, environment variables, ports and volume mounts. Deployments can be scaled, rolled out, and rolled back. We'll specify the container image that we want to run, which for us is the MySQL 8.0.27 image.The persistent disk that was created in the previous step will be mapped here. The name of the PVC is used in the volumes specification, and we map it to a mount path that MySQL will use to store our database.

If you notice in the container spec, there is a reference to a secret for the root user password. In conjunction with creating this deployment, we need to create a secret. We'll need a generic secret created from a literal value. That value will get base64 encoded and stored in the Kubernetes etcd process. The deployment will inject this value into the environment of the container that is running the database. There are other ways and means of handling secrets, but that's a topic for another time.

Create the secret.

```shell
kubectl create secret generic mysql-pass --from-literal=password=super-secret-password --namespace=blog
```

```shell
kubectl get secret/mysql-pass --namespace blog --output yaml
```

```yaml
apiVersion: v1
data:
  password: c3VwZXItc2VjcmV0LXBhc3N3b3Jk
kind: Secret
metadata:
  name: mysql-pass
  namespace: blog
type: Opaque
```

> Notice how the value for the password is base64 encoded.

With the secret in place, we can create the deployment.

```shell
kubectl apply --filename kubernetes/mysql-deployment.yaml
```

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: blog
  labels:
    app: blog
spec:
  selector:
    matchLabels:
      app: blog
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: blog
    spec:
      containers:
      - image: mysql:8.0.27
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
```

Check the status of the deployment.

```shell
kubectl get deployments --namespace blog

NAME    READY   UP-TO-DATE   AVAILABLE   AGE
mysql   1/1     1            1           5m
```

When ready is `1/1` and available is `1`, the application should be up and running. Verify by taking a look at the logs for the MySQL application.

```shell
kubectl logs --namespace blog deployment/mysql

2021-10-23 05:01:36+00:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 8.0.27-1debian10 started.
2021-10-23 05:01:36+00:00 [Note] [Entrypoint]: Switching to dedicated user 'mysql'
2021-10-23 05:01:36+00:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 8.0.27-1debian10 started.
2021-10-23 05:01:36+00:00 [Note] [Entrypoint]: Initializing database files
2021-10-23T05:01:36.702659Z 0 [System] [MY-013169] [Server] /usr/sbin/mysqld (mysqld 8.0.27) initializing of server in progress as process 43
...
```

Looks like the database is up and running! We can move on to deploying the Rails application and configuring the database.

Learn more about [deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) and [secrets](https://kubernetes.io/docs/concepts/configuration/secret/) in the official Kubernetes documentation.

### Rails Deployment

We'll follow a similar process for deploying the Rails application as we did the MySQL database. We will need to define a deployment, service and secrets. In addition, an ingress will also need to be declared. The ingress will allow web traffic to be routed from outside the cluster to the Rails application.

#### Service

Let's start with the service. The Rails application is listening on port 3000 as was defined in the Dockerfile. So we need to map port 80 to port 3000. Once again, we'll specify a clusterIP of none.

You can create the service with the kubectl command, or apply the yaml file directly.

```shell
kubectl create service clusterip rails --namespace=blog --clusterip=None --tcp=80:3000
```

or just apply the existing manifest file.

```shell
kubectl apply --filename kubernetes/rails-service.yaml
```

The service manifest will look as follows:

```yaml
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: rails
  name: rails
  namespace: blog
spec:
  clusterIP: None
  ports:
    - name: 80-3000
      port: 80
      protocol: TCP
      targetPort: 3000
  selector:
    app: rails
  type: ClusterIP
```

#### Secrets

At a minimum, Rails needs a couple of secrets to function, the session key and the database URL.

Create a session key using the rails secret command.

```shell
rails secret | base64

ZDYwOTU0MzJlNDc3NjM5OGI4MDQ2ZTczODE3YmYxMzQ2NzlkNmI0ZDlmYjdkYjc5ODExMjli
MjAxNzc5OGMxNzZhNDA1Yjc4NDllNzUwOTgzMTU2YjM1ODNlZWY2ZDA0YWIyMjY2NjUyYTMw
ZWZlMmU3YjdjZTVlNDkyNDczZjEK
```

The database URL of course tells Rails how to find the database. You can specify individual parameters in the Rails database.yaml config file, or use a more succinct approach, the DATABASE_URL. The DATABASE_URL allows you to specify all the connection parameters in one. It has the following format:

```shell
mysql2://myuser:mypass@example.com/mydatabase
```

To fill out the DATABASE_URL, we'll need the root password that we created for MySQL earlier, and the DNS name for the MySQL service. Kubernetes provides DNS names for services running the cluster. It follows a convention:

```shell
<service-name>.<namespace>.svc.cluster.local
```

So in our situation, the DNS name for the MySQL database service is:

```shell
mysql.blog.svc.cluster.local
```

Using that DNS name, we can build our DATABASE_URL connection string.

```shell
echo "mysql2://root:super-secret-password@mysql.blog.svc.cluster.local/rails_blog_production" | base64
```

Learn more about [database configuration options](https://guides.rubyonrails.org/configuring.html#configuring-a-database) in the Rails documentation and [DNS for Services](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/) in the Kubernetes documentation.

Armed with our secrets, we can create a secrets manifest file and add them to the cluster.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: rails-secrets
  namespace: blog
type: Opaque
data:
  database-url: |
    bXlzcWwyOi8vcm9vdDpzdXBlci1zZWNyZXQtcGFzc3dvcmRAbXlzcWwuYmxvZy5zdmMuY2x1
    c3Rlci5sb2NhbC9yYWlsc19ibG9nX3Byb2R1Y3Rpb24K
  secret-key-base: |
    Y2U5NDE4OWZhZGYwYTRhZTg2ZmZkM2NlMjEyZjJkYzEyMzFkMjI5ZTNjNDcwMjk4OGJmMjRj
    MzAyMWNhZWJlNWI4ZTRmNWExZjQxM2JhYWVjMTU0MDRiZDE2MmNkYzZlY2M5NWZjMGQ1Njhi
    MGFmZTljMTY1NTY1NGMyNWU4NzUK
```

```shell
kubectl apply --filename kubernetes/rails-secret.yaml
```

> For a real production application you of course would not want to commit your secret values to your source control. There are better and more secure means to handle secrets that are the subject of another story.

#### Deployment

We're ready for the deployment now, and once again this is very similar to the MySQL deployment we created earlier. Pretty straight-forward deployment, the only things of note are exposing the application on port 3000 and adding references to our secret values.

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: rails
  name: rails
  namespace: blog
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rails
  template:
    metadata:
      labels:
        app: rails
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

Apply this deployment.

```shell
kubectl apply --filename kubernetes/rails-deployment.yaml
```

Wait for it to become available.

```shell
kubectl get deployments --namespace blog

NAME    READY   UP-TO-DATE   AVAILABLE   AGE
mysql   1/1     1            1           3d16h
rails   1/1     1            1           3d
```

Our Rails application is now deployed to Kubernetes!

#### Database Setup

As mentioned earlier, you can't do anything with Rails without a database. Unfortunately, I have yet to come across an elegant means of initializing the database. It feels crude, but a simple `exec` command will get us going. In a fully built out application with a CI/CD pipeline, you could run migrations in a Job as part of the deployment process, but that is overkill for us here at the moment.

Use kubectl to create the database and run the migrations.

```shell
kubectl exec deployment/rails -it --namespace blog -- rails db:create db:migrate
```

#### Ingress

The final step in making the Blog application available is to create an Ingress. An ingress creates a load balancer or endpoint that external processes can use to make requests of services running inside our cluster. Typically, this is for HTTP requests.

You can create an ingress on the fly using the kubectl command. In this command, we are mapping all requests to this ingress to port 80 of the `rails` service. If you recall, we created a service for the Rails app that listens on port 80 and targets port 3000 on any object with an `app: rails` label.

```shell
kubectl create ingress rails --namespace blog --rule="/*=rails:80"
```

This creates our ingress object.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rails
  namespace: blog
spec:
  rules:
  - http:
      paths:
      - backend:
          service:
            name: rails
            port:
              number: 80
        path: /
        pathType: Prefix
```

We can get the ingresses in our namespace to get the external IP address.

```shell
kubectl get ingress --namespace blog

NAME    CLASS    HOSTS   ADDRESS                                                                  PORTS   AGE
rails   <none>   *       a65e9ab6f9ea449149814be66c783071-339151308.us-east-1.elb.amazonaws.com   80      5m
```

Your IP address will vary depending on where your cluster is running. AWS provides a nice, unmemorable address that can easily be mapped to a domain name in their Route53 DNS service. In the spirit of keeping things simple, I'm going to save the details of updating DNS and implementing TLS for another story.

With the ingress created, copy/paste the address into your favorite browser and enjoy the sample Rails Blog application.

Learn more about [ingresses](https://kubernetes.io/docs/concepts/services-networking/ingress/) in the official Kubernetes documentation.

## Conclusion

We've come a long way from deploying apps on noisy servers running the closet. There's still something to be said for the simplicity of just racking a server, installing an OS, deploying some code, updating DNS and profiting. When you're just starting out, or for a toy/learning application that's a perfectly viable route. But eventually that model doesn't scale - especially if you're trying to grow a business around it. Having a platform like Kubernetes to build your application on makes a world of difference. With many proven patterns, software services, and a thriving user community to call upon for help, you can focus on your application and just know that the deployment is not a concern. As a DevOps/SRE/Full Stack Engineer, Kubernetes would have made my life so much easier on this journey from the closet to the Cloud.
