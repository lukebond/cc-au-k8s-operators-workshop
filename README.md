# Container Camp AU 2017 - Building a Kubernetes Operators

## Welcome!

Thanks for attending this workshop about building Kubernetes operators!

Here is what we're going to cover:

  - A brief introduction to the concept of operators
  - An overview of CoreOS's Etcd operator, as an example and a model for us to follow
  - The practical workshop:
    - Preliminaries: getting the code, using Katacoda, minikube, etc.
    - Third party resources
    - Watching for events
    - Working towards a skeleton Postgres operator

The codebase with which we're working is based directly on the Etcd operator.
I've basically stripped it down to its most basic form, removing anything Etcd-
specific, and put in placeholders for Postgres functionality, in order to
demonstrate what's common to the operator pattern.

This is a short workshop. I'm not going to take you from zero to a production-
ready Postgres operator in the time we have.

The Etcd operator is written in Golang, and therefore so will our workshop
codebase. This is not a Golang workshop, and I'm not a Golang expert. If there
is one thing I want you to take away from this, it's the demystification of
operators; a practical understanding of what's involved in building one.

## Pre-requisites

I expect you to have:

  - Some coding experience; at least enough to be able to read code and have
    an idea of what it's doing
  - Ability to use a terminal and a text editor
  - Basic Git skills
  - A basic working knowledge of Kubernetes core concepts and familiarity
    with `kubectl`

If you don't have some, or any, of these, don't panic. I think there is some
value to be gained from being here, reading the material and looking at the
code. After all, the main take-away is about comprehension, not coding.

## Introduction to Operators

Kubernetes' job is to run containerised applications and to manage their
configuration, storage, load balancing, service discovery and scaling.
Kubernetes is entirely API-driven: whatever Kubernetes itself can do, you can
also do using the API. The API is also highly extensible.

The above reveals the fact that Kubernetes was designed with automation in
mind from day one. Just as API-powered cloud providers made possible the era
of infrastructure-as-code and the automation of previously manual
infrastrucure tasks, so does Kubernetes make possible "no-ops", or the
complete automation of deploying and managing software, once the domain of
sysadmins with a high bus-factor.

Getting the concept of operators can be initially difficult, because of the
amount of DevOps and Kubernetes jargon involved, but the concept is actually
quite simple: it is the automation of operational and maintenance tasks on a
Kubernetes cluster.

Kubernetes does a lot for you in terms of launching your applicatoin and
keeping it running, but what it can't or won't do for you is application-
specific things like database node failover and backups. Basically, anything
that falls outside the 12-Factor App philosophy. In short: stateful
applications. This can be databases or "legacy" stateful applications that
don't like to be moved about (perhaps they store files on disk).

Databases are very specialised applications. Most of them were written before
the era of PaaS and containers, and don't like to be moved about. The process
expects its data on disk to persist across executions. They don't expect to be
moved from one host to another. In short, they're not cloud-native. Aside from
their storage requirements, they also differ widely from one database to the
next: operations such as node fail-over and backup are not the same for any
two databases, and the knowledge of how to perform these operations resides,
by and large, in the heads and run-books of sysadmins.

Kubernetes allows us to extend the API to define our high-maintenance,
stateful apps as resource types that Kubernetes recognises, and we can then
write code against the Kubernetes API to monitor and manipulate these
resources. This is a powerful concept. The astute will understand that the gap
between having Kubernetes APIs available on the one hand and replacing
experienced sysadmins with a few lines of Golang on the other is a veritable
chasm, but we needn't consider this an all or nothing affair- we can benefit a
lot from the simplest of operators.

For a more detailed introduction to operators, see my slides from a recent
talk on the subject:

> _https://github.com/lukebond/coreos-london-operators-20170124_

## CoreOS's Etcd Operator

CoreOS were the first to introduce the concept of the operator, with the
announcement of the release of the Etcd operator in November of 2016. This
blog post serves as a vision statement for the future of operations as well
as a set of guidelines for how to build Kubernetes automation software that
follows the operator pattern.

The core of the operator pattern is the "observe, analyse, act" cycle, which
aligns nicely with the declarative nature of Kubernetes resources such as
ReplicaSets.

  - **Observe** - fetch the state of the cluster (poll or watch)
  - **Analyse** - compare the observed state with the desired state, and
    determine the actions that will bring the system to the desired state
  - **Act** - perform the actions in order to bring the system towards the
    desired state

For example, an Etcd cluster may have a size of 3 before the operator is asked
to scale it up to 5. The operator _observes_ that there are 3 nodes, it
_analyses_ this information and concludes that it needs to _act_ to spin up
two more nodes.

The code for the Etcd operator can be found here:

> _https://github.com/coreos/etcd-operator.git_

## Accessing your Workshop Sandbox in Katacoda

TODO:
- accessing Katacoda
- starting the right environment
- testing kubectl is working
- pull the codebase

## Third Party Resources

If you're familiar with Kubernetes you will have heard of Pods, Services,
ReplicationControllers and the like. These are in-built Kubernetes
_Resources_. They are entities in the Kubernetes API, with a schema, and there
are various commands that can be performed on them.

Kubernetes allows us to define our own resources, which are called _Third
Party Resources_ (TPR). An operator's first task is to register a TPR for the
service it is managing. For example, the Etcd operator registers a TPR called
"EtcdCluster". We will be registering one called "PostgresOperator".

