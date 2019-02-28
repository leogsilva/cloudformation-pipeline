# Pipeline to Delivery Cloudformation Templates

![Alt text](img.png?raw=true "Pipeline")


### Introduction


This project consists in an environment that creates a pipeline using native AWS tools. This Pipeline is used to deliver Cloudformation templates, ensuring reliability to deploy.

The idea here is design a conceptual pipeline according to DevOps best practices. It's a Conceptual pipeline for delivery of stacks written in cloudformation


At CI (Continuous Integration) step, the pipeline executes these tools to verify Cloudformation templates:
- Cfn-lint
- Cfn_nag
- TaskCat

After that, CodePipeline starts CD (Continuous Delivery) step. The "Test" step creates a replica of the production environment.

In parallel, creates a ChangeSet to be applied at Production Environment. 

After these steps, a PullRequest is automatically created. 

At this time, the approver has various information available to decide whether to accept or not the PullRequest.

The approver can access the Replica environment and the ChangeSet to verify all the changes that will be applied at Production environment.

There's a lambda function that get the change status when PR is approved and allow pipeline to go on.
Once the PR is approved, the pipeline merge at branch master. Developers only commit at branch staging and the pipeline merges. 

After that, ChangeSet is applied to Production environment and Replica environemnt is deleted. 

Two lambda functions send informations to slack channel. One to CodePipeline and another to CodeCommit.


### AWS Services:
- CodeCommit
- Code Pipeline
- Code Build
- Lambda
- CloudWatch
- CloudFormation


## Getting Started

These instructions will get you a copy of the project up and running on your local machine for development and testing purposes. See deployment for notes on how to deploy the project on a live system.

Clone this repo and create a pipeline stack
```
aws cloudformation create-stack --stack-name <MyStackName> --template-body file://templates/cfn-pipeline.yaml --capabilities CAPABILITY_IAM 
```

After that, populate the CodeCommit Repository:
```
git clone https://git-codecommit.us-east-1.amazonaws.com/v1/repos/<MyRepoName> .
<MyRepoName> = <MyStackName> 

cp -rpf /files/* .
cp -rpf hooks/* .git/hooks 

git add *
git commit -m "first commit" 
git push origin master
git checkout -b staging master
git commit -m "first commit at staging" 
git push origin staging   
```

At this point CodePipeline starts and deploy Production and Replica Stacks.


### Project structure

![Alt text](img02.png?raw=true "Pipeline")
    
    
### Configuration
templates/cfn-pipeline.yaml

1) Edit the variable TemplateToDeploy with the name of your cloudformation template. Remember that your CF template must be inside "templates" directory.
2) Slack Variables
   - SlackUser
   - SlackChannel
   - SlackURL

You can also pass these informations as command line parameters.


## Authors

* **Henrique Bueno** - [GitHub](https://github.com/hgbueno)



