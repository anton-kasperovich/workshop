# Scully Workshop

[Scully](https://github.com/the-jenkins-x-files/scully) is the Frontend UI component used for the workshop. It is a [React](https://reactjs.org/) application and has a dependency on our [Mulder](https://github.com/the-jenkins-x-files/mulder) Backend API component.

The goal of the workshop is to **import** this repository into your Jenkins X cluster, and do the necessary changes to "make it work" - both inside the [Preview Environments](https://jenkins-x.io/about/features/#preview-environments) and the staging/production environments.

We will play with the following tools:
- [jx](https://github.com/jenkins-x/jx) - the Jenkins X CLI - of course
- [Jenkins](https://jenkins.io/)
- [Github](https://github.com/)
- [Git](https://git-scm.org/)
- `Jenkinsfile` pipeline
- [Helm](https://helm.sh/)
- [Skaffold](https://skaffold.dev/)
- [Docker](https://www.docker.com/)
- [ChartMuseum](https://chartmuseum.com/)
- [Kubernetes API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.13/)
- [kubectl](https://kubernetes.io/docs/reference/kubectl/overview/) - Kubernetes CLI

You don't need to master all of these tools. But we'll need to touch a little of each, see how they interact, and where to find documentation.

Ready? Let's start!

- First, [import Scully in Jenkins X](import.md)
- Then, we'll need to [fix the build](fix-build.md)
- And to [add Mulder as a dependency](add-mulder-dependency.md) to make it work
- We will also [setup and run the unit & integration tests](run-tests.md)
- And [deploy to production](prod-deploy.md)
