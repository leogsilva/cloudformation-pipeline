# Pipeline to Delivery Cloudformation Templates
This project consists in an environment that creates a pipeline using native AWS tools. This Pipeline is used to deliver Cloudformation templates, ensuring reliability to deploy.


![Alt text](img/img01.png?raw=true "Pipeline")


![Alt text](img/img02.png?raw=true "CodePipeline")


The idea here is design a conceptual pipeline according to DevOps best practices. It's a Conceptual pipeline for delivery of stacks written in cloudformation.


***

# Overview


## Continuous Integration

For CI (Continuous Integration) step, the pipeline executes these tools to verify Cloudformation templates:
*  **Cfn-lint** [https://github.com/awslabs/cfn-python-lint]
*  **Cfn_nag** [https://github.com/stelligent/cfn_nag]
*  **TaskCat** [https://github.com/aws-quickstart/taskcat]

A good way to get fast fail and fast feedback in your changes is using **Client Side Hooks**.

**Client Site Hooks** are validations made **before** send new code to remote repository.


For this project I created two of them - *pre-commit* and *pre-push*.

* **Pre-Commit** executes Cfn-lint and Cfn_Nag
* **Pre-Push** validates if pipeline is currently running. If yes, Push Command fails.



## Continuous Delivery

After that, CodePipeline starts CD (Continuous Delivery) step. 

### Test 
The "Test" step creates a replica of the production environment, create the ChangeSet and apply into Replica Environment.

In parallel, creates a ChangeSet to be applied at Production Environment - **but do not apply yet**.


### Approve

After these steps, a PullRequest is **automatically** created. 


At this time, the approver has many important information to help him decide whether accept, or not, the new code.


The approver can access **Replica Environment** and **ChangeSet** to verify all the changes that will be applied into Production environment.


There's a lambda function that get the change status when PR is approved and triggers pipeline to go on.
Once PR is approved, pipeline merge at master branch . ***Developers only commit at staging branch and the pipeline takes care of merge***. 


### Promote

ChangeSet is applied to Production environment and Replica environemnt is deleted. 


## Leading with More than One Pipeline execution

A way that I found to do not permit execute more than one instance of the pipeline per time is disabling transition from Source to Build when pipeline is already running. This is made by a codebuild project.

When Pipeline start Build session, this CodeBuild project disable transition. When Pipeline finish Promote session, another CodeBuildProject enable transition. 


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



### Project structure
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
    
---

### AWS Services:

* CodeCommit
* Code Pipeline
* Code Build
* Lambda
* CloudWatch
* CloudFormation

