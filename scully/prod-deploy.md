# Production Deployment

You might have noticed that so far we only deployed to the **staging** environment. And what about the **production** environment?

If you open the `Jenkinsfile`, at the bottom it says that its promoting `through all 'Auto' promotion Environments`. What is an `Auto` environment? It's an environment where new releases will be automatically deployed. The alternative is `Manual` promotion environment, where we need to manually deploy the new releases.

Run the `jx get environments` to list all known environment, and their promotion type: `Auto` or `Manual`. You can see that the **staging** environment is setup for automatic promotion, but the **production** one is setup for manual promotion.

That means that if we want to deploy to production, we need to run the [jx promote](https://jenkins-x.io/commands/jx_promote/) command. First, we need to find the version to deploy. Run `jx get releases` to see all available releases. Then, run the manual promotion:

```
$ jx promote --verbose --env production --version 0.0.X
```

It will create a new PR in your "production environment" repository, merge it, and then deploy your application to production. At the end, it will tell you on which URL you can access your application. You can also get the URL by running the [jx get applications](https://jenkins-x.io/commands/jx_get_applications/) command:

```
$ jx get applications -e production
```

So our current pipeline is doing **Continuous Delivery**, not **Continuous Deployment**: you still need to run a manual command to deploy to production. What if we want to switch to continuous deployment? We have 2 solutions:

- update the production environment's configuration, using the `jx edit environment production` command, to switch to `Auto` promotion - that will apply to all applications.
- update the `Jenkinsfile` of an application to run the `jx promote --env production` command right after the `jx promote --all-auto` command - this will only apply to that single application.
