# Adding Mulder as a dependency

**Using a pre-packaged chart**

There is a pre-packaged chart for [Mulder](https://github.com/the-jenkins-x-files/mulder) at <https://the-jenkins-x-files.github.io/charts/>. You can inspect it from your laptop using [Helm](https://helm.sh/):

```
$ helm repo add the-jenkins-x-files https://the-jenkins-x-files.github.io/charts
$ helm repo update
$ helm search mulder
```

You should see a chart named `the-jenkins-x-files/mulder` at version `1.0.0`. And you can inspect its content by running `helm inspect the-jenkins-x-files/mulder`.

## New dependency

To add a dependency on the Mulder chart, you will need to add a new `requirements.yaml` file in the `charts/scully` directory. See [Helm's documentation](https://docs.helm.sh/developing_charts/#chart-dependencies). You can find this file at [requirements.yaml](requirements.yaml).

But adding a dependency in the `requirements.yaml` file is not enough. As written in [Helm's documentation](https://docs.helm.sh/developing_charts/#chart-dependencies), we need to run `helm dependency update` to retrieve the dependencies charts. Because we don't want to store the charts archives in our source code, we'll run that command in our CI/CD pipeline - that means in our `Jenkinsfile`.

The "Jenkins X way of doing things" is to wrap commands in the `jx` CLI, so that it can take care of little details, like initializing the Helm client-side before using it, and so on. In our case, instead of calling `helm dependency update`, we'll call [jx step helm build](https://jenkins-x.io/commands/jx_step_helm_build/). See the documentation for [jx step](https://jenkins-x.io/commands/jx_step/) to see all the steps you can use in your `Jenkinsfile`.

So, in the `Jenkinsfile`, in the `CI Build and push snapshot` stage, you need to run `jx step helm build` in the `charts/scully` directory, before creating the preview environment. Your `Jenkinsfile` should look like:

```
container('nodejs') {
  sh "npm install"
  [...]
  sh "jx step post build --image $DOCKER_REGISTRY/$ORG/$APP_NAME:$PREVIEW_VERSION"
  dir('./charts/scully') {
    sh "jx step helm build"
  }
  dir('./charts/preview') {
    sh "make preview"
    [...]
  }
}
```

Now, Jenkins will install both our Scully and Mulder applications together. Perfect! Let's commit the changes, and push them. This will trigger a new Jenkins build, and update your preview environment. Let's open the Scully UI in the browser, and click on the `Mulder, tell me something` button... it still doesn't work. Because we deployed a Mulder instance, but we never told Scully to connect to it. Let's fix that.

## Updating the deployment

First, we'll need to know the Mulder "hostname" that Scully can use. In the Kubernetes world, applications talk together through services. So we'll need to list all the services in the namespace used by our preview environment. First, let's find the namespace in which our preview environment has been deployed. Each preview environment is deployed in its own namespace. If you run `jx get previews` you can see a list of all preview environments, with their namespaces. The namespace for the first PR should be `jx-XXX-scully-pr-1`.

```
$ kubectl get services -n jx-XXX-scully-pr-1
```

We see 3 services:
- our scully service
- a `jx-xxx-scully-pr-1-mulder` service - which is the one we need to connect to
- a `mulder-redis-master` service - because internally, Mulder uses Redis

The issue is that the name of the service is dynamic: it's based on the PR number. In fact, it's based on the Helm "release name". If you run `helm list`, you can list all the applications (releases) installed by Helm. Amongst them, there is our preview environment named `jx-xxx-scully-pr-1`.

Another way to find the relation between the preview and the Helm release, is to list the previews in YAML format, with `jx get previews -o yaml`: you will get the whole definition of the preview environment in YAML, including the `jenkins.io/chart-release` annotation, whose value is the Helm release name.

So, now that we have the Mulder service name, we can use it to configure Scully. If we check [Scully's README](https://github.com/the-jenkins-x-files/scully/blob/master/README.md) in the **Env variables** section it says that we need to use the `SERVER` environment variable to configure the Mulder hostname to connect to.

Let's update our Kubernetes [deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) to do that. Open the `charts/scully/templates/deployment.yaml` file. You should also open the [Deployment's API reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.13/#deployment-v1-apps) to have the documentation for each field. Locate the `containers` array - it should have only 1 container. This is where we'll need to add the environment variables for our application.

If you look at the [Container's API reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.13/#container-v1-core), you can see that it has an optional `env` field, whose value is an array of [EnvVar](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.13/#envvar-v1-core) - which is just an object with a `name` and a `value`:
- name: `SERVER`
- value: `http://{{ .Release.Name }}-mulder`. `{{ .Release.Name }}` is a special Helm variable that will be replaced by the name of the release - see [Helm's list of predefined values](https://docs.helm.sh/developing_charts/#predefined-values).

You can find the whole file at [deployment.yaml](deployment.yaml).

Time to commit! And to push, of course. This will trigger a new Jenkins build. You can check the progress of the build, and at the end, test if the Scully application has been re-deployed. If you run the `helm list` command, you can see that your release is now at its third revision. Running `helm history jx-xxx-scully-pr-1` will show you all revisions. And if you install the [helm-diff](https://github.com/databus23/helm-diff) plugin, you will be able to display the changes between revision 2 and 3, by running the command `helm diff revision jx-xxx-scully-pr-1 2 3`.

So, back to our application. If you try to visit its URL again, and click on the `Mulder, tell me something` button, this time you should see a random quote! Click again, and see a different quote!

Well, now that everything's working, it's time to merge the PR! This will automatically trigger a new PR in your "staging environment" git repository, which in turn will trigger a new Jenkins build - to deploy your new release to the staging environment.

Once its done, you can check the `jx-staging` Helm release to see its changes - using `helm history jx-staging` for example, or even `helm diff`. And if you open <http://scully.jx-staging.YOUR_DOMAIN/> in your browser, you should have a working Scully UI, that will give you random Mulder quotes!

One more comment on the PR that was merged. Closing or merging a PR will automatically delete the associated Preview Environment, but not in real time. There is a separate process, that runs at regular interval, and check for Preview Environments that should be deleted. It's called `jenkins-x-gcpreviews`, and is setup as a Kubernetes [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/). You can see it by running `kubectl get cronjobs` - and if you run `kubectl get jobs` you will see its last runs. So don't worry if your preview is still there even if the PR has been merged: it may take some time for the job to run and garbage-collect it.

So, everything done? Well, not yet. We have a functional CI/CD pipeline for our application, but so far we had to do manual testing. What about [running the integration tests automatically](run-tests.md)?
