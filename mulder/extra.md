# Extra Steps

Let's finish with a few (optional) extra steps:

- [Kubernetes Healthcheck](#kubernetes-healthcheck)
- [Code change](#code-change)
- [Production Deployment](#production-deployment)

## Kubernetes Healthcheck

Kubernetes can check the health of your application, and either restart a failing pod or "disable" it, using [liveness and readiness probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/).

[Mulder's README](https://github.com/the-jenkins-x-files/mulder/blob/master/README.md) documents a `/healthz` endpoint, that can be used to retrieve the health of the application, and the connection to Redis. So, let's update our deployment to use this new endpoint, instead of the default `/` one. If you open the `charts/mulder/templates/deployment.yaml` file, you can see that it's using the `probePath` value. So let's update our `charts/mulder/values.yaml` file to set the new value for `probePath`:

```
probePath: /healthz
```

And create a new Git branch, commit, push, open a PR, make sure it works, and merge the PR.

## Code change

This time, let's change some of the code, and see what happens. For example, in the `main.go` file, you can remove all the existing lines in the `quotes` array, and add only 1 or 2 of your own quotes.

As usual, branch/commit/push/PR and make sure it works.

## Production Deployment

You might have noticed that so far we only deployed to the **staging** environment. And what about the **production** environment?

If you open the `Jenkinsfile`, at the bottom it says that its promoting `through all 'Auto' promotion Environments`. What is an `Auto` environment? It's an environment where new releases will be automatically deployed. The alternative is `Manual` promotion environment, where we need to manually deploy the new releases.

Run the `jx get environments` to list all known environment, and their promotion type: `Auto` or `Manual`. You can see that the **staging** environment is setup for automatic promotion, but the **production** one is setup for manual promotion.

That means that if we want to deploy to production, we need to run the [jx promote](https://jenkins-x.io/commands/jx_promote/) command. First, we need to find the version to deploy. Run `jx get releases` to see all available releases. Then, run the manual promotion:

```
$ jx promote --verbose --env production --version 0.0.X
```

It will create a new PR in your "production environment" repository, merge it, and then deploy your application to production. At the end, it will tell you on which URL you can access your application.

That means our current pipeline is doing **Continuous Delivery**, not **Continuous Deployment**: you still need to run a manual command to deploy to production. What if we want to switch to continuous deployment? We have 2 solutions:

- update the production environment's configuration, using the `jx edit environment production` command, to switch to `Auto` promotion - that will apply to all applications.
- update the `Jenkinsfile` of an application to run the `jx promote --env production` command right after the `jx promote --all-auto` command - this will only apply to that single application.

You are now ready to move to [Scully](../scully/README.md)!
