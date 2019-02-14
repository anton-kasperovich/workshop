# Adding Mulder as a dependency

Our application is deployed as a [Helm](https://helm.sh/) chart, which means that we need to use Helm to manage our dependencies. The first step is to find a chart for [Mulder](https://github.com/the-jenkins-x-files/mulder).

Here, we have 2 solutions:
- if you already completed the [Mulder workshop](../mulder/README.md), you should have a chart for Mulder deployed in the internal repository. In this case, we will [re-use it](add-internal-mulder-chart.md)
- otherwise, don't worry. We already have a Mulder chart ready for you, and that's [the one we will use](add-external-mulder-chart.md)
