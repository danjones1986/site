---
layout: post
title: Manipulating Pipelines with Templates and Expressions
subtitle: Azure Pipelines
description: How to manipulate an Azure Pipeline using templates and expressions to enforce compliance, improve security and more.
date: 2021-03-18 12:00:00
author: Dan Jones
tags: azure-pipelines azure-devops templates
menubar_toc: true
---

### Introduction
Beyond creating standard Azure DevOps Pipelines, we can use templates to manipulate a pipelines definition. We can make custom steps run before or after tasks, or add stages based on certain conditions, or choose to reject a pipeline if certain definitions may or may not exist.

By passing the stages, jobs or steps to a template, we can within the template read the definition, and manipulate it how we see fit.

When passing definitions to a template, you must pass them as a specific data type, you can find a list in the [Pipelines runtime parameters](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/runtime-parameters?view=azure-devops&tabs=script#parameter-data-types) documentation.

Data types relevant to the context of manipulating our pipeline definitions with templates include the following.

| Data type        | Notes           | 
| ------------- |:-------------:| 
| step      | a single step | 
| stepList      | sequence of steps | 
| job     | a single job | 
| jobList | sequence of jobs | 
| deployment | a single deployment job |
| deploymentList | sequence of deployment jobs |
| stage | a single stage |
| stageList | sequence of stages |

### Passing steps into a template
We will create a template (`build-policy.yml`), which takes pipeline steps as it's parameter.

{% raw %}
```yml
parameters:
- name: stepList
  type: stepList
  default: []

steps: ${{ parameters.stepList }}
```
{% endraw %}

This template will just run the given steps. It can be used like the below example.

{% raw %}
```yml
- template: build-policy.yml
  parameters:
    stepList:
    - script: echo "Hello World 1"
    - script: echo "Hello World 2"
```
{% endraw %}

### Intercepting and injecting steps

What if we wanted to run our own steps before or after the pipeline steps ran in to the template? We would do this by looping through the steps list and render each step individually, which will allow us to write our own tasks before or after.

Going back to our `build-policy.yml` example, it would now look like the below.

{% raw %}
```yml
parameters:
- name: stepList
  type: stepList
  default: []

steps:
- script: echo "Begin running pipeline steps"
- ${{ each step in parameters.stepList }}:
  - ${{ each pair in step }}:
      ${{ pair.key }}: ${{ pair.value }}  
- script: echo "Finished running pipeline steps"
```
{% endraw %}

When run the pipeline definition will execute the same as below.

```yml
steps:
- script: echo "Begin running pipeline steps"
- script: echo "Hello World 1"
- script: echo "Hello World 2"
- script: echo "Finished running pipeline steps"
```

### Iterate through the steps
{% raw %}
In the above example you will notice `- ${{ each pair in step }}:`, which loops through the top level definition of the steps, which is a little confusing at first, but in reality, it's pretty simple, which I will try to clarify with an example.
{% endraw %}

{% raw %}
```yml
parameters:
  demo:
    - segment: first
      action: test
      value: 1
    - segment: second
      action: run
      value: 2

steps:
- ${{ each demoItem in parameters.demo }}:
  - ${{ each pair in demoItem.segment }}:
    - script: echo "${{ pair.key }} - ${{ pair.value }}"
```
{% endraw %}

The above example will output the following.

```
segment - first
action -  test
value - 1
segment - second
action - run
value - 2
```

### Throwing compile time errors with conditions

When a pipeline is first executed, as part of the normal checks, it will run any logic and perform any substitutions within `${{ }}` syntax, if this fails for any reason, it will output errors before starting the pipeline run, we can take advantage of this, by providing our own custom exception output.

In the above examples, we use the `script` task, which depending on the OS will execute in `bash` or `powershell`, which isn't great for reusability, so what if we want to stop our pipeline from using it altogether?

Our first step is to edit our definition and if the `script` task is detected, throw an error at compile time to stop the pipeline even starting.

{% raw %}
```yml
parameters:
- name: stepList
  type: stepList
  default: []

steps:
- bash: echo "Begin running pipeline steps"
- ${{ each step in parameters.stepList }}:
  - ${{ each pair in step }}:
      ${{ if eq(pair.key, 'script') }}:
        'script tasks are not allowed!': error
      ${{ if ne(pair.key, 'script') }}:
        ${{ pair.key }}: ${{ pair.value }}  
- bash: echo "Finished running pipeline steps"
```
{% endraw %}

There aren't many options for exceptions in Azure Pipelines, but by providing a message, followed by `: error`, an exception will be thrown at compile time, which will output your message on screen.

If we want to say throw an error if a condition is met, else it will allow the predefined step to run, we do this with 2 `if` statements of opposing condition, since there is no `if, else` syntax within pipelines.

### Enforcing standards with extension templates

Lastly what if we wanted to enforce this throughout the pipeline? We do this by changing how we use the template in the pipeline and use the `extends` syntax.

{% raw %}
```yml
extends:
  template: build-policy.yml
  parameters:
    stepList:
    - script: echo "Hello World 1"
    - script: echo "Hello World 2"
```
{% endraw %}

The `extends` syntax is used at the top of the pipeline, with any `steps`, `jobs` or `stages` being passed in as parameters, so the template will have access to the entire set of pipeline tasks.

Extension templates can optionally be stored in a shared repository and used within a pipeline of a different repository, which is great for shared company policy templates.

If you moved our templates file in to a repository called `automation`, we could then implement it by using a repository resource similar to the one below.

```yml
resources:
  repositories:
  - repository: automation  # identifier
    type: git 
    name: fabrikam/automation  # repository name (format depends on `type`)
    ref: refs/heads/master  

extends:
  template: build-policy.yml@automation
  parameters:
    stepList:
    - script: echo "Hello World 1"
    - script: echo "Hello World 2"
```

### Things to be aware of

When consuming extension templates from a shared repository, you need to be aware of the following;

*1* - Local templates must have `@self` on the end when being imported in to the pipeline definition.

Before using an extension template.
```yml
# ci/azure-pipelines.yml
...
steps:
- template: templates/build.yml
```

After applying an extension template.
```yml
# ci/azure-pipelines.yml
...
steps:
- template: ci/templates/build.yml@self
```

*2* - Local template imports now become relative to `$(System.DefaultWorkingDirectory)`, instead of the pipeline directory (shown in the above example).

*3* - If you have other templates you wish to use inside the extension template context, from the shared repository, they will need to be included relative to the extension templates folder.

If we have an extension template in a shared repository, lets say `extensions/policy.yml` and a reusable build template in the same repository `templates/build.yml`, then in the pipeline, we would use them together as follows.

```yml
resources:
  repositories:
  - repository: automation  # identifier
    type: git 
    name: fabrikam/automation  # repository name (format depends on `type`)
    ref: refs/heads/master  

extends:
  template: extensions/policy.yml@automation
  parameters:
    stepList:
    # relative to extensions folder
    - template: ../templates/build.yml@automation
```

### Powerful stuff

By using the examples above in regard to `steps` (stepList), you can apply the same processes over `jobs` (jobsList) and `stages` (stageList) too, and can get even more complex by iterating stages and their jobs, or even the jobs and their steps and apply your own logic in regard to the defined pipeline.

Overall expressions and templates within Azure Pipelines are very powerful, allowing you to enforce your own standards and checks, allowing compliance and security to be built in by default in to every new pipeline by sharing them.

### What kind of things can I do?

There are many use cases for manipulating the pipeline using templates, avoiding you having to keep repeating your code in every pipeline, or just to enforce compliance. Here is just a few scenarios to get you started.

* **Pushing notifications** to remote systems when a new stage or deployment job starts or ends, for auditing or updates to service tickets.
* **Avoid certain pipeline tasks being used** (like the script example discussed above), to ensure a consistent approach, or to help with security concerns.
* **Restrict deployments** to specific branches to enhance security.
* **Injecting steps** before or after steps defined in the pipeline run, maybe to get the environment in to a secure or useable state before a pipeline executes its steps.
* **Enforcing a branching strategy** by failing pipelines before they run if the correct branch isn't used. You could also check a pull requests source and target branches at runtime and fail if it's not a correct transition.
* **Checking with remote services** before deploying to ensure certain checks have been passed before continuing.

There are a lot more things you could do, but the main point is that these templates are abstracted from the pipeline. This has the benefit's of being repeatable and the consumer doesn't need to care about what, or how they are doing something, it should simply do it.

### Related resources

* [Azure Pipelines Expressions](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/expressions?view=azure-devops)
* [Azure Pipelines Templates](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops)
* [Azure Pipelines Schema Reference](https://docs.microsoft.com/en-us/azure/devops/pipelines/yaml-schema?view=azure-devops&tabs=schema%2Cparameter-schema)
* [Azure Pipelines Runtime Parameters](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/runtime-parameters?view=azure-devops&tabs=script)