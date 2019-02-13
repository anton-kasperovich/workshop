# Jenkins X Workshop

Welcome to **The Jenkins X Files** - the [Jenkins X](https://jenkins-x.io/) workshop. Or yet-another-jenkins-x-workshop.

## Why this workshop?

Because we wanted a simple-enough set of applications that people can play with, and that represent more or less (part of) our stack:
- a [React](https://reactjs.org/) frontend UI -> [Scully](https://github.com/the-jenkins-x-files/scully)
- a [Go](https://golang.org/) backend API -> [Mulder](https://github.com/the-jenkins-x-files/mulder)
- a database - here we are using [Redis](https://redis.io/) because it's easy to setup and use, but any database would do

The **goal** of this workshop is to write the CI/CD pipelines for the whole stack, connecting applications together.

## Intial Setup

Before starting, you'll need to [create a new Jenkins X cluster](https://jenkins-x.io/getting-started/create-cluster/). This is well documented in the [Jenkins X Documentation](https://jenkins-x.io/documentation/).

For example, on [GKE](https://cloud.google.com/kubernetes-engine/) we used the following command:

```
$ jx create cluster gke -n jx-workshop -p MY_GCP_PROJECT -z europe-west1-b -m n1-standard-2 --min-num-nodes 3 --max-num-nodes 5 --default-admin-password=PLEASE_CHANGE_ME --default-environment-prefix jx-workshop
```

and then just used the default options everywhere.

Note that if you have an error while creating the new cluster, you can add the `--verbose` flag to get more details - usually a permission issue if you are using a public cloud provider.

## Let's get to work!

Now that everything is ready, you need to make a choice: Mulder or Scully? But don't worry, it can also be both!

You need at least to start somewhere. If you plan to do both, you should start with Mulder: implement the CI/CD pipeline for the backend API first, and then deploy the frontend UI on top of it. That way, you will be able to re-use your Mulder [Helm](https://helm.sh/) chart when deploying Scully.

- Start here for the [Mulder Workshop](mulder/README.md)
- Or here for the [Scully Workshop](scully/README.md)
