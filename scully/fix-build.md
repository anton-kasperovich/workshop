# Fixing the build

If we open [Scully's README](https://github.com/the-jenkins-x-files/scully/blob/master/README.md), in the **Building** section we can see that we need to run the `npm run build` command. This is because it's a [React](https://reactjs.org/) application, so it needs to be "build" - or "transpilled" - before it can be run.

First, we will open the generated `Jenkinsfile`, and we can see that it is running the following commands:
- `npm install`
- `npm test`
- `skaffold build`

The interesting one is the last one. [Skaffold](https://github.com/GoogleContainerTools/skaffold) is a tool that *handles the workflow for building, pushing and deploying your application*. You can open the `skaffold.yaml` file to see its configuration. The interesting part is the `build` one, and we can see that it's using [Docker](https://www.docker.com/), using the default `Dockerfile`.

So let's open the `Dockerfile` now, and we can see that it's just copying files, but it's never running the `npm run build` command. So we'll need to fix that! Just add a `RUN` instruction after the `COPY` one, to run our build command. 
You can find the whole file at [Dockerfile](Dockerfile).

Now, Docker will "build" our application and produce a container image with all the required files. Perfect! Let's create a new git branch, commit your changes, and push them to your fork.
You might have notice that our CI/CD pipeline is configured to only build Pull Requests - see the `when { branch 'PR-*' }` condition in the `Jenkinsfile` - so you will need to create a Pull Request for your branch. Open github in your browser - you can use `jx repo` to do that - and create a new PR. Take care that by default, Github will try to be smart, and will set the "base repository" of the Pull Request to [the-jenkins-x-files/scully](https://github.com/the-jenkins-x-files/scully) instead of your fork at `YOUR_USER_NAME/scully`. You will need to fix the "base repository" field in the UI before creating the PR, otherwise your Jenkins will never build this PR.

Next, Jenkins will start a new job to build your PR - and to create a preview environment. It will also comment on the PR in github with the link to the application. Let's open it: this time it's working, you get to see Scully! Now click on the `Mulder, tell me something` button. This will send a request to our [Mulder](https://github.com/the-jenkins-x-files/mulder) backend API, but as you can see it doesn't work. Because we need to [add Mulder as a dependency](add-mulder-dependency.md) so that Scully can talk to it.
