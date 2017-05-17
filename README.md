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
also do using the API. The API is also extensible.

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
specific things like moving a database node failover and backups. Basically,
anything that falls outside the 12-Factor App philosophy. In short: statefu
applications. This can be databases or "legacy" stateful applications that
don't like to be moved about.

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
between having Kubernetes APIs available and replacing experienced sysadmins
with a few lines of Golang is a veritable chasm, but we needn't consider this
an all or nothing affair- we can benefit a lot from the simplest of operators.
