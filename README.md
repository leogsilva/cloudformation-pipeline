# CloudFormation CI/CD - Infrastructure
The main goal of this project is to present some of the tools currently available to test and deliver IaC with quality in an automated pipeline.

This project consists in an environment that creates a pipeline using native AWS tools. This Pipeline is used to deliver Cloudformation templates ensuring reliability to deploy. 

The idea of this project is that you can use parts of it in your own pipeline and understand how interact with these tools.


![Alt text](img/img01.png?raw=true "Pipeline")

___

### AWS Services:

* CloudFormation
* Code Pipeline
* Code Build
* CodeCommit
* Lambda
* CloudWatch


The idea here is design a conceptual pipeline according to DevOps best practices.

It's a Conceptual pipeline for delivery of stacks written in cloudformation.


![Alt text](img/img02.png?raw=true "CodePipeline")





***


# Overview


## Continuous Integration

For CI (Continuous Integration) Stage, the pipeline executes these tools to verify Cloudformation templates:
*  **Cfn-lint** https://github.com/awslabs/cfn-python-lint
*  **Cfn_nag** https://github.com/stelligent/cfn_nag
*  **TaskCat** https://github.com/aws-quickstart/taskcat


### Source
Technically, *Continuous Integration*  process starts at Developer's notebook. 

A good practice is use IDE PlugIns to starts Code validation during development. At this case, we can use **Cfn-Lint** and **Cfn_Nag** PlugIns to Visual Studio Code. 
It helps get a very fast feedback regarding your code, validating sintax/semantic and governance/business logic.

A good way to get fast fail and fast feedback in your changes is using **Client Side Hooks**.
**Client Site Hooks** are validations made **before** send new code to remote repository.

For this project I created two of them - *pre-commit* and *pre-push*.

* **Pre-Commit** - Executes Cfn-lint and Cfn_Nag
* **Pre-Push** - Validates if pipeline is currently running. If yes, Push Command fails.

### Build
In Build Stage, there are four tasks running simultaneously:
* **Disable transition** - Block another pipeline execution if pipeline is already running.
* **Cfn-Lint** - Validate syntax and semantic
* **Cfn-Nag** - Validates governance and business logic
* **TaskCat** - Validates if template can be deployed in specific regions 


***


## Continuous Delivery

After that, CodePipeline starts CD (Continuous Delivery) step. 

### Test 
This stage creates a replica of the production environment, create the ChangeSet and apply into Replica Environment.

In parallel, creates a ChangeSet to be applied at Production Environment - **but do not apply yet**.


### Approve

At this stage, a PullRequest is **automatically** created. 

This is a very important Stage because here we ensure that other technical person will help the developer to analyze the quality of the code and the impact of that change in the environment. 

At this time, the approver has many important information to help him decide whether accept, or not, the new code.


The approver can access **Replica Environment** and **ChangeSet** to verify all the changes that will be applied into Production environment.


There's a lambda function that get the change status when PR is approved and triggers pipeline to go on.
Once PR is approved, pipeline merge at master branch . ***Developers only commit at staging branch and the pipeline takes care of merge***. 


### Promote
At this point, after all these Stages, we are high confidence that this change can be applied in our Production Environment.

ChangeSet is applied to Production environment and Replica environemnt is deleted. 
And, finally, we **enable transition** again and allow new execution of the pipeline. 


***


## Leading with more than one pipeline execution

A way that I found to do not permit execute more than one instance of the pipeline per time is **disabling transition** from ***Source*** to ***Build*** when pipeline is already running. This is made by a codebuild project.

When Pipeline start **Build Stage**, this CodeBuild project ***disable transition***. 

When Pipeline finish **Promote Stage**, another CodeBuildProject **enable transition** again and allow new execution of the pipeline. 

Without this trick, I could be problems with ChangeSet e ReplicaStacks.

Another way to handle this is just using the pre-push client-side-hook that validates if the pipeline is already running and does not allow the push to the repository.


***


## Pipeline Feedback using Slack

Two lambda functions send informations to slack channel. One to CodePipeline and another to CodeCommit.
* **Sentinel-Pipeline** - Send information about any change in your pipeline
* **Sentinel-CodeComit** - Send information when a PullRequest is Open/Closed/Merged.


---


# Getting Started


### Clone this repo and create the pipeline stack


```

git clone https://github.com/hgbueno/cloudformation-pipeline.git 

aws cloudformation create-stack --stack-name <MyStackName> --template-body file://templates/cfn-pipeline.yaml --capabilities CAPABILITY_IAM 

```

While Pipeline Stack is creating, configure some files to deploy your Production Stack.


### Configuration


1. Edit the variable TemplateToDeploy with the name of your cloudformation template. Remember that your CF template must be inside "templates" directory.


2. Slack Variables
   * SlackUser
   * SlackChannel
   * SlackURL
You can also pass these informations as command line parameters.


3. Edit TaskCat file ci/taskcat.yml and change variable "template_file" with the name of your cloudformation template.
By default TaskCat get templates at "/templates" directory


4. Copy ClientSite Hooks:
 ```   

cp -rpf hooks/* .git/hooks 

 ```   

5. Populate CodeCommit Repository. Remember that "MyRepoName" is the same as "MyStackName".

```
git clone https://git-codecommit.us-east-1.amazonaws.com/v1/repos/<MyRepoName> .

# (At this time, this repo is empty)

git add *

git commit -m "first commit" 

git push origin staging

```

Pay attention to always push to **staging** branch! **Only the Pipeline must merge to master!**

***Pipeline will start when you push to staging branch***


---

## Project structure
```

.
├── buildspecs
│   ├── buildspec-cfn-createpullrequest.yml
│   ├── buildspec-cfn-createreplica.yml
│   ├── buildspec-cfn-disabletransition.yml
│   ├── buildspec-cfn-enabletransition.yml
│   ├── buildspec-cfn-lint.yml
│   ├── buildspec-cfn_nag.yml
│   └── buildspec-taskcat.yml
├── ci
│   ├── debug-input.json
│   └── taskcat.yml
├── hooks
│   ├── pre-commit
│   └── pre-push
├── readme.md
└── templates
    ├── cfn-layer-base.yaml
    ├── cfn-layer_application.yaml
    └── cfn-pipeline.yaml


 ```   
    

