# Mulder Workshop

[Mulder](https://github.com/the-jenkins-x-files/mulder) is the Backend API component used for the workshop. It is written in [Go](https://golang.org/) and has a dependency on a [Redis](https://redis.io/) database.

The goal of the workshop is to **import** this repository into your Jenkins X cluster, and do the necessary changes to "make it work" - both inside the [Preview Environments](https://jenkins-x.io/about/features/#preview-environments) and the staging/production environments.

We will play with the following tools:
- [jx](https://github.com/jenkins-x/jx) - the Jenkins X CLI - of course
- [Jenkins](https://jenkins.io/)
- [Github](https://github.com/)
- [Git](https://git-scm.org/)
- [Go](https://golang.org/)
- `Makefile`
- `Jenkinsfile` pipeline
- [Helm](https://helm.sh/)
- [Kubernetes API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.13/)
- [kubectl](https://kubernetes.io/docs/reference/kubectl/overview/) - Kubernetes CLI

You don't need to master all of these tools. But we'll need to touch a little of each, see how they interact, and where to find documentation.

Ready? Let's start!

- First, [import Mulder in Jenkins X](import.md)
- Then, we'll need to [add Redis as a dependency](add-redis-dependency.md) to make it work
- We will also [setup and run the unit & integration tests](run-tests.md)
- And finish with a few (optional) [extra steps](extra.md) - including deploying to production

At the end, you should have a fully functional CI/CD pipeline with a safety net - the automated tests - so it's time to move to [Scully](../scully/README.md)!
