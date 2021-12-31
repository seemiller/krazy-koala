---
title: "Rails From the Closet to Kubernetes - SSL"
date: 2021-12-28T17:23:54-07:00
draft: true
---

# Rails - From the Closet To Kubernetes - SSL

Part 3. Building and maintaining a web application back in the day was a lot of work. There were of course standards and best practices, but actual implementation was often quite difficult or laborious. As an example, let us consider securing a web application with TLS certificates. Years ago everything was harder when developing with a TLS certificate. Acquiring a certificate was difficult and expensive. Using TLS certificates meant slower transactions and increased loads on the servers. Hosting companies would even offer servers with on-board numerical processors specifically designed to speed up the math involved in negotiating connections. Nowadays, it's hard to imagine a site without TLS, but for a very long time that was the accepted standard.

When we were developing the subscription service for Pivotal Tracker, using SSL was an option that was originally considered a "Pro" level feature. If you wanted 'https' in the URL of your project, you'd have to pay for it. Ultimately though, we decided to make it a configurable option regardless of your subscription plan. A user could enable or disable SSL as a project preference. To support this, we spent countless hours tweaking the code with `if/else` statements, checking the project's options and either allowing or redirecting requests. Tweaks to the Nginx configs were also necessary to allow http/https behavior. All this created some ugly code paths. To be honest, I always hated how we allowed http or https.

Times have certainly changed. Secure sites and web applications are now definitely the norm. Any sites without a certificate are considered suspect. However, it can still be difficult for engineers to obtain and work with TLS certificates. The maturity level of your organization may, or may not, determine how you can obtain a certificate.  Most of the time it comes down to how much freedom and flexibility do you as an engineer have to make decisions regarding certificates. The bigger the org, the more red-tape, lawyers and security managers concerned with who has access to the certificates and their private keys.

Back in the day, our engineering team was allowed to procure and manage our certificates. Procuring and updating certificates was always viewed as a cumbersome process. Having to repeat this process yearly was a hassle, so at least once we opted for certificates that expired 2 or 3 years out. That felt like a good idea at the time, but as far as things are concerned today, that was a terrible idea. The longer the certificate is valid, the longer that it could become compromised. If the key was ever compromised, a bad actor could impersonate your site for up to 3 years. Not good! The best practice is to have a shorter expiry date so that if a key is compromised, a bad actor will have a minimal time to use it.

Wildcard certificates were also a favorite. We had a variety of different environments to support: production, staging, alpha, beta, demo, etc. Having to request and purchase certificates for each of these would definitely be a pain. So, to keep it simple (stupid!), just get a wildcard certificate. Now we can use that same certificate across all of our environments. One certificate to rule them all! It seemed quick, easy and simple. And it was. But once again in hindsight, it was not a best practice. In the event of a compromise of the key, a bad actor could once again impersonate us using the subdomain of their choice. The best practice here is to obviously use a single certificate for a single domain and avoid wildcards.

Your hosting infrastructure may also expose your certificates and keys to others as well. For many years we were hosted at BlueBox. Their support team was fantastic and we trusted them with everything. This included our certificates. Due to the nature of how their infrastructure was configured, we didn't have access to the load balancers fronting our environment. Any change to the load balancers required a support ticket. This included updating the certificates and keys. We would typically upload our new certs to an app server in our environment and note in a support ticket the server name and path where they could be found. We obviously trusted BlueBox to handle our certs, but once again, not a best practice. The ideal is to have the certificates and keys delivered directly to the infrastructure without ever having to be handled by a human. 

