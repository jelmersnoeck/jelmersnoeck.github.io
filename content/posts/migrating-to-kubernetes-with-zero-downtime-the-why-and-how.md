---
title: "Migrating to Kubernetes with zero downtime: the why and how"
date: 2018-03-21
---

Originally posted on [the Manifold blog](https://manifold.co/blog/migrating-to-kubernetes-with-zero-downtime-the-why-and-how-d64ba9a92619)

![header](/images/migrate-k8s-000.png)

We at [Manifold](https://www.manifold.co/) always strive to get the most out of everything we do. For this reason, we continuously evaluate what we've done to see if it still holds up to our standards. A while back, we decided to take a deeper look at our infrastructure setup.

In this blog post, we'll look at the reasons why we moved to [Kubernetes](https://kubernetes.io/) and the questions we asked ourselves. We'll then look at some of the compromises we had to make and why we had to make them. We'll also have a look how we configured our cluster to achieve our goals.

When we started Manifold, we did what we knew worked well. Use [Terraform](https://www.terraform.io/) to deploy containers on [AWS EC2](https://aws.amazon.com/ec2/) and expose these through [ELBs](https://aws.amazon.com/elasticloadbalancing/). We found ourselves in a position where we could spend some extra time on building a more mature platform. The initial implementation was very simple to begin with, but we started to see some pain points:

- Deploying was slow (~15min)
- No [Continuous Delivery](https://continuousdelivery.com/) meant only the Ops people knew how to deploy
- Running a single container per instance can become expensive. By increasing [container density](https://containerjournal.com/2017/03/15/dockers-big-differentiator-vms-density/), we could decrease cost

In the past year, [Kubernetes has become very popular](https://trends.google.com/trends/explore?date=today%205-y&q=kubernetes). With the experience the team had, we strongly believed in the future of this new technology. For this reason, we created our first [Kubernetes Integration](https://github.com/manifoldco/kubernetes-credentials). We also started thinking about integrations which would make Kubernetes more accessible. This is where the idea of building [Heighliner](https://heighliner.com/) was born.

This leads us to another principle we live by: [dogfooding](https://www.urbandictionary.com/define.php?term=dogfooding%20%28to%20dogfood%29). By using Manifold to build Manifold, we'd know exactly what our users need.

## Choosing a cluster

The first question we asked ourselves was "where are we going to run this cluster?". AWS does not offer a Kubernetes solution yet but [Azure](https://azure.microsoft.com/en-us/blog/introducing-azure-container-service-aks-managed-kubernetes-and-azure-container-registry-geo-replication/) and [Google Cloud Platform](https://cloud.google.com/kubernetes-engine/) do. Do we need to stay within AWS and manage our own cluster or do we want to move everything to another Cloud Provider?

The key questions we wanted answers to were:

- Can we create a High Availability cluster on AWS and how easy is it to manage this?
- How do we connect to our [RDS](https://aws.amazon.com/rds/) instance and what will the latency be?
- What do we do about our [KMS](https://aws.amazon.com/kms/) encryption?

## High Availability (HA) with kops

If we can easily create and manage a cluster within AWS, it would lower the necessity to move providers. The initial tests we did with [kops](https://github.com/kubernetes/kops) looked promising and we decided to take it a step further. It's time to set up a High Availability cluster.

To understand what HA means for Kubernetes, we need to first understand what it means in general.

> The central foundation of a highly available solution is a redundant, reliable storage layer. The number one rule of high-availability is to protect the data. Whatever else happens, whatever catches on fire, if you have the data, you can rebuild. If you lose the data, you're done.
>
> — [Kubernetes docs](https://kubernetes.io/docs/admin/high-availability/building/)

![Kubernetes components in a High Availability configuration](/images/migrate-k8s-001.png)
*Kubernetes components in a High Availability configuration*

Within a Kubernetes cluster, this storage layer is [etcd](https://coreos.com/etcd/) and runs on the master instances. etcd is a distributed key/value store which follows the [Raft consensus algorithm](https://raft.github.io/) to achieve quorum. Achieving quorum means having a set of servers agreeing on a set of values. To reach this consensus, it needs *lower(n/2)+1* parties to agree. Therefore we always need an uneven amount of instances with at least 3.

Below, we'll look at a few possible disruption cases.

## Tolerating instance failure

The first scenario we'll look at is to see what happens when a single instance terminates. Can we recover from this?

![Tolerating instance failure](/images/migrate-k8s-002.gif)
*Tolerating instance failure*

By specifying the amount of nodes we want, kops creates an [Auto Scaling Group](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html) per instance group. This ensures that when an instance terminates, a new one gets created. This allows us to keep consensus across our cluster when we lose an instance.

## Tolerating zone failure

Having set up instance failure allows us to tolerate failure of a single machine. But what happens when the whole datacenter is having issues due to a power cut for example? This is where [*Regions* and *Availability Zones*](https://buildazure.com/2017/09/22/azure-regions-and-availability-zones/) come into play.

Let's look back at our consensus formula: *lower(n/2)+1 instances with at least 3 instances*. We can translate this to zones, which would result in *lower(n/2)+1 zones with at least 3 zones*.

![3 master nodes spread across 2 zones](/images/migrate-k8s-003.png)
*3 master nodes spread across 2 zones*

![3 master nodes spread across 3 zones](/images/migrate-k8s-004.png)
*3 master nodes spread across 3 zones*

With kops, this too is simple. By specifying the zones we want to run both our masters and nodes in, we can configure HA at the zone level. This however is where we ran into our first roadblock. For arbitrary reasons, when we started Manifold, we decided to use the *us-west-1* region. As it turns out, this region only has 2 zones available. This meant that we'd have to find another solution to tolerate zone failure.

## Tolerating region failure (and beyond)

The main goal was to replicate the existing infrastructure. Our legacy setup did not run across multiple regions, so the new setup didn't have to either. We do believe that with the help of [Kubernetes Federation](https://kubernetes.io/docs/concepts/cluster-administration/federation/), this will be easier to set up.

## Internal Networking with Peering

Because of our regional restrictions, we had to find other ways to tolerate zone failure. One option is to create our cluster in a separate region.

Each region runs its own separated network. This means that we can't just use resources from one region in the other. For this, we looked into [inter-region VPC peering](https://docs.aws.amazon.com/AmazonVPC/latest/PeeringGuide/Welcome.html). This would allow us to connect to our us-west-1 region and access [RDS](https://aws.amazon.com/rds/) and [KMS](https://aws.amazon.com/kms/).

![Inter Region peering between us-west-1 and us-west-2](/images/migrate-k8s-005.png)
*Inter Region peering between us-west-1 and us-west-2*

This too set us back. As it turns out, the *us-west-1* region isn't the best region you could use. At the time we investigated this, *us-west-1* didn't support inter-region VPC peering. This means that we couldn't use this solution either.

## Decisions and compromises

With all this new knowledge, it was time to make a decision. Would we stay with AWS or move over to another provider?

It's worth noting that moving to another provider comes with a lot of extra overhead as well. We'd have to expose our database, migrate our KMS and re-encrypt all our data.

In the end, we decided to **stick with AWS** and run with the tolerating node failure solution. With [the announcement of Amazon EKS](https://aws.amazon.com/eks/) and inter-region peering coming soon, we felt like this was a good enough first step.

Managing your own cluster can be time consuming. To date, we've seen minimal impact, but we definitely **accounted for cluster maintenance**. The most time consuming would be **cluster updates**.

From a financial standpoint, we also compromised. Yes, it'd be cheaper than the legacy setup, but it'd be **more expensive than the competitors**. Azure and GCP both provide the master nodes for free, which cuts down in cost quite a bit.

## kops tips

For us, kops has worked great. It does come with a set of defaults that you should be aware of and overwrite. One of the key things to do is to **enable etcd encryption**. This is done by providing the *--encrypt-etcd-storage* flag.

By default, kops also doesn't **enable RBAC**. [RBAC](https://kubernetes.io/docs/admin/authorization/rbac/) is a great mechanism to limit the scope of your applications within your cluster. We highly recommend enabling this and setting up appropriate roles.

For security reasons, we've **disabled SSH** into our instances. This ensures that no one can access these boxes, even when we run with a **private network topology**.

## Configuring the cluster

With a cluster up and running, it's time to get to work. The next step would be to configure it so that we can start deploying applications into it.

Having managed our services with [Terraform](https://www.terraform.io/) before meant that we had quite a bit of control on how to set things up. We managed our ELBs, DNS, logging etc. through our Terraform configuration. We needed to make sure we can do this with our Kubernetes setup as well.

## Load Balancing

Kubernetes has the notion of [Services](https://kubernetes.io/docs/concepts/services-networking/service/) and [Ingresses](https://kubernetes.io/docs/concepts/services-networking/ingress/). With a Service, it's possible to group pods — usually managed by a [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) — and expose them under the same endpoint. This endpoint could either be internal or external. When configuring a Service as a [LoadBalancer](https://en.wikipedia.org/wiki/Load_balancing_%28computing%29), Kubernetes will generate an ELB. This ELB is then linked to the configured service.

![Service LoadBalancer](/images/migrate-k8s-006.png)
*Service LoadBalancer*

This is great, but there are [limits on the amount of ELBs](https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/elb-limits.html) you can have. By using an Ingress, we can create a single ELB and route the traffic within our cluster. There are several Ingresses available, but we went with the default [Nginx Ingress](https://github.com/kubernetes/ingress-nginx).

![Ingress LoadBalancer](/images/migrate-k8s-007.png)
*Ingress LoadBalancer*

Now that we can route traffic to a service, it's time to expose these through a domain. To do this, we used the [External DNS](https://github.com/kubernetes-incubator/external-dns) project. This is a great way to keep the configured domain names close to your application.

The last step we had to look at for exposing our services was making sure we served traffic through SSL. This turned out to be easy as well as there are already available solutions to this. We settled on [cert-manager](https://github.com/jetstack/cert-manager), which integrates with our Nginx Ingress.

## Service Configuration

Service Configuration was an easy win for us. We already started building Manifold on top of Manifold with our [Terraform Integration](https://github.com/manifoldco/terraform-provider-manifold/). Because of this, all the credentials we needed were already configured.

We designed our [Kubernetes Integration](https://github.com/manifoldco/kubernetes-credentials) with our Terraform Integration in mind. We kept the underlying semantics the same which meant that migrating credentials was a breeze.

We also added the [option for custom secret types](https://github.com/manifoldco/kubernetes-credentials/pull/20). This allowed us to configure a Docker Auth secret. When [pulling Docker images from a private registry](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/), you need this secret.

## Telemetry

One of the most important things of running a distributed system is knowing what's going on inside of it. For this, you need to set up centralized logging and metrics. We had this in our legacy platform so we definitely needed this in our new platform.

For logging, we expanded our dogfooding and set up [our LogDNA Integration](https://www.manifold.co/services/logdna) to gather logs. [LogDNA](https://logdna.com/) themselves provide a [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) [configuration](https://docs.logdna.com/docs/kubernetes). This allows you to ship the logs from your cluster to their platform.

For metrics, we were relying on [Datadog](https://www.datadoghq.com/) which worked well for us so far. As with LogDNA, Datadog also [provides a DaemonSet configuration](https://docs.datadoghq.com/agent/basic_agent_usage/kubernetes/). They even have a [great blog post](https://www.datadoghq.com/blog/monitor-kubernetes-docker/) on how to set this up!

## Migration

With the cluster configured and our applications deployed, it was time to migrate. To **ensure zero down time**, we had to do this in several stages.

The first stage was running the cluster on a **separate domain**. By connecting the two systems, we could test it without interrupting anyone. This helped us find and fix some early stage issues.

In the next stage, we'd **route some traffic to the Kubernetes cluster**. To do this, we set up [round robin](https://en.wikipedia.org/wiki/Round-robin_DNS). This is a great way to see how your cluster behaves with actual traffic. After about a week, we had enough confidence to move on to the next phase.

![DNS round-robin between the Legacy infrastructure and our Kubernetes cluster](/images/migrate-k8s-008.gif)
*DNS round-robin between the Legacy infrastructure and our Kubernetes cluster*

The third stage involved **removing the legacy DNS records**. After removing the appropriate Terraform configuration, all our traffic would flow through Kubernetes!

Because of DNS cache, we decided to keep the legacy up and running for a few more days. This way, people with a cached DNS entry would not encounter an error. This also gave us the possibility to rollback in case we saw something go wrong.

## Conclusion

Now that we've migrated, we can reflect back on things. Our team — and the company — has called this migration a success. We **decreased our deployment time from ~15min to ~1.5min** and managed to **cut operational costs** by doing so.

We still **haven't finished up our Continuous Delivery** pipeline, but we're working on it. We've started work on [Heighliner](https://heighliner.com/) which will in the first place help ourselves, but hopefully help others as well.

We encountered one major setback: not having 3 availability zones available to us. This **prevented us from running Highly Available across zones**. It's a compromise we decided to make to get us up and running and we'll be looking at fixing that soon.

Oh, and one more thing. **So. Much. YAML**. This is where [Heighliner](https://heighliner.com/) will help us out.

Massive shout-out to [Meg Smith](https://medium.com/@megthesmith) for her work on the images for this blog post.
