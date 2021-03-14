---
layout: post
title: Azure Pipelines
subtitle: Manipulating Pipelines with Templates and Expressions
description: How to manipulate an Azure Pipeline, intercept calls, enforce  standards, replace functionality, using templates and compile time expressions.
date: 2021-03-14 18:00:00
author: Dan Jones
tags: azure-pipelines azure-devops templates advanced
---

Beyond creating standard Azure DevOps Pipeline definitions, we can actually use templates to manipulate a pipelines definition. We can make scripts before or after tasks, or add stages based on certain condition, or choose to reject a pipeline if certain definitions may or may not exist.

By passing the stages, jobs or steps to a template, we can within the template read the definition, and manipulate it how we see fit.

When passing definitions to a template, you must pass them as a specific data type, you can find a list in the [Pipelines runtime parameters](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/runtime-parameters?view=azure-devops&tabs=script#parameter-data-types) documentation.

Most relevant to the context of manipulating our pipeline definitions with templates include the following.

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

We will create a template (`build-policy.yml`), which takes pipeline steps as its parameter.

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

What is we wanted to run a step before or after the steps passed in to the template? We would do this by looping through the steps list and render each step individually, which will allow us to write our own tasks before or after.

Going back to out `build-policy.yml` example, our example would now look like the below.

{% raw %}
```yml
parameters:
- name: stepList
  type: stepList
  default: []

steps:
- script: echo "Begin running pipeline steps"
- ${{ each step in paramters.stepList }}:
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

In the above example you will notice `- ${{ each pair in step }}:`, which loops through the top level definition of the step, which is a little confusing at first, but in reality, it's pretty simple, which I will try clarify with an example.

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
    - script: echo "${{ pair.key }} - ${{ pair.key }}"
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

In the above examples, we use the `script` task, which depending on the OS will execute in `bash` or `powershell`, which isn't great for reuseability, so what if we want to stop stop our pipeline from using it altogether?

Our first step is to edit our definition and if `- script` is detected, throw an error at compile time to stop the pipeline even starting if it detects it.

{% raw %}
```yml
parameters:
- name: stepList
  type: stepList
  default: []

steps:
- bash: echo "Begin running pipeline steps"
- ${{ each step in paramters.stepList }}:
  - ${{ each pair in step }}:
    - ${{ if eq(pair.key, 'script') }}:
        'script tasks are not allowed!': error
    - ${{ if ne(pair.key, 'script') }}:
        ${{ pair.key }}: ${{ pair.value }}  
- bash: echo "Finished running pipeline steps"
```
{% endraw %}

The aren't many options for exceptions in Azure Pipelines, but by providing a message, followed by `: error`, an exception will be thrown at compile time, which will output you message on screen.

Lastly what if you wanted to enforce this through out the pipeline and ensure it can't be circumvented? We do this by changing how we use the template in the pipeline and use the `extends` syntax.

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

Using extends templates in the pipeline forces the stages, jobs or steps to run in the context of the template.

Extension templates can be stored in another repository and shared with other pipelines, which is great for company policy templates, you can even use Azure DevOps environments approvals and checks section to enforce that a deployment must implement a given template or it won't be able to execute.

If you moved our templates file in to a repository called `automation`, we could then use it by using a repository resource similar to the below example.

```yml
resources:
  repositories:
  - repository: automation  # identifier
    type: git 
    name: fabrikim/automation  # repository name (format depends on `type`)
    ref: refs/heads/master  

extends:
  template: build-policy.yml@automation
  parameters:
    stepList:
    - script: echo "Hello World 1"
    - script: echo "Hello World 2"
```

When consuming extension templates from a shared repository, you need to be aware of the following;
* Local templates must have `@self` on the end when being imported in to the pipeline.
* Local template imports now become relative to `$(System.DefaultWorkingDirectory)`, instead of the pipeline directory.
* If you have other templates you wish to use inside the extension template, from the shared repository, they will need to be included relative to the extension templates folder.

Overall expressions and templates within Azure Pipelines are very powerful, allowing you to enforce your own standards and checks, allowing compliance and security to be built in by default in to every new pipeline by sharing them.