# Importing Mulder in Jenkins X

Let's start by importing [Mulder](https://github.com/the-jenkins-x-files/mulder) in your Jenkins X Cluster:

- Head over to <https://github.com/the-jenkins-x-files/mulder> to **fork** the repository to your own account
- Clone your fork
    ```
    $ git clone git@github.com:YOUR_USER_NAME/mulder.git
    ```
- At this point, you can inspect the source code, read the *README* file
- Run the [jx import](https://jenkins-x.io/commands/jx_import/) command:
    ```
    $ jx import
    ```
  It will create a new commit with:
    - a `Makefile` pre-configured to build our project
    - a `Dockerfile` to build a Docker image
    - a `Jenkinsfile` with a CI/CD pipeline
    - 2 [Helm](https://helm.sh/) charts, in the `charts` repository:
        - the `mulder` chart: this is the main chart for our application, with the Kubernetes manifests in the `templates` subdirectory
        - the `preview` chart, which is just an umbrella chart with a dependency on the `mulder` chart, and is used when deploying the preview environments
- While you were inspecting the changes made by Jenkins X, Jenkins did the first build of your project:
    - it is a build of the `master` branch, so the pipeline created a new release `v0.0.1` that you can find at <https://github.com/YOUR_USER_NAME/mulder/releases>
    - it also created (and merged) a Pull Request in your staging environment Git repository at <https://github.com/YOUR_USER_NAME/environment-jx-workshop-staging/pulls?q=is%3Apr>
    - and as a result, it deployed the [Mulder](https://github.com/the-jenkins-x-files/mulder) application in the **staging** environment - which is the `jx-staging` namespace in your Jenkins X Kubernetes Cluster.

This is great, and so far Jenkins X did most of the work - but now it's our turn ;-)

First, let's check if we can use our application. We need to find the [namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) used for the **staging** environment. To do that, we can use the `jx get environments` command. It confirms that we need to use the `jx-staging` namespace.

We need to retrieve the endpoint of our application, and to do that, we'll check the [ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) in the `jx-staging` namespace:

    ```
    $ kubectl get ingress -n jx-staging
    ```

The endpoint's host is something like `mulder.jx-staging.XXX.nip.io`. If you open this URL in your browser, it should... not work. To debug our issue, we can start by inspecting the [pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/) in the `jx-staging` namespace:

```
$ kubectl get pods -n jx-staging
```

We can see that our `mulder` pod is crashing. Let's have a look at the logs:

```
$ kubectl logs POD_NAME -n jx-staging
```

and we should get the following error:

> dial tcp :6379: connect: connection refused

which means that it can't connect to Redis. Of course. So the next step is to [add Redis as a dependency](add-redis-dependency.md) to make it work.
