# Importing Scully in Jenkins X

Let's start by importing [Scully](https://github.com/the-jenkins-x-files/scully) in your Jenkins X Cluster:

- Head over to <https://github.com/the-jenkins-x-files/scully> to **fork** the repository to your own account
- Clone your fork
    ```
    $ git clone git@github.com:YOUR_USER_NAME/scully.git
    ```
- At this point, you can inspect the source code, read the *README* file
- Run the [jx import](https://jenkins-x.io/commands/jx_import/) command:
    ```
    $ jx import
    ```
  It will create a new commit with:
    - a `Dockerfile` to build a Docker image
    - a `Jenkinsfile` with a CI/CD pipeline
    - 2 [Helm](https://helm.sh/) charts, in the `charts` repository:
        - the `scully` chart: this is the main chart for our application, with the Kubernetes manifests in the `templates` subdirectory
        - the `preview` chart, which is just an umbrella chart with a dependency on the `scully` chart, and is used when deploying the preview environments
- While you were inspecting the changes made by Jenkins X, Jenkins did the first build of your project:
    - it is a build of the `master` branch, so the pipeline created a new release `v0.0.1` that you can find at <https://github.com/YOUR_USER_NAME/scully/releases>
    - it also created (and merged) a Pull Request in your staging environment Git repository at <https://github.com/YOUR_USER_NAME/environment-jx-workshop-staging/pulls?q=is%3Apr>
    - and as a result, it deployed the [Scully](https://github.com/the-jenkins-x-files/scully) application in the **staging** environment - which is the `jx-staging` namespace in your Jenkins X Kubernetes Cluster.

This is great, and so far Jenkins X did most of the work - but now it's our turn ;-)

First, let's check if we can use our application. We can list the deployed applications in a given environment by running the [jx get applications](https://jenkins-x.io/commands/jx_get_applications/) command:

```
$ jx get applications -e staging
```

The endpoint's URL is something like `http://scully.jx-staging.XXX.nip.io`. If you open this URL in your browser, you should get a `503 Service Temporarily Unavailable` error message. To debug our issue, we need to find our application's [pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/). In Kubernetes, pods are deployed in [namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/). So we need to find the namespace used for the **staging** environment. To do that, we can use the [jx get environments](https://jenkins-x.io/commands/jx_get_environments/) command:

```
$ jx get environments
```

It confirms that we need to use the `jx-staging` namespace. And if we want to list the pods in the `jx-staging` namespace:

```
$ kubectl get pods -n jx-staging
```

We can see that our `scully` pod is running, but has already restart a few times. Let's have a look at the logs:

```
$ kubectl logs POD_NAME -n jx-staging
```

and we should get the following error:

> Error: ENOENT: no such file or directory, stat '/usr/src/app/build/index.html'

If you don't, maybe the container just started. In this case, you can see the logs of the previous container instance by running:

```
$ kubectl logs POD_NAME -n jx-staging -p
```

This error means that we have some missing files in our container image. Most likely a build issue. So [let's fix our build](fix-build.md)!
