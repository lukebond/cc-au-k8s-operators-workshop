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

The Etcd operator is written in Golang, and therefore so will our workshop be.
This is not a Golang workshop, and I'm not a Golang expert. If there is one
thing I want you to take away from this, it's the demystification of
operators; a conceptual understanding of what's involved in building one.

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
complete automation of the deployment and management of containerised software
on Kubernetes, once the domain of high bus-factor sysadmins.

Getting the concept of operators can be difficult at first, because of the
amount of DevOps and Kubernetes jargon involved, but the concept is actually
quite simple: it is the automation of operational and maintenance tasks on a
Kubernetes cluster.

Kubernetes does a lot for you in terms of launching your applications and
keeping them running, but what it can't or won't do for you is application-
specific things like database node failover and backups. Basically, anything
that falls outside the 12-Factor App philosophy (i.e. stateful applications).
This could be databases or "legacy" stateful applications that don't like to
be moved about (perhaps they store files on disk).

Databases are very specialised applications. Most of them were written before
the era of PaaS and containers. The process expects its data on disk to
persist across executions. They don't expect to be moved from one host to
another. In short, they're not cloud-native. Aside from their storage
requirements, they also differ widely from one database to the next:
operations such as node fail-over and backup are not the same for any two
databases, and the knowledge of how to perform these operations resides, by
and large, in the heads and run-books of sysadmins.

Kubernetes allows us to extend the API to define our high-maintenance,
stateful apps as resource types that Kubernetes recognises, and we can then
write code against the Kubernetes API to monitor and manipulate these
resources. This is a powerful concept. The astute will understand that the gap
between having Kubernetes APIs available on the one hand and replacing
experienced sysadmins with a few lines of Golang on the other is a veritable
chasm, but we can benefit a lot from the simplest of operators.

For a more detailed introduction to operators, see my slides from a recent
talk on the subject:

> _https://github.com/lukebond/coreos-london-operators-20170124_

## CoreOS's Etcd Operator

CoreOS were the first to introduce the concept of the operator, with the
announcement of the release of the Etcd operator in November 2016. The
announcement blog post serves as a vision statement for the future of
operations as well as a set of guidelines for how to build Kubernetes
automation software that follows the operator pattern.

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

## The Practical Workshop

### 00 - Accessing and Familiarising yourself with your Workshop Sandbox in Katacoda

TODO:
- what is Katacoda
- accessing Katacoda
- starting the right environment
- testing kubectl is working
- pull the codebase
- explain that the codebase is in stages via branches, and that each branch of
  the code relates to a docker image tag in quay.io

### 01 - Third Party Resources

If you're familiar with Kubernetes you will have heard of Pods, Services,
ReplicationControllers and the like. These are in-built Kubernetes
_Resources_. They are entities in the Kubernetes API, with a schema, and there
are various commands that can be performed on them.

Kubernetes allows us to define our own resources, which are called _Third
Party Resources_ (TPR). An operator's first task is to register a TPR for the
service it is managing. For example, the Etcd operator registers a TPR called
"EtcdCluster". We will be registering one called "PostgresOperator".

The Kubernetes client library in Golang makes it very simple to register a
TPR:

```go
func createTpr() error {
	tpr := &v1beta1extensions.ThirdPartyResource{
		ObjectMeta: v1.ObjectMeta{
			Name: "postgres-cluster.lukeb0nd.com",
		},
		Versions: []v1beta1extensions.APIVersion{
			{Name: "v1"},
		},
		Description: "Semi-automatic PostgreSQL cluster",
	}
	_, err := c.KubeCli.Extensions().ThirdPartyResources().Create(tpr)
	if err != nil {
		return err
	}
	return nil
}
```

This is the first step in building our operator. Check out the branch
`workshop-01` to see the code for an operator that simply registers a TPR and
then stops.

```
$ git checkout workshop-01
```

