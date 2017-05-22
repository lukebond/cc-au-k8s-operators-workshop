# Container Camp AU 2017 - Building a Kubernetes Operators

## Welcome!

Thanks for attending this workshop about building Kubernetes operators!

My name is Luke Bond, and I work for the UK government as a DevOps engineer,
working with Docker and Kubernetes.

Any questions or follow-up, you can reach me at @lukeb0nd on Twitter, or you
can email me at luke.n.bond@gmail.com.

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
demonstrate what's common to the operator pattern. This isn't original code,
it's mostly copy-paste!

We only have a few hours! Since this is a short workshop, I'm not able to take
you from zero to a production-ready Postgres operator in that time.

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
  - Basic Docker experience
  - A basic working knowledge of Kubernetes core concepts and familiarity
    with `kubectl`

If you don't have some, or any, of these, don't panic. I think there is some
value to be gained from being here, reading the material and looking at the
code. After all, the main take-away is about comprehension, not coding.

You will need to have a working Minikube environment, as well as Git and
kubectl:

> https://kubernetes.io/docs/tasks/tools/install-minikube/
> https://kubernetes.io/docs/tasks/tools/install-kubectl/

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

The code we're working with today is taken from the Etcd operator codebase a
few months ago, at commit SHA `11d6afff17abd9708909b06324d4c272920701f2`.

## The Practical Workshop

### 00 - Accessing and Familiarising yourself with your Workshop Sandbox in Katacoda

Ensure your minikube installation is functioning:

```
$ minikube status
minikubeVM: Running
localkube: Running
```

If you don't see this you may need to start one:

```
$ minikube start
```

If you have trouble getting it started, try a `minikube delete` before
running `minikube start`.

Clone my Postgres operator codebase:

```
$ git clone https://gitlab.com/lukebond/postgres-operator.git
```

Check that kubectl is pointing to your local cluster:

```
$ kubectl config current-context
minikube
$ kubectl get pods
No resources found
```

