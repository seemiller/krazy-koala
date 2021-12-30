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


The first to step is to tell Kubernetes about the issuer.

`cat clusterissuer-staging.yaml`

This specifies an Automatic Certificate Management Environment, or ACME, issuer type.

ACME Certificate authorities need to verify that the requester owns the domain. To perform this verification, the CA will issue a challenge.

In this example the HTTP challenge method will be used.

Also notice that we’re starting with the staging servers for Let’s Encrypt.


The Production server is rate limited, so it is best to develop using staging before switching to production.

I’ve prepared both staging and production issuers, let’

`kubectl apply -f clusterissuer-staging.yaml -f clusterissuer-prod.yaml`

Next is to configure our ingress.

`cat ingress-staging.yaml`

We’ve added annotations to reference

* the issuer
* Force redirects to SSL
* And setup an ingress shim

We then specify that the certificate should be

* Stored in the wordpress-tls secret
* And that the cert is for the wordpress.k8squid.com host

Let’s apply that file,

`kubectl apply -f ingress-staging.yaml`

And then watch for our certificate to become ready.

`kubectl get certificates`

We can inspect the certificate to see its details.

kubectl get certificate/wordpress-tls -o yaml | more

Items of note are the status timestamps indicating expiration and renewal dates times.

And if we are really curious, we can always extract the actual certificate from the secret...

```
kubectl get secret/wordpress-tls -o json | jq -r '.data."tls.crt"' | base64 -d | openssl x509 -noout -text | more
``` 

Going back to the browser,

refreshing the page redirects us to a secure URL, but the certificate isn’t trusted.

While the certificates from the Let’s Encrypt staging servers are real, the root certificate is not distributed, as these certs are only meant to used in the development and test of your workflow.

Now that we know that the process is working, we’re ready to switch to the production servers and certificates.

Making the switch to production is quite easy,

There are 3 simple steps

* The first is to update the ingress resource to reference the production issuer for Let’s Encrypt
* `k edit ingress/wordpress`
* and then delete the staging certificate and secret.
* `kubectl delete certificate/wordpress-tls`
* `kubectl delete secret/wordpress-tls`
* Since the ingress is still in place, the declarative features of Kubernetes will kick in - the control loop pattern sees that a secret and certificate are being referenced but they doesn’t exist.

* This causes the operator to request a new certificate and store it in a new secret.

Back to the browser, refreshing reveals a secure page.

So let's see how we fared according to our best practices.

Let's inspect our certificate again,

* `/Issuer` We are using a quiality certificate authority
* `/Valid` Our Expiration date is 90 days out
* `Alter` We are using a Subject Alternative Name and specific hostnames.
* This certificate is being stored as a secret, so no human ever had to handle the actual certificate files.
* And the entire process was automated through Kubernetes
