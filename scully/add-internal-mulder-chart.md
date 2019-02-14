# Adding Mulder as a dependency

**Re-using our Mulder chart**

If you already completed the [Mulder workshop](../mulder/README.md), you should have a chart for Mulder deployed in the internal repository.

## Retrieving the Mulder chart and image

Jenkins X comes with a few components that are automatically deployed in the Kubernetes cluster:
- [ChartMuseum](https://chartmuseum.com/) to store the [Helm](https://helm.sh/) charts
- [Docker Registry](https://docs.docker.com/registry/) to store the [Docker](https://docker.com/) images

You can get the URLs of these components by running the [jx get urls](https://jenkins-x.io/commands/jx_get_urls/) command:

```
$ jx get urls
```

If you can also see all the Docker images stored in the Docker registry, by opening the `/v2/_catalog` endpoint on the Docker registry URL, and then the `/v2/YOUR_USER_NAME/mulder/tags/list` endpoint. You can see 1 image tag per release, plus one per PR build.

And if you open the `/api/charts` endpoint on the ChartMuseum URL, you should get back a JSON response with a list of all the charts stored in this repository.

Another way to interact with ChartMuseum is to use your Helm CLI:

```
$ helm repo add jx-workshop http://chartmuseum.jx.XXX.nip.io
$ helm repo update
$ helm search mulder
```

You should see a chart named `jx-workshop/mulder`. And you can inspect its content by running `helm inspect jx-workshop/mulder`.

## New dependency

To add a dependency on the Mulder chart, you will need to add a new `requirements.yaml` file in the `charts/scully` directory. Exactly as we did in the Mulder chart, to add a dependency on the Redis chart. The repository URL should be the internal DNS for the ChartMuseum service, that you can find by listing all the services: `kubectl get services`. You can see that the service is named `jenkins-x-chartmuseum` and is listening on port `8080` - so the internal URL is `http://jenkins-x-chartmuseum:8080`. And you can see the latest version of Mulder by running `helm search mulder` or `jx get releases`.

And as you already know, adding a dependency in the `requirements.yaml` file is not enough: you also need to run the [jx step helm build](https://jenkins-x.io/commands/jx_step_helm_build/) command in the `Jenkinsfile`:

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

First, we'll need to know the Mulder "hostname" that Scully can use - that means the name of the Mulder service. If you remember from the [Mulder workshop](../mulder/README.md), the service name is just `mulder`. But we can check that by listing all services in the namespace used by our preview environment. Let's first run `jx get previews` to get a list of all preview environments, with their namespaces. The namespace for the first PR should be `jx-XXX-scully-pr-1`. You can now see all services with the following command:

```
$ kubectl get services -n jx-XXX-scully-pr-1
```

We see a few services, including the `mulder` one. We can also confirm that our Mulder chart came with its Redis dependency, because we can see the redis services. And of course the `scully` service.

So, now that we have the Mulder service name, we can use it to configure Scully. If we check [Scully's README](https://github.com/the-jenkins-x-files/scully/blob/master/README.md) in the **Env variables** section it says that we need to use the `SERVER` environment variable to configure the Mulder hostname to connect to.

Let's update our Kubernetes [deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) to do that. Open the `charts/scully/templates/deployment.yaml` file. You should also open the [Deployment's API reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.13/#deployment-v1-apps) to have the documentation for each field. Locate the `containers` array - it should have only 1 container. This is where we'll need to add the environment variables for our application.

If you look at the [Container's API reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.13/#container-v1-core), you can see that it has an optional `env` field, whose value is an array of [EnvVar](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.13/#envvar-v1-core) - which is just an object with a `name` and a `value`:

```
env:
  - name: SERVER
    value: http://mulder
```

Time to commit! And to push, of course. This will trigger a new Jenkins build. You can check the progress of the build, and at the end, test if the Scully application has been re-deployed. If you run the `helm list` command, you can see that your release is now at its third revision. Running `helm history jx-xxx-scully-pr-1` will show you all revisions. And if you install the [helm-diff](https://github.com/databus23/helm-diff) plugin, you will be able to display the changes between revision 2 and 3, by running the command `helm diff revision jx-xxx-scully-pr-1 2 3`.

So, back to our application. If you try to visit its URL again, and click on the `Mulder, tell me something` button, this time you should see a random quote! Click again, and see a different quote!

Well, now that everything's working, it's time to merge the PR! This will automatically trigger a new PR in your "staging environment" git repository, which in turn will trigger a new Jenkins build - to deploy your new release to the staging environment.

But... the deployment to the staging environment never finishes. If you check the Helm releases - with `helm list -a` - you can see that the latest revision of the `jx-staging` release is in `PENDING_UPGRADE` status for a while, and then it changes to the `FAILED` status. In the output of the Jenkins build that did the deployment, you can see the following error:

> Error: UPGRADE FAILED: timed out waiting for the condition

This is because Helm waits for every Kubernetes resource to be ready, but here, for some reason, some resources were not created. Let's find the reason for this! We can start by listing all revisions of the `jx-staging` release, using the `helm history jx-staging`.

Then, if you didn't install the [helm-diff](https://github.com/databus23/helm-diff) plugin, now is a good time to do it. We'll use it to see the diff between the latest revision and the previous one. If the `FAILED` revision was the 8th, then you need to run:

```
$ helm diff revision jx-staging 7 8
```

This will show you the diff between revision 7 and 8. And you can see the following errors at the beginning of the output:

> Error: Found duplicate key "jx-staging, mulder, Service (v1)" in manifest

So we have 2 Mulders? Yes, because we already have an existing Mulder instance running in the staging environment, and when we tried to deploy Scully, it also tried to deploy a new Mulder as a dependency. We don't need 2 Mulder instances in staging or production, we should connect our Scully application to our already deployed Mulder instance. So we need to somehow disable the dependency for staging/production deployments.

## Optional dependency

As explained in [Helm's documentation](https://docs.helm.sh/developing_charts/#tags-and-condition-fields-in-requirements-yaml), dependencies can have a `condition` field. And because we only need the Mulder dependency for the Preview Environments, we're going to disable it by default, but enable it only for the previews.

To do that, we'll update our `charts/scully/requirements.yaml` file to add a new `condition` field, set to the `mulder.enabled` value.

We'll then update the `charts/scully/values.yaml` file to set our default value: add a new `mulder` section, with the `enabled` key,and `false` as its value:

```
mulder:
  enabled: false
```

This means that by default, the Mulder dependency won't be used.

And the last step is to enable the dependency for the previews. We can do that in the `charts/preview/values.yaml`, and add the same lines, except with the `true` value. This means that when deploying the preview chart, we will also deploy the mulder chart.

Create a new branch for your change, commit, push and create a new Pull Request. Make sure that the new Preview Environment has been created with everything: Scully, Mulder and Redis. And then merge the PR - this will trigger a new deployment to the staging environment, which should be successful this time. Open <http://scully.jx-staging.YOUR_DOMAIN/> in your browser, you should have a working Scully UI, that will give you random Mulder quotes!

So, everything done? Well, not yet. We have a functional CI/CD pipeline for our application, but so far we had to do manual testing. What about [running the integration tests automatically](run-tests.md)?
