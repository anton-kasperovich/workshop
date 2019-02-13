# Running the unit & integration tests automatically

If we read [Mulder's README](https://github.com/the-jenkins-x-files/mulder/blob/master/README.md) you can see that it already has unit and integration tests.

## Unit Tests

Let's start with the unit tests. We'll need to update the `Makefile` to add our command to run the unit tests, and then update the `Jenkinsfile` to call our new target. Let's name this new target `test-unit`, to follow the [gazr](https://gazr.io/) convention:

```
test-unit:
	$(GO) test -v .
```

And in the `Jenkinsfile`, we'll execute the unit tests before building the application:

```
dir('/home/jenkins/go/src/github.com/xxx/mulder') {
  checkout scm
  sh "make test-unit"
  sh "make linux"
  [...]
}
```

Let's create a new Git branch, commit our changes, and push. Then open a new Pull Request on Github, and... let Jenkins work ;-)

## Integration Tests

We'll follow the same steps for the integration tests, starting with the `Makefile`. The new target will be named `test-integration`, based on the [gazr](https://gazr.io/) convention:

```
test-integration:
	$(GO) test -v ./tests -addr ${MULDER_ADDR}
```

To run, the integration tests requires the host and port of a running Mulder instance, passed through the `-addr` flag. And when executing them via `make`, we'll need to pass it using the `MULDER_ADDR` argument: `make test-integration MULDER_ADDR=localhost:8080` for example. So, we need to find which value we can use to point our integration tests to our just-deployed Preview Environment. In a Kubernetes cluster, the easiest solution is to targer the internal service. [Kubernetes automatically manages DNS records for services](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#services), using the `SERVICE_NAME.NAMESPACE_NAME` syntax. So, we need to retrieve both the service name of our Mulder application, and the namespace in which the Preview Environment has been deployed.

The service name is easy: it's defined in the `charts/mulder/templates/service.yaml` file, under `metadata.name`. And in fact it's using the `service.name` value, which is defined in the `charts/mulder/values.yaml`, and is set to... `mulder`. All that for that! We can confirm by listing all the services in our cluster - with `kubectl get services --all-namespaces` - that our Mulder services are named `mulder`.

Now, how can we get the namespace used by the preview environment? If you look at the `Jenkinsfile`, you can see that in fact it's already defined as an environment variable named `PREVIEW_NAMESPACE`. Perfect! Except that if you do the math, it doesn't match: its defined as `$APP_NAME-$BRANCH_NAME`, and `APP_NAME` is set to `mulder` - at the beginning of the file. But so far we've observed that the namespace used by our first preview environment was `jx-XXX-mulder-pr-1`. Why doesn't it match? Just because the `PREVIEW_NAMESPACE` environment variable is not used when deploying the preview environment! If you run `jx preview -h`, you can see that there is a `--namespace` flag that is not used in the `Jenkinsfile`.

So, let's fix the `jx preview` command, and add a new step after the preview environment deployment, to execute the integration tests:

```
dir('/home/jenkins/go/src/github.com/XXX/mulder/charts/preview') {
  sh "make preview"
  sh "jx preview --app $APP_NAME --namespace $PREVIEW_NAMESPACE --dir ../.."
}
dir('/home/jenkins/go/src/github.com/XXX/mulder') {
  sh "make test-integration MULDER_ADDR=mulder.$PREVIEW_NAMESPACE"
}
```

Commit your changes and push them. And... the Jenkins build should fail, with a `Connection timed out` error, in the integration tests step. This is because we are running our integration tests too quickly after the preview environment deployment, and it's not ready yet. So we need to wait a little before running our integration tests. We have 2 solutions:

- use the `sleep` command, quick and dirty
- send requests to our application's endpoint until it's ready. We can use [curl](https://curl.haxx.se/) or [wget](https://www.gnu.org/software/wget/) to do that.

Let's use [wget](https://www.gnu.org/software/wget/), with the right options to retry in case of failure:

```
$ wget --server-response --output-document=/dev/null --timeout=60 --tries=10 --retry-connrefused URL
```

Note that `wget` has a [retry-on-http-error](http://tomszilagyi.github.io/2017/02/Wget-retry-on-http-error) flag, but it's too recent and is not in the version provided in the Docker image used by our Jenkins build.

If we add our new `wget` command in the `Jenkinsfile`, just after the deployment:

```
dir('/home/jenkins/go/src/github.com/XXX/mulder/charts/preview') {
  sh "make preview"
  sh "jx preview --app $APP_NAME --namespace $PREVIEW_NAMESPACE --dir ../.."
  sh "wget --server-response --output-document=/dev/null --timeout=60 --tries=10 --retry-connrefused http://mulder.$PREVIEW_NAMESPACE/"
}
```

Let's commit and push again, and enjoy the green Jenkins build! It's time to merge your PR.

This time we have a fully functional CI/CD pipeline with a safety net, so we can start developing new features with confidence!

You can either:
- finish with a few (optional) [extra steps](extra.md) - including deploying to production
- or move directly to [Scully](../scully/README.md)
