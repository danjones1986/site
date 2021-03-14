---
layout: post
title: Azure DevOps
subtitle: Advanced Pipeline Templates with Expressions
description: Azure Pipelines Extension Templates
date: 2021-03-14 09:00:00
author: Dan Jones
tags: azure-pipelines azure-devops templates advanced
---

In this post, we will be looking a more advanced use case for templates in Azure Pipelines, including, filtering, extending, or enforcing your own standards.

The most common use case for templates in Azure Pipelines is to abstract out bits of the pipeline and create reusabble segments, these segments could be in the same repository, or shared with multiple pipelines by hosting in a shared repository.

By the end, you will understand how to use expressions and to traverse the pipeline within a template to manipulate it how you see fit.

This guide assumes you have some familiarity with Azure Pipelines. We will start with a very basic example of a pipeline and refactor as we go.

```yml
trigger: 
- master
- develop
- release/*

pr:
- feature/*

pool:
  vmImage: ubuntu-16.04

variables:
  name: 'world'

stages:
- stage: Build
  jobs:
  - job: BuildHello
    steps:
    - script: echo 'Hello $(name) - Building'
- stage: Deploy
  dependsOn: Build
  jobs:
  - deployment: DeployHello
    environment: 'smarthotel-dev'
    strategy:
      runOnce:
        deploy:
          steps:
          - script: echo 'Hello $(name) - Deployed'
```

Our team have decided that the current approach we take to pipelines is too repetitive, with too many pipelines per repository and things are getting difficult to manage.

Its also been decided that we need to be able to enforce certain standards for security and compliance and you've been tasked to come up with a pattern that can be used to acheieve this.

A couple of simple scenarios for enforcement have been identified to begin with, such as;
* Only stages that are prefixed with the name "Deploy" can actually run deployments.
* Only certain branches are able to run deployments, including `release` and `master` branches.

Expressions and templates are the best way to achieve the above goals, extension templates can also be used to manipulate and verify your pipeline.

# Step 1 - Deploy only on master branch

Starting with the example above, we want to initially make sure we can only run pipeline on `master` branch

We can achieve this one of 2 ways, either with a condition or an expression.

A `condition` is evaluated at runtime so can be flexible where your pipeline is complex and you may have variables that changes state throughout.

```yml
...
- stage: Deploy
  dependsOn: Build
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/master'))
  jobs:
  ...
```

Within the condition segment you can include conditional expressions.
- `and` - Both passed arguments must be true `and(argument1, argument2)`
- `succeeded()` - Ensured previous stage, job or step succeeded before running this one.
- `eq` - First argument must equal the same as the second `eq(argument1, argument2)`
 
A conditional `expression` is compile time statement, which can be used for most use cases, but with an additional benefit that, if condition is true to skip a step, job or stage, it will not show in the Pipeline UI in the first place.

The syntax is different, but ultimately similar.

```yml
...
- ${{ if and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/master')) }}:
  - stage: Deploy
    dependsOn: Build
    jobs:
  ...
```

There is no `if else` syntax, instead you would write 2 opposing if statements.

```yml
steps:
  - ${{ if and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/master')) }}:
    - script: echo "I'm the master!"
  - ${{ if and(succeeded(), ne(variables['Build.SourceBranch'], 'refs/heads/master')) }}:
    - script: echo "I'm not the master..."
```

- `ne` - Not equal, opposite to `eq`.

Compile time expressions use the same syntax as parameters `${{  }}` and to do advances things with templates, you need to understand `if` and `each expressions`, which we will go through in more detail later.

Because we are using a compile time variable `Build.SourceBranch`, we've decided to take advantage of compile time expressions, the added benefit is we only see the stages in the pipeline we care about for that run.

This achieves the first scenario of using one pipeline for a single repo, since now except for master, any other run will only perform a build.

# Step 2 - Only allow deployment for Deploy* stages

This is a more complex scenario, we need to implement something like the following sudo code.

```javascript
if (!stage.name.startsWith('Deploy')) {
    foreach(var job in stage.jobs) {
        if (job.type == "deployment") {
            throw error 'Deployments can only occur in stages prefixed with Deploy';
        }
    }
}
```

To implement this requires the ability to traverse the pipeline and make our own decisions based on the make up of it, to do this we need to implement and extension template.

`parameters` within a template, can contain data types such as `stageList`, a reference of the standard data types can be found [here](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/runtime-parameters?view=azure-devops&tabs=script#parameter-data-types).

There are a few things to know before we start that are important to creating extension templates.

Data types such as `stageList`, `jobList` and `stepList` are arrays containing objects that can be accessed using the schema references on the [Azure Pipelines Reference](https://docs.microsoft.com/en-us/azure/devops/pipelines/yaml-schema?view=azure-devops&tabs=schema%2Cparameter-schema#stage) page.

For example the schema for `stages` is

```yml
stages:
- stage: string  # name of the stage (A-Z, a-z, 0-9, and underscore)
  displayName: string  # friendly name to display in the UI
  dependsOn: string | [ string ]
  condition: string
  variables: # several syntaxes, see specific section
  jobs: [ job | templateReference]
```

To loop through stages, you use the `each` compile time expression, which would look like
```yml
- ${{ each stageItem in parameters.stagesList }}:
```

`stageItem` now contains the stage, in the form of the schema, so to access the stage name, I could use `stageItem.stage`, to access the display name, it would be `stageItem.displayName`.

Finally you can also loop on stageItem itself, which will return a key, value pair. For example based on the above example, looping stageItem in the deploy stage would give the following;

| Key      | Value |
| ----------- | ----------- |
| stage      | Deploy       |
| dependsOn   | Build        |
| jobs      | jobs segment |

The following example will scan through the stages, and filter out any defined `dependsOn` segment.

```yml
steps:
  - ${{ each entry in stageItem }}:
    ${{ if ne(entry.key, 'dependsOn') }}:
      - ${{ entry.key }}: ${{ entry.value }}
```

Our template will be called `policy.yml` and take a parameter that contains out stages in the current pipeline, which changes our pipeline to the following.

```yml
trigger: 
- master
- develop
- release/*

pr:
- feature/*

pool:
  vmImage: ubuntu-16.04

variables:
  name: 'world'

extends:
  template: policy.yml
  parameters:
    stageList:
    - stage: Build
      jobs:
      - job: BuildHello
        steps:
        - script: echo 'Hello $(name) - Building'
      - ${{ if and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/
        - stage: Deploy
          dependsOn: Build
          jobs:
          - deployment: DeployHello
          environment: 'smarthotel-dev'
          strategy:
              runOnce:
              deploy:
                  steps:
                  - script: echo 'Hello $(name) - Deployed'
```





[Data Type References](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/runtime-parameters?view=azure-devops&tabs=script#parameter-data-types)

[Steps list](https://docs.microsoft.com/en-us/azure/devops/pipelines/yaml-schema?view=azure-devops&tabs=schema%2Cparameter-schema#steps)

[Pipeline expressions](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/expressions?view=azure-devops)