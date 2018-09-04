# Lab: Hello World

This lab will walk you through deploying a very simple "Hello World" web app.

## Pre-reqs

## Preparing your environment

Start by starting your local minikube cluster:

```bash
minikube start
```

Then clone this repo and cd into this directory:

```bash
git clone https://github.com/99/meetup-workshop
cd meetup-workshop
```

We are now ready to deploy our Hello world web app!

## Run a bare Pod

A pre-built Docker image has been pushed to our Image repository. This means that we are ready to use the [pods.yaml](pod.yaml) in this directory to stand up a single [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/):

```bash
kubectl create -f pod.yaml
```

Breaking this down, we're using the Kubernetes CLI ([kubectl](https://kubernetes.io/docs/reference/kubectl/overview/)) and we're telling it to create the resource specified in the given file (`-f pod.yaml`).

From here, you can watch your Pod come up:

```bash
kubectl get pods -w
```

If you run this fast enough, you may see your Pod in a `ContainerCreating` or `Pending` state. Finally, you'll eventually see it transition to `Running`.

## Poke the Pod

Now that your Pod is running, let's look at a few things:

```bash
kubectl describe pod hello-world
```

This will show you how the Pod has been configured, which image it's running, which port(s) it exposes (if any), any recent events, crashes, or restarts, which node it's running on, and much more.

If we'd like to view the logs from the application running in the Pod:

```bash
kubectl logs hello-world
```

Or perhaps we'd like to attach to a shell within the Pod (sort of like SSH'ing):

```bash
kubectl exec -it hello-world /bin/sh
```

You can also find and do all of this the Kubernetes dashboard:

```bash
minikube dashboard
```

## Exposing the Pod

If you'd like to reach a specific port on a particular pod, you can port forward it to your local machine by doing something like:

```bash
# The port format here is LOCAL_PORT:REMOTE_PORT
kubectl port-forward hello-world 5000:5000
```

However, this is not useful for a production application. Your Pods will be constantly coming and going, and port-forwarding to our machine does nothing for our users.

We'll want to instead create a Kubernetes [Service](https://kubernetes.io/docs/concepts/services-networking/service/). We've included one in [service.yaml](service.yaml). Similar to how we created the Pod, we'll create the Service like this:

```bash
kubectl create -f service.yaml
```

After the Service is created, you can have minikube point your browser at it:

```bash
minikube service hello-world
```

This works a bit differently in production, but your traffic was routed through what amounts to a front-facing load balancer for the Pod(s) that matches the Service's selector.

## Enter: Deployment

It is rare that we use a stand-alone Pod by itself, primarily due to two attributes:

* A single Pod can't be cloned to scale up.
* If the Node that a Pod is running on dies, it will be marked as Terminated and will not be re-scheduled.

To simulate Node failure, let's delete your single Pod and try to go through our Service again:

```bash
kubectl delete pod hello-world
# See the Pod terminate and goes away. Append a -w to watch in realtime.
kubectl get pods
# Try to reach your Service.
minikube service hello-world
```

You'll see that we're now failing to connect, as expected. There are no Pods backing the Service. The users are furiously refreshing.

To handle our availability and scaling concerns, we instead use a [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) object. A Deployment uses a `template` to create the number of desired Pods (in the `replicas` field). If a Pod is Terminated, the Deployment will ensure that it gets replaced. It also becomes easy to scale the Pod count up or down.

Let's start by creating the included Deployment:

```bash
kubectl create -f deployment.yaml
```

You should now be able to see a single Pod via:

```bash
kubectl get pods
```

Unlike with our bare Pod, you'll notice that the name has a long hash appended to the name. For example `hello-world-596fc9fdcf-x9rlz`. To break this name down:

* `hello-world` = The value in `Deployment.metadata.name`
* `596fc9fdcf` = The ReplicaSet name. We'll gloss over this for now, but it'll correspond to a distinct "version" managed by the Deployment.
* `x9rlz` = A unique hash, to avoid Pod name collisions.

The Pod acts just like our previous bare pod, but has a slightly different name.

Moving on, you should once again be able to reach your app through its service:

```bash
minikube service hello-world
```

## Simulating failure

If you recall how we deleted our single bare Pod earlier, let's go on ahead and attempt to do the same with the Pod that our Deployment created:

```bash
# Find the name of the Pod to delete, since it's now long and semi-random
kubectl get pods
# Delete the Pod (your name will be different)
kubectl delete pod hello-world-xxxxxxxxx-yyyyy
# Make sure the Pod Terminates and goes away.
kubectl get pods
```

Depending on how quickly you get the updated Pod list, you'll either see a Terminating Pod and a new Pod, or just the new Pod (identifiable by its recent `AGE` column). The Deployment as a `replica` count of `1`, so Kubernetes knows to replace any Terminated Pods.

However, there was a second or two after you deleted the Pod to where your imaginary users were unable to reach your app.

## Scaling up

Let's reduce the likelihood of service disruptions by scaling the Deployment up!

```bash
# This will an open an editor. You'll want to edit the 'replicas' field
# from 1 to 3. Save+Quit and your changes will be applied.
kubectl edit deployment hello-world
# Or you can run this shorter command:
kubectl scale --replicas=3 deployment/hello-world
# After doing one or the other:
kubectl get pods
```

You should now see three Pods, two being newer than the other. You'll notice that the names have some overlap until the last hash.

You can now delete any of the three Pods without taking your app down!

## Deploying a new version your app

Now it's time to deploy a new version of your smash hit, "Hello world!". Let's pivot a bit and greet this Meetup group instead:

```bash
# We'll re-open our editor. Edit the 'image' field's tag from 1.0 to 1.1.
# Save+Quit and your changes will be applied.
kubectl edit deployment hello-world
# Let's re-visit your service:
minikube service hello-world
```

You should now see "Hello Meetup Group!" instead of "Hello world!". We'll go into this in more detail later, but what happened:

* The initial state was three pods running version 1.0.
* After you made your changes, the Deployment spun up a fourth Pod running 1.1.
* Once the new Pod passed health checks, one of the 1.0 Pods was terminated.
* New Pods are continuously stood up, replacing the old ones until all Pods are running 1.1.


## Summary

This completes our Kubernetes Workshop! To review:

* A `Pod` is the unit of work in Kubernetes.
* A Pod's ports can be reached via `kubectl port-forward` or through a Service.
* We use a Deployment and its `template` to stamp out a desired number of `replica` Pods. Unhealthy or terminated Pods get replaced for us with a Deployment.
