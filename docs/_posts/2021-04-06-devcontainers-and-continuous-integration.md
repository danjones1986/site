---
layout: post
title: Devcontainers and Continuous Integration
subtitle: Azure Pipelines
description: How to use devcontainers to create a consistent development and continuous integration work flow.
date: 2021-04-06 08:00:00
author: Dan Jones
tags: 
- github-actions 
- vscode 
- devcontainers 
- docker
- azure-pipelines
menubar_toc: true
---

### Introduction to devcontainers
Devcontainers within Visual Studio Code (VS Code) is a powerful concept, being able to create a Docker container with all the tooling and runtime stacks a developer needs to work on a given project, without having to install anything but [Docker](https://docs.docker.com/get-started/), [VS Code](https://code.visualstudio.com/) and the [VS Code Remote Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) extension.

Essentially devcontainers are VS Code workspace instances that live within the Docker container, when first opening the container the workspace instance is installed and connected to the VS Code UI working in the browser or your local machine.

![VS Code remote containers architecture diagram](https://code.visualstudio.com/assets/docs/remote/containers/architecture-containers.png)
*The above diagram is taken from the VS Code [website](https://code.visualstudio.com/docs/remote/containers)*

Probably you've heard a lot about GitHub Codespaces lately, an upcoming feature in GitHub, which essentially uses devcontainers to make its features work.

### What has this got to do with continuous integration?
Devcontainers as a concept have little to do with continuous integration (CI), but the Docker container used for the devcontainer to run in does. Your dockerfile contains all the specific versions of tools and runtime stacks you need to do the work. 

When moving to CI you then need to make your CI runner build and test the application, meaning you now need to ensure the runner has all the correct tooling with the same versions to do the job correctly. Though having slightly different versions can make little difference, for consistency we want to ensure what the developer is developing with the same tools used in CI.

### What are we going to do?
We will create a simple application to run within VS Code devcontainers and set up testing, then move to the CI to run the build and test process there within the Docker container. We will be using GitHub actions for the example, but the concepts work with most modern CI systems.

*Before continuing, ensure you have Docker and VS Code installed on to your machine.*

### Create a devcontainer workspace
We want to build a React web app, so we need a container that has NodeJS installed, we can create our own Dockerfile with all our tools and configurations setup, or pull an existing image from a Docker repo, for this app, we will use the Microsoft standard NodeJS devcontainer.

Firstly, lets create our workspace.

```bash
mkdir ci-sample-app
cd ci-sample-app
```

Next create a folder `.devcontainer` and inside add a new file `.devcontainer.json` and inside add the image we want to use as our devcontainer.

```json
{
  "image": "mcr.microsoft.com/vscode/devcontainers/typescript-node:0-12"
}
```

Next we will install the `Remote - Containers` extension for VS Code. You can install this from the extension's menu in VS Code or if you have the VS Code path reference set, you can run a command in the root of the project.

```bash
code --install-extension ms-vscode-remote.remote-containers
```

You may need to restart VS Code and once you do, you will be asked if you would like to open the project in a devcontainer.

![Open in devcontainer](/img/blog/open-in-dev-container.png)

Click reopen. The first time you do this it will pull the Docker image down from the remote repository, which may take some time.

When everything is complete, everything will still look the same as before, but the VS Code workspace is now running exclusively inside the Docker container.

### Creating the React web app
Now we are running in the context of the container, we will create the sample React app and get it running using [npm](https://www.npmjs.com/package/npm) as our package manager.

Using the VS Code terminal, which can be opened from the top menu under `Terminal`, `New Terminal`, run the following to create a new React app.

```bash
npx create-react-app app
cd app
rm -rf .git # remove git binding as we want to initialize in the root directory
npm start
```

This will start the React app on port 3000, but if you try to access [http://localhost:3000/](http://localhost:3000/), you will notice you can't, this is because we first need to expose the port from within the devcontainer, which can be done inside `devcontainer.json`.

```json
{
  "image": "mcr.microsoft.com/vscode/devcontainers/typescript-node:0-12",
  "forwardPorts": [3000]
}
```

Inside the `.devcontainer.json` you can also add VS Code extensions, so every time someone opens the devcontainer they are greeted with the same IDE setup, for example, you may want to add an ESLint extension.

```json
{
  "image": "mcr.microsoft.com/vscode/devcontainers/typescript-node:0-12",
  "extensions": ["dbaeumer.vscode-eslint"],
  "forwardPorts": [3000]
}
```

There are extensions to add Unit Test UI for easy of testing and many, many more.

When editing .devcontainer within the devcontainer, we need to rebuild the container to get the changes.

At the bottom left of VS Code you will see a section containing the words `Dev Container`.

![Dev Container](/img/blog/vscode-devcontainer-bottom-left.png)

Clicking this will bring up a menu at the top, select "Remote-Containers: Rebuild Container".

![Rebuild Container](/img/blog/rebuild-devcontainer.png)

You should now be able to visit [http://localhost:3000/](http://localhost:3000/) and see the basic React app up and running.

The sample contains tests, which can be run using npm.

```bash
npm test
```

You can create a release bundle by running `npm run build`, which will output the content to a folder called `build`.

### CI with GitHub Actions
For this next bit, we will use GitHub Actions to build and test our app using the Docker image used within the devcontainer above.

Create folders in the root of the project as `.github\workflows`, then create a file called `build.yml` inside the `workflows` folder, this will contain our CI code.

We will be using a Linux Hosted machine to do our build, which supports container jobs.

```yml
name: Build

on: push

jobs:
  container:
    runs-on: ubuntu-latest
    container: mcr.microsoft.com/vscode/devcontainers/typescript-node:0-12
    steps:
```

Our pipeline is now defined to pull down the same container we used inside our devcontainer and when it runs the `steps` (which we will define next), these will be run in the context of the container.

We can run the predefined npm tasks with GitHub Actions, and it will run against our version of npm, or we can run the same commands we have already used to keep consistency.

```yml
name: Build

on: push

jobs:
  container:
    runs-on: ubuntu-latest
    container: mcr.microsoft.com/vscode/devcontainers/typescript-node:0-12
    steps:
      - uses: actions/checkout@v2

      - run: npm install
        name: Install npm packages
        working-directory: app
      
      - run: CI=true && npm test
        name: Test React application
        working-directory: app
      
      - run: CI=true && npm run build
        name: Build React app bundle
        working-directory: app

      - uses: actions/upload-artifact@v2
        with:
          name: app
          path: app/build
```

The last command, will take our build and publish it to the artifacts section of the pipeline run.

Now our code is written we will push it up to a GitHub repo.

```bash
git init
git add .
git commit -m "devcontainer CI sample"
```

Next we will publish to our remote repo (You will want to substitute the GitHub repo with your own).

```bash
git remote add origin https://github.com/skilledcookie/devcontainers-and-ci.git
git branch -M main
git push -u origin main
```


This should automatically run the CI pipeline, which you can see under "Actions" in the repository. [See this in our GitHub repo now!](https://github.com/skilledcookie/devcontainers-and-ci/actions)

### CI in Azure Pipelines example
Azure Pipelines is very similar to GitHub Actions syntax, if you would like to run this example using Azure DevOps, below is the equivalent to the script posted above.

```yml
pool:
  vmImage: 'ubuntu-latest'

container: "mcr.microsoft.com/vscode/devcontainers/typescript-node:0-12"

steps:
- bash: npm install
  displayName: Install npm packages
  workingDirectory: app

- bash: npm test
  displayName: Test React application
  workingDirectory: app
  env:
    CI: true

- bash: npm run build
  displayName: Build React app bundle
  workingDirectory: app
  env:
    CI: true

- publish: app/build
  artifact: app
  displayName: Publish to Pipeline Artifacts
```

### More patterns
Above you saw a basic example of how you can use the same Docker container you use in devcontainers in your CI pipelines, but there are other patterns you can use alongside this to add more flexibility to your development workflow.

* **Use a Dockerfile** to customize your container adding more tools specific to your development and CI workflow, for example adding Terraform to manage your infrastructure.
* **Remote Docker repositories** like Docker Hub or Azure Container Registry can store and share your private or public images across projects, without the need to build each time.
* **Use Docker tags** to version your Docker images, so devcontainers and CI can target a specific version, and you can upgrade when you choose, or force the use of the `latest` tag to ensure everyone is always working with the latest development container.

### Exciting times ahead
Overall devcontainers are a very powerful concept and bringing the same container in to your CI pipeline ensures consistency in your development workflow, ensuring you aren't using different tools or versions of tools in either place. 

It will be interesting to see how the devcontainer landscape evolves once GitHub Codespaces is finally released to the public.

### Related resources
* [Git Hub actions](https://docs.github.com/en/actions)
* [Azure Pipelines](https://azure.microsoft.com/en-gb/services/devops/pipelines/)
* [Developing inside a container](https://code.visualstudio.com/docs/remote/containers)
* [Getting started with Docker](https://docs.docker.com/get-started/)
* [Getting started with VS Code](https://code.visualstudio.com/docs)

### Source code
The source code for this post can be found at [https://github.com/skilledcookie/devcontainers-and-ci](https://github.com/skilledcookie/devcontainers-and-ci)