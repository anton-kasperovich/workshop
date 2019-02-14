# Integration Workshop

If you implemented CI/CD pipelines for both [Mulder](https://github.com/the-jenkins-x-files/mulder) and [Scully](https://github.com/the-jenkins-x-files/scully), you are ready to go to the next step: the integration between both. For example, you might have noticed that we hardcoded the version of Mulder in the Scully chart. That means we will need to update it manually?

This is where Jenkins X shines: because it is really a set of practices and tools, and it comes with a very interesting one: [updatebot](https://github.com/jenkins-x/updatebot). This little tool can be used to update Scully's `requirements.yaml` file each time a new Mulder version is released. With it, your workflow becomes:
- new Mulder release
- **updatebot** creates a PR on the Scully repository, with the new version of Mulder in the chart dependencies
- Jenkins starts a build of the new PR, deploying a new Preview Environment that includes the latest Mulder release

And you will see that it is very easy to setup and use. The first step is to create a new `.updatebot.yml` file at the root of the Mulder repository, with the following content:

```
github:
  organisations:
  - name: YOUR_GITHUB_USER
    repositories:
    - name: scully
```

Of course, don't forget to replace `YOUR_GITHUB_USER` with your real github user.

This file is used to define all the repositories that needs to be updated for each new release of Mulder. Here, we only cares about updating the Scully repository.

The second step is to update the `Jenkinsfile` to call the `updatebot` tool at the end of the release:

```
stage('Promote to Environments') {
  steps {
    container('go') {
      dir('/home/jenkins/go/src/github.com/XXX/mulder/charts/mulder') {
        [...]
        // release the helm chart
        sh "jx step helm release"
        [...]
      }
      dir('/home/jenkins/go/src/github.com/XXX/mulder') {
        // update the chart version everywhere it's referenced
        sh "updatebot push-version --kind helm mulder \$(cat VERSION)"
      }
    }
  }
}
```

We just added the following line at the end, in the right directory:

```
updatebot push-version --kind helm mulder \$(cat VERSION)
```

This will instruct `updatebot` to push a new version in every Helm dependency file - `requirements.yaml` - if it can find a dependency on the `mulder` chart.

Commit and push your changes, and then check your Scully repository: a new PR will be created automatically. Nice! Let Jenkins run the integration tests for you, and merge the change when you are confident that everything is working as it should.