You don't need to be able to build and run the code in my repository as I've
already built and pushed Docker images for the three stages of the workshop.
Unless you're particularly interested in editing the code, you don't need to
do the following steps. However, if you'd like to play with the code you can
build it like so (you will need a working Golang build environment:

```
$ make build
```

On successful build it will create a Docker image locally on your laptop. To
get that onto minikube, I find the easiest way is to export it from your
laptop's Docker daemon and then import it into minikube.

```
$ docker save quay.io/lukebond/postgres-operator:workshop-01 -o postgres-operator-workshop-01.tar
$ minikube mount .
Mounting . into /mount-9p on the minikubeVM
This daemon process needs to stay alive for the mount to still be accessible...
ufs starting
^z
$ bg
$ minikube ssh
$ chmod 666 /mount-9p/postgres-operator-workshop-01.tar
$ docker load -i /mount-9p/postgres-operator-workshop-01.tar
```

This will load your latest built image into the Docker daemon in Minikube.

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

```
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
deployment "postgres-operator" created
```

We can see that a TPR has been created as a result:

```
$ kubectl get thirdpartyresources
NAME                            DESCRIPTION                         VERSION(S)
postgres-cluster.lukeb0nd.com   Semi-automatic PostgreSQL cluster   v1
```

To demonstrate that we've extended the Kubernetes API, we can observe that
some new endpoints are now available:

```
$ kubectl proxy --port=8080 &
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

> Kubernetes uses Etcd natively to store information about the state of
> what's running in the cluster. Watching Etcd for changes is a common
> operation when writing code that interacts with Kubernetes, as operators do.

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
$ kubectl delete deployment postgres-operator
$ kubectl delete thirdpartyresources postgres-cluster.lukeb0nd.com
```

Now deploy our new operator and then create a PostgresCluster:

```
$ kubectl create -f kube/deployment.yaml
deployment "postgres-operator" created
$ kubectl create -f kube/example-postgres-cluster.yaml
postgrescluster "example-postgresql-cluster" created
```

Now if we check the logs for our operator, we should see the `ADDED` message:

```
$ kubectl get pods
NAME                                 READY     STATUS    RESTARTS   AGE
postgres-operator-1337880999-ph9md   1/1       Running   1          11m
$ kubectl logs postgres-operator-1337880999-ph9md
Third-party resource already exists; continuing
Finding existing clusters...
Watch version: %s 4566
Controller initialised
Starting at watchVersion %v 0
Postgres cluster event: %v %v ADDED &{3 9.6.3 map[] false}
&{ADDED 0xc42034e5a0}
```

Let's modify the cluster and see what happens. Open up `kube/example-postgres-cluster.yaml`
in a text editor (e.g. `vim`) and change the cluster size to `5`. Now post it
to Kubernetes:

```
$ kubectl replace -f kube/example-postgres-cluster.yaml
postgrescluster "example-postgresql-cluster" replaced
$ kubectl logs postgres-operator-1337880999-ph9md
Third-party resource already exists; continuing
Finding existing clusters...
Watch version: %s 4566
Controller initialised
Starting at watchVersion %v 0
Postgres cluster event: %v %v ADDED &{3 9.6.3 map[] false}
&{ADDED 0xc42034e5a0}
Postgres cluster event: %v %v MODIFIED &{5 9.6.3 map[] false}
&{MODIFIED 0xc42034eb40}
```

You can see that the new cluster size of 5 is available in the event that is
printed out here. Our operator will use this information to reconcile the
cluster size.

Now try deleting the cluster:

```
$ kubectl delete -f kube/example-postgres-cluster.yaml
postgrescluster "example-postgresql-cluster" deleted
$ kubectl logs postgres-operator-1337880999-ph9md
Third-party resource already exists; continuing
Finding existing clusters...
Watch version: %s 4566
Controller initialised
Starting at watchVersion %v 0
Postgres cluster event: %v %v ADDED &{3 9.6.3 map[] false}
&{ADDED 0xc42034e5a0}
Postgres cluster event: %v %v MODIFIED &{5 9.6.3 map[] false}
&{MODIFIED 0xc42034eb40}
Postgres cluster event: %v %v DELETED &{5 9.6.3 map[] false}
&{DELETED 0xc420136240}
```

### 03 - Cluster Reconciliation

We've learned the basics of watching Etcd for changes to our Third Party
Resource. Let's look a bit more deeply into what's going on here, and extend
it a little to show how an operator might reconcile a cluster to a desired
configuration.

After the `monitor()` function has completed and pushed events onto the events
channel, we are currently iterating over them and printing them out. Let's
replace those printouts with some more detailed skeleton code.

One of the main additions is an event and error channel for each cluster.

Check out the branch:

```
$ git checkout workshop-03
```

Note the new code in the `Run()` function in `controller.go`, in the switch
statement. On `ADDED`, `MODIFIED` and `DELETED` events we are now calling out
to some new code. The new code in `pkg/cluster/cluster.go` in `New()`,
`setup()` and `create()` initialises the cluster state and then has a place-
holder printout that says "PLACEHOLDER: Launch seed Postgres node!". This is
where we'd start a Postgres container for the first node and bootstrap the
cluster from there. We won't get that far today.

In `pkg/cluster/cluster.go` we have new functions `Update()` and `Delete()`
that get called when the `MODIFIED` and `DELETED` events arrive. These two
functions just pushes the events onto the cluster's channel, using new event
types `eventModifyCluster` and `eventDeleteCluster`. These events are
handled in `run()` in `pkg/cluster/cluster.go`.

Have a read through the code I've pointed out and see if you can guess what
you will see in the logs when you run it.

Once you're ready to find out, delete the previous operator to start from a
clean slate:

```
$ kubectl delete deployment postgres-cluster
$ kubectl delete thirdpartyresources postgres-cluster.lukeb0nd.com
```

Now deploy our new operator and then create a PostgresCluster:

```
$ kubectl create -f kube/deployment.yaml
deployment "postgres-operator" created
$ kubectl get thirdpartyresources
NAME                            DESCRIPTION                         VERSION(S)
postgres-cluster.lukeb0nd.com   Semi-automatic PostgreSQL cluster   v1
$ kubectl create -f kube/example-postgres-cluster.yaml
postgrescluster "example-postgresql-cluster" created
$ kubectl get pods
NAME                                 READY     STATUS    RESTARTS   AGE
postgres-operator-1449816488-xhs5d   1/1       Running   1          11m
$ kubectl logs postgres-operator-1449816488-xhs5d
Third-party resource already exists; continuing
Finding existing clusters...
Watch version: %s 15333
Controller initialised
Starting at watchVersion %v 0
Postgres cluster event: %v %v ADDED &{5 9.6.3 false map[] false}
&{ADDED 0xc420100480}
creating cluster with Spec (%#v), Status (%#v) &{5 9.6.3 false map[] false} <nil>
PLACEHOLDER: Launch seed Postgres node!
PLACEHOLDER: Start cluster!
Running: %v, pending: %v [] []
All Postgres pods are dead. Trying to recover from a previous backup
PLACEHOLDER: disaster recovery!
```

Was this close to what you expected?

What we're seeing here is a placeholder to create the seed Postgres node
(but of course no such node was created), and then the operator declaring that
the whole cluster is dead and needs to be restored from a backup.

If we had actually created the seed node we wouldn't have seen the disaster
message, but instead would have seen reconciliation related messages. Notice
that in the cluster `run()` function there is a timer to periodically
reconcile the cluster. It fetches the number of pods, compares it against the
desired state, and makes decisions about what to do.

This is the more complex part of the operator and you should spend some time
reading the code and trying to understand it.

### Bonus Challenges for the Early Finisher

Or "homework" for the rest of us!

- Build the codebase yourself to create a Docker image for the operator
- Launch a seed Postgres node, using the Etcd operator as a guide, and a
  publically-available Postgres image (e.g. the official one)
- Reconcile the instance count to get a healthy cluster
- Simulate losing a node (hint: kubectl delete pod) and see if your operator
  can recover it

## Wrap-up

You've learned the basics of how to structure a Kubernetes operator written in
Golang. It may seem that we didn't get very far towards a production-ready
operator, but bear in mind that operators were only announced late last year,
and there are only a handful of them that have been released. That fact that
you're familiar with them and could start writing one soon already means
you're ahead of most people!

Next time you're releasing stateful software onto Kubernetes, consider
automating its operations by writing an operator.

For the sake of time I left out chaos monkey style testing for operators.
CoreOS, in their guidelines for building operators, recommend these kind of
tests. I discuss this in my recent talk, the link to which you can find above.
It's really easy to add to your application, so do check it out!

Any questions or follow-up, you can reach me at @lukeb0nd on Twitter, or you
can email me at luke.n.bond@gmail.com.
