# Running the integration tests automatically

If we read [Scully's README](https://github.com/the-jenkins-x-files/scully/blob/master/README.md) you can see that it already has unit and integration tests.

## Unit Tests

If you open the `Jenkinsfile`, you can see that it's already running the `npm test` command, so it's already running our unit tests.

## Integration Tests

We'll need to do the same thing for the integration tests, or `E2E` - which stands for "End to End" - tests, as they are called in Scully. [Scully's README](https://github.com/the-jenkins-x-files/scully/blob/master/README.md) says that the command to run them is `npm run e2e`. To run, these tests requires the URL of a running Scully instance, passed through the `UI` environment variable. So, we need to find which value we can use to point our tests to our just-deployed Preview Environment. In a Kubernetes cluster, the easiest solution is to targer the internal service. [Kubernetes automatically manages DNS records for services](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#services), using the `SERVICE_NAME.NAMESPACE_NAME` syntax. So, we need to retrieve both the service name of our Scully application, and the namespace in which the Preview Environment has been deployed.

The service name is easy: it's defined in the `charts/scully/templates/service.yaml` file, under `metadata.name`. And in fact it's using the `service.name` value, which is defined in the `charts/scully/values.yaml`, and is set to... `scully`. All that for that! We can confirm by listing all the services in our cluster - with `kubectl get services --all-namespaces` - that our Scully services are named `scully`.

Now, how can we get the namespace used by the preview environment? If you look at the `Jenkinsfile`, you can see that in fact it's already defined as an environment variable named `PREVIEW_NAMESPACE`. Perfect! Except that if you do the math, it doesn't match: its defined as `$APP_NAME-$BRANCH_NAME`, and `APP_NAME` is set to `scully` - at the beginning of the file. But so far we've observed that the namespace used by our first preview environment was `jx-XXX-scully-pr-1`. Why doesn't it match? Just because the `PREVIEW_NAMESPACE` environment variable is not used when deploying the preview environment! If you run `jx preview -h`, you can see that there is a `--namespace` flag that is not used in the `Jenkinsfile`.

So, let's fix the `jx preview` command, and add a new step after the preview environment deployment, to execute the integration tests:

```
container('nodejs') {
  [...]
  dir('./charts/preview') {
    sh "make preview"
    sh "jx preview --app $APP_NAME --namespace $PREVIEW_NAMESPACE --dir ../.."
  }
  sh "UI=http://scully.$PREVIEW_NAMESPACE/ npm run e2e"
}
```

Let's create a new Git branch, commit our changes, and push. Then open a new Pull Request on Github, and... let Jenkins work ;-)
Enjoy the green Jenkins build! It's time to merge your PR. And to update your local `master` branch.

This time we have a fully functional CI/CD pipeline with a safety net, so we can start developing new features with confidence! And [deploy to production](prod-deploy.md).
