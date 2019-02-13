# Adding Redis as a dependency

Our application is deployed as a [Helm](https://helm.sh/) chart, which means that we need to use Helm to manage our dependencies. The first step is to find a chart for [Redis](https://redis.io/). We can use <https://hub.helm.sh/> for that.

Let's use the [Redis chart from the "stable" repository](https://hub.helm.sh/charts/stable/redis).

## New dependency

To add a dependency on the Redis chart, you will need to add a new `requirements.yaml` file in the `charts/mulder` directory. See [Helm's documentation](https://docs.helm.sh/developing_charts/#chart-dependencies). You can find this file at [requirements.yaml](requirements.yaml).

But adding a dependency in the `requirements.yaml` file is not enough. As written in [Helm's documentation](https://docs.helm.sh/developing_charts/#chart-dependencies), we need to run `helm dependency update` to retrieve the dependencies charts. Because we don't want to store the charts archives in our source code, we'll run that command in our CI/CD pipeline - that means in our `Jenkinsfile`.

The "Jenkins X way of doing things" is to wrap commands in the `jx` CLI, so that it can take care of little details, like initializing the Helm client-side before using it, and so on. In our case, instead of calling `helm dependency update`, we'll call [jx step helm build](https://jenkins-x.io/commands/jx_step_helm_build/). See the documentation for [jx step](https://jenkins-x.io/commands/jx_step/) to see all the steps you can use in your `Jenkinsfile`.

So, in the `Jenkinsfile`, in the `CI Build and push snapshot` stage, you need to run `jx step helm build` in the `charts/mulder` directory, before creating the preview environment. Your `Jenkinsfile` should look like:

```
dir('/home/jenkins/go/src/github.com/XXX/mulder') {
  checkout scm
  [...]
}
dir('/home/jenkins/go/src/github.com/XXX/mulder/charts/mulder') {
  sh "jx step helm build"
}
dir('/home/jenkins/go/src/github.com/XXX/mulder/charts/preview') {
  sh "make preview"
  [...]
}
```

Now, Jenkins will install both our application and Redis together. Perfect! Let's create a new git branch, commit your changes, and push them to your fork.
You might have notice that our CI/CD pipeline is configured to only build Pull Requests - see the `when { branch 'PR-*' }` condition in the `Jenkinsfile` - so you will need to create a Pull Request for your branch. Open github in your browser - you can use `jx repo` to do that - and create a new PR. Take care that by default, Github will try to be smart, and will set the "base repository" of the Pull Request to [the-jenkins-x-files/mulder](https://github.com/the-jenkins-x-files/mulder) instead of your fork at `YOUR_USER_NAME/mulder`. You will need to fix the "base repository" field in the UI before creating the PR, otherwise your Jenkins will never build this PR.

Next, Jenkins will start a new job to build your PR - and to create a preview environment. It will also comment on the PR in github with the link to the application. Let's try to open it... it doesn't work. Again.

Let's inspect this issue. First, let's find the namespace in which our preview environment has been deployed. Each preview environment is deployed in its own namespace. If you run `jx get previews` you can see a list of all preview environments, with their namespaces. The namespace for the first PR should be `jx-XXX-mulder-pr-1`.

If we list the pods in this namespace - with `kubectl get pods -n jx-XXX-mulder-pr-1` - we can see that our mulder pod is still failing. And if we have a look at the logs - with `kubectl logs POD_NAME -n jx-XXX-mulder-pr-1` - we can see that we still have the same error:

> dial tcp :6379: connect: connection refused

This is because we deployed a Redis instance, but we never told Mulder to connect to it. Let's fix that.

## Updating the deployment

First, we'll need to know the Redis "hostname" that Mulder can use. In the Kubernetes world, applications talk together through services. Let's list all the services in the namespace used by our preview environment:

```
$ kubectl get services -n jx-XXX-mulder-pr-1
```

We see 3 services:
- our mulder service
- a `jx-xxx-mulder-pr-1-redis-master` service
- a `jx-xxx-mulder-pr-1-redis-slave` service

This is because the Redis chart deployed a master/slave setup by default. In our case, we'll need to connect to the master. The issue is that the name of the service is dynamic: it's based on the PR number. In fact, it's based on the Helm "release name". If you run `helm list`, you can list all the applications (releases) installed by Helm. Amongst them, there is our preview environment named `jx-xxx-mulder-pr-1`.

Another way to find the relation between the preview and the Helm release, is to list the previews in YAML format, with `jx get previews -o yaml`: you will get the whole definition of the preview environment in YAML, including the `jenkins.io/chart-release` annotation, whose value is the Helm release name.

So, now that we have the Redis service name, we can use it to configure Mulder. If we check [Mulder's README](https://github.com/the-jenkins-x-files/mulder/blob/master/README.md) in the **Flags** section it says that we need to use the `-redis-addr` flag to configure the Redis `host:port` to connect to.

Let's update our Kubernetes [deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) to do that. Open the `charts/mulder/templates/deployment.yaml` file. You should also open the [Deployment's API reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.13/#deployment-v1-apps) to have the documentation for each field. Locate the `containers` array - it should have only 1 container. This is where we'll need to add the flags for our application.

If you look at the [Container's API reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.13/#container-v1-core), you can see that it has an optional `args` field, whose value is a string array. We will just need to add 2 arguments:
- the flag name: `-redis-addr`
- and its value: `{{ .Release.Name }}-redis-master:6379`. `{{ .Release.Name }}` is a special Helm variable that will be replaced by the name of the release - see [Helm's list of predefined values](https://docs.helm.sh/developing_charts/#predefined-values).

You can find the whole file at [deployment.yaml](deployment.yaml).

Time to commit! And to push, of course. This will trigger a new Jenkins build. You can check the progress of the build, and at the end, test if the Mulder application has been re-deployed. If you run the `helm list` command, you can see that your release is now at its second revision. Running `helm history jx-xxx-mulder-pr-1` will show you all revisions. And if you install the [helm-diff](https://github.com/databus23/helm-diff) plugin, you will be able to display the changes between revision 1 and 2, by running the command `helm diff revision jx-xxx-mulder-pr-1 1 2`.

So, back to our application. If you try to visit its URL you can see that it's not working. So let's inspect the pod once again, and its logs. This time, we have a different error message:

> Failed to connect to jx-xxx-mulder-pr-1-redis-master:6379: NOAUTH Authentication required.

Ahah. Authentication issue. So installing Redis is a little more complex than just adding a new Helm dependency!

## Fixing the default configuration

Let's get back to the [Redis chart documentation](https://hub.helm.sh/charts/stable/redis): each chart comes with default values. We just need to have a look at these values, to understand the default setup that was used. If you search for "password", you can see that the default is a "randomly generated" password. Hum, so we can either force a specific password, or... disable authentication completely. Let's do that for now, and if you want you will be able to come back later and fix that, to use a real password.

So, we need to set the `usePassword` value to `false`. We'll add that value to our own default values, which are stored in the `charts/mulder/values.yaml` file. But because this value is used by the `redis` chart, we need to put it in the `redis` section:

```
redis:
  usePassword: false
```

You can commit again, and push - again. You know how it works now: wait for the end of the Jenkins build, and visit the preview environment's URL - this time you should have a `200 OK` response! Try to get the `/quote/random` endpoint: you should get a JSON response with a random quote! Try a few times, to make sure you get different quotes.

You can also use the `helm list` or `helm history` commands to see that our release has a new revision. And if you run `helm get jx-xxx-mulder-pr-1` you will see a lot of information about the release, including all the values, Kubernetes manifests and Helm hooks.

Well, now that everything's green, it's time to merge the PR! This will automatically trigger a new PR in your "staging environment" git repository, which in turn will trigger a new Jenkins build - to deploy your new release to the staging environment.

Once its done, you can check the `jx-staging` Helm release to see its changes - using `helm history jx-staging` for example, or even `helm diff`. And if you open <http://mulder.jx-staging.YOUR_DOMAIN/quote/random> in your browser, you should get a random quote!

One more comment on the PR that was merged. Closing or merging a PR will automatically delete the associated Preview Environment, but not in real time. There is a separate process, that runs at regular interval, and check for Preview Environments that should be deleted. It's called `jenkins-x-gcpreviews`, and is setup as a Kubernetes [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/). You can see it by running `kubectl get cronjobs` - and if you run `kubectl get jobs` you will see its last runs. So don't worry if your preview is still there even if the PR has been merged: it may take some time for the job to run and garbage-collect it.

So, everything done? Well, not yet. We have a functional CI/CD pipeline for our application, but so far we had to do manual testing. What about [running the unit & integration tests automatically](run-tests.md)?