Fast-forward to today, there are now protocols and processes that have implemented all of these best practices. There are 2 resources available that can make this happen. The first is [Let's Encrypt](https://letsencrypt.org). Let's Encrypt is a non-profit service provided by the [Internet Security Research Group](https://www.abetterinternet.org). their mission is to reduce the financial, technological, and educational barriers to secure communication over the Internet. To accomplish this, they are giving away free certificates. Utilizing a protocol called [Automated Certificate Management Environment](https://en.wikipedia.org/wiki/Automated_Certificate_Management_Environment), or ACME, they will issue TLS certificates to secure web applications and services. What could be better than a free service to secure your site?

Well, the next best thing is open source software that handles the automated part of that ACME protocol. The software package [cert-manager](https://cert-manager.io) enables seamless integration between your Kubernetes cluster and ACME services. This includes Let's Encrypt as well as other public key infrastructure services (PKI) services such as [Hashicorp Vault](https://www.vaultproject.io). cert-manager has all the smarts to make requests to an ACME service and handle the challenges to prove that you own the domain that the certificate is for. With the certificate in hand, cert-manager will save it as a secret within your cluster to be used by an ingress resource.

So let us take a look at using cert-manager and Let's Encrypt to secure our example Rails application. I've already created a Rails application that we can use for this, it's the Rails Blog application that is detailed in the [Ruby on Rails - Getting Started](https://guides.rubyonrails.org/v6.0/getting_started.html) guide. I've already followed the guide and committed it to a GitHub [repository](https://github.com/seemiller/rails-blog). This demonstration will be building off of the Tanzu Community Edition cluster that was deployed to AWS in [Part 1](https://talesfromthecommandline.com/post/20211225-rails-from-the-closet-to-kubernetes-deployment/).

> This guide builds off of Part 1

## Installation

After following the steps from Part 1, all we need to install here is the cert-manager package.

```shell
tanzu package install cert-manager \
  --package-name cert-manager.community.tanzu.vmware.com \
  --version 1.5.3
```

## Configuration

The first to step is to tell Kubernetes who is going to issue the certificates. This is done with an issuer. There are 2 types of issuers, an `Issuer` that is restricted to a specific namespace, or a `ClusterIssuer` which can be used cluster wide. We'll use a `ClusterIssuer` here. 

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    email: joe@example.com
    privateKeySecretRef:
      name: letsencrypt-staging
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    solvers:
    - http01:
        ingress:
          class: contour
```

> Be sure to use your real email to get notifications from Let's Encrypt regarding the status of your certificates.

This issuer specifies an Automatic Certificate Management Environment, or ACME, issuer type. ACME Certificate authorities need to verify that the requester owns the domain. To perform this verification, the CA will issue a challenge. There are DNS and HTTP challenges. In this example, the HTTP challenge method will be used, and the ingress controller is Contour.

Also notice that we’re using the staging servers for Let’s Encrypt. The Production server is rate limited, so it is best to develop and demonstrate using staging before switching to production. The staging server also issues certificates that are not trusted. This is so that you can verify that your pipelines and environments are configured correctly before switching to real, production certificates.

The certificates issued by Let's Encrypt expire after 90 days. But as long as your processes are still up and running, the certs will automatically be renewed.

Apply the yaml for the cluster issuer, then verify that it is ready.
```shell
kubectl apply --filename kubernetes/clusterissuer-staging.yaml
```

```shell
> kubectl get clusterissuers

NAME                  READY   AGE
letsencrypt-staging   True    28s
```

Next we need to update our ingress and instruct it to use TLS and request a certificate using the `letsencrypt-staging` cluster issuer via cert-manager. Our updated ingress will look like this:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rails
  namespace: blog
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-staging
    ingress.kubernetes.io/force-ssl-redirect: "true"
    kubernetes.io/tls-acme: "true"
    kubernetes.io/ingress.class: contour
spec:
  tls:
  - secretName: rails-blog-tls-secret
    hosts:
      - www.k8squid.com
  rules:
  - host: www.k8squid.com
    http:
      paths:
      - backend:
          service:
            name: rails
            port:
              number: 80
        path: /
        pathType: Prefix
```

Patch the ingress for the Rails Blog. This will immediately make a request through cert-manager to Let's Encrypt to get a TLS certificate for our site.

```shell
kubectl apply --filename kubernetes/rails-ingress-tls.yaml 
```

Check for the certificate and the certificate request. Within a minute, we have a certificate!

```shell
> kubectl get certificate,certificaterequest --namespace blog

NAME                                                READY   SECRET                  AGE
certificate.cert-manager.io/rails-blog-tls-secret   True    rails-blog-tls-secret    1m

NAME                                                             APPROVED   DENIED   READY   ISSUER                REQUESTOR                                         AGE
certificaterequest.cert-manager.io/rails-blog-tls-secret-54b88   True                True    letsencrypt-staging   system:serviceaccount:cert-manager:cert-manager    1m
```

Going back to the browser and refreshing the page, we're redirected to a secure URL. Trust the certificate and enjoy your freshly secured site!

## Conclusion

All the concerns from the past are handled with ease. No more concerns about what provider to use, or how much will it cost, or handling private key files. If I had to it again, I would definitely consider using this process. With the combination of Kubernetes, cert-manager and Let's Encrypt, it has never been easier or quicker to secure a website. Best of all, it follows all the best practices:
* Using a quality, trusted certificate authority
* Short, 90 day expiration
* No human ever had to handle the actual certificate files
* The entire process can be automated
* Free!
