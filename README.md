# Continuous Delivery Pipeline for Serverless.com with AWS CodePipeline and CloudFormation

While the serverless.com framework is a great tool to develop and deploy applications to AWS, its configuration file `serverless.yml` can become quite bloated fast. 

This repository showcases a reference architecture for a fully automated serverless project including surrounding infrastructure like databases, while seperating concerns like disposable and non-disposable infrastructure with different lifecycles.

## The problem

When used with AWS, the `serverless.yml` provides a nice way to add additional resources, like databases, [in the `resources` section](https://serverless.com/framework/docs/providers/aws/guide/resources/).

One problem is that currently `serverless.yml` does not allow to reference additionally created resources everywhere in the YAML file as stated here. This could lead to hacks like hardcoding references.

Another problem one might experience is that already existing / legacy resources have to be referenced in the `serverless.yml`

## To the rescue: CodePipeline, CloudFormation and `${cf:}` 

### CodePipeline

 - Includes 3 stages
 - Uses CodeBuild for serverless deploy
 
### `${cf:}` Syntax

With the `${cf:}` syntax, resources created in other CloudFormation stacks can be referenced in the `serverless.yml`, and not restricted to a particular part or section.

To remove unnecessary API calls, we first assign the references to a custom variable in the `serverless.yml`:

```yaml
custom:
  userPoolArn: ${cf:app-userpool.UserPoolArn}
```
This assigns the Output `UserPoolArn` of the CloudFormation stack `app-userpool` to the `serverless.yml` variable `userPoolArn`.

And in the next step this variable could for example be used in a another location of `serverless.yml` - here is an example:
```yaml
functions:
  get:
    handler: ...
    events:
    - http:
      authorizer:
        arn: ${self:custom.userPoolArn}
```
### CloudFormation

The CloudFormation template for the stack `app-userpool` which creates the Cognito Userpool would look like this:

```yaml
AWSTemplateFormatVersion: '2010-09-09'

Resources:
  UserPool:
    Type: AWS::Cognito::UserPool
    Properties:
      UserPoolName: someuserpool

Outputs:
  UserPoolArn:
    Value: !GetAtt UserPool.Arn
```
All Outputs defined in the Output section can be consumed by `serverless.yml` with the `${cf:}` syntax.

### The missing link: CodePipeline and CodeDeploy
So how would we combine this? The missing link is a Continuous Delivery Pipeline, in AWS specifically the service CodePipeline which first creates and/or updates the CloudFormation stack(s), and second calls `serverless deploy` to deploy the service defined in `serverless.yml`.

A sample combination of CodePipeline and CodeBuild could look like this:

```yaml
Resources:

  CodePipeline:
    Type: AWS::CodePipeline::Pipeline
    Properties:
      RoleArn: !GetAtt CodePipelineRole.Arn
      Stages:
      - Name: Source
        Actions:
        ...
      - Name: InfrastructureDeploy
        Actions:
        - Name: DeployAppUserPool
          InputArtifacts:
          - Name: SourceArtifact
          ActionTypeId:
            Category: Deploy
            Owner: AWS
            Provider: CloudFormation
            Version: 1
          Configuration:
            ActionMode: CREATE_UPDATE
            StackName: app-userpool
            TemplatePath: SourceArtifact::infrastructure/app-userpool.yml
            RoleArn: !GetAtt CloudFormationInfrastructureRole.Arn
          RunOrder: 1
        - Name: ServerlessDeploy
          InputArtifacts:
          - Name: SourceArtifact
          ActionTypeId:
            Category: Build
            Owner: AWS
            Version: 1
            Provider: CodeBuild
          OutputArtifacts:
          - Name: DeployArtifact
          Configuration:
            ProjectName: !Ref ServerlessDeploy
          RunOrder: 1

  ...

  ServerlessDeploy:
    Type: AWS::CodeBuild::Project
    Properties:
      Name: !Sub "serverless-deployment-${AWS::StackName}"
      ServiceRole: !Ref ServerlessDeployRole
      Artifacts:
        Type: CODEPIPELINE
      Environment:
        ComputeType: BUILD_GENERAL1_SMALL
        Image: aws/codebuild/eb-nodejs-6.10.0-amazonlinux-64:4.0.0
        Type: LINUX_CONTAINER
      Source:
        Type: CODEPIPELINE
        BuildSpec: codebuild/buildspec_deploy.yml
```
First a CodePipeline is defined with a stage called `InfrastructureDeploy` which consists of 2 actions. For simplicity the `Source` stage has been omitted here.
The first Action `DeployAppUserPool` is of the native CodePipeline type `CloudFormation` and takes care of deploying the cloudformation stack `app-userpool` with the template `SourceArtifact::infrastructure/app-userpool.yml`. The first part (`SourceArtifact`) is the input artifact name within the CodePipeline definition and the second part (`infrastructure/app-userpool.yml`) is the path to the CloudFormation template within the artifact.

The full source code can be found in [infrastructure/pipeline.yml]()

## Summary

Advantages of this solution

<!-- ## Shims for legacy infrastructure -->