Read through the code and try to understand it. There is a bit of boilerplate,
but start with `main()` in `cmd/operator/main.go`, see how the Kubernetes
client is configured and the operator is launched. Then focus on
`pkg/controller/controller.go`, in particular the call to
`KubeCli.Extensions().ThirdPartyResources().Create(tpr)`.

This branch has been built into a container, tagged and pushed to a repository
on quay.io: `quay.io/lukebond/postgres-operator:workshop-01`.

We can run this on Minikube using the yaml file provided in the repository:

```
$ kubectl create -f kube/deployment.yaml
```

We can see that a TPR has been created as a result:

```
$ kubectl get thirdpartyresources
NAME                            DESCRIPTION                         VERSION(S)
postgres-cluster.lukeb0nd.com   Semi-automatic PostgreSQL cluster   v1
```

To demonstrate that we've extended the Kubernetes API, we can observe that
some new endpoints are now available:

// TODO ensure this proxy endpoint is accessible in Katacoda

```
$ curl 127.0.0.1:8080/apis/lukeb0nd.com/v1
{
  "kind": "APIResourceList",
  "apiVersion": "v1",
  "groupVersion": "lukeb0nd.com/v1",
  "resources": [
    {
      "name": "postgresclusters",
      "namespaced": true,
      "kind": "PostgresCluster",
      "verbs": [
        "delete",
        "deletecollection",
        "get",
        "list",
        "patch",
        "create",
        "update",
        "watch"
      ]
    }
  ]
```

We'll be working with some of these endpoints in the upcoming sections.

### 02 - Watching our TPR API Endpoint

Now that we have a TPR of our own, what to do with it? We can create an
instance of that resource as well as watch Etcd for changes to it. The
creation of an instance of this resource will come through as a change in
Etcd. In this part of the workshop we will create a `monitor()` function that
will detect and print out these changes.

Check out the branch and explore the code:

```
$ git checkout workshop-02
```

In particular, note the addition of the call to `findAllClusters()` in
`initResource()`, and the `watchVersion` value that it returns. This variable
is the Etcd version number of the latest version of our `PostgresCluster`
Kubernetes resource, which increases with each change made to our cluster. We
use it when watching Etcd for changes, as we are basically saying to Etcd
"tell me of all changes since this version". Therefore it's important to
determine it at startup.

Also note the addition of the call to `monitor()` in the `Run` function in
the controller. This is the function that performs an Etcd watch based on the
version number fetched previously.

The `monitor()` function does an HTTP request to the Kubernetes API with
`watch=true`, which means an HTTP long poll. The HTTP body returned gets
parsed by a JSON decoder and unmarshalled into either an object of type
Status or Error, objects we've defined in `controller.go`.

The `monitor()` function uses an `event` and an `error` channel to communicate
events back to the calling function (`Run()`), where we print them out to the
console.

Notice that in the `kube` directory there is a new file
`example-postgres-cluster.yaml`. Running `kubectl create -f` on this file will
tell Kubernetes to create a new instance of type `PostgresCluster` that we
registered when we launch the operator. Eventually, this will actually
launch Postgres containers on the cluster, but for now we'll use it to trigger
the event monitoring we just added. It will result in the `ADDED` event being
logged by the operator, which is printed out in `Run()` in `controller.go`.

First remove what we previously deployed so that we have a clean slate:

```
$ kubectl delete deployment postgres-cluster
$ kubectl delete thirdpartyresources postgres-cluster.lukeb0nd.com
```

Now deploy our new operator and then create a PostgresCluster:

```
$ kubectl create -f kube/deployment.yaml
$ kubectl get thirdpartyresources
NAME                            DESCRIPTION                         VERSION(S)
postgres-cluster.lukeb0nd.com   Semi-automatic PostgreSQL cluster   v1
$ kubectl create -f kube/example-postgres-cluster.yaml
```

Now if we check the logs for our operator, we should see the `ADDED` message:

```
$ kubectl get pods
TODO insert example output
postgres-operator-abcxy
$ kubectl logs postgres-operator-abcxy
TODO insert example output
```

### 03 - 
