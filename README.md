# CloudFormation StackSet Demo

- [CloudFormation StackSet Demo](#cloudformation-stackset-demo)
  - [Deploy AWSCloudFormationStackSetAdministrationRole](#deploy-awscloudformationstacksetadministrationrole)
  - [Deploy AWSCloudFormationStackSetExecutionRole](#deploy-awscloudformationstacksetexecutionrole)
  - [Deploy StackSet](#deploy-stackset)
  - [Add Stack Instances to the StackSet to deploy our actual resources](#add-stack-instances-to-the-stackset-to-deploy-our-actual-resources)
  - [Clean-up](#clean-up)


This demos creating S3 buckets in two different AWS regions using an AWS StackSet within a single AWS account, with the typical two roles used for CloudFormation StackSets with slef-managed permissions deployed _with_ CloudFormation. Resources chosen to be low cost- here 2 * S3 bucket.

First we need to create the two roles needed for self managed permissions for CloudFormation:

- AWSCloudFormationStackSetAdministrationRole
- AWSCloudFormationStackSetExecutionRole

We can use the example roles discussed at https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-prereqs-self-managed.html for this - downloading the relevant files to a suitable location and then

## Deploy AWSCloudFormationStackSetAdministrationRole

```bash
aws cloudformation create-stack \
    --stack-name AWSCloudFormationStackSetAdministrationRole \
    --template-body file://AWSCloudFormationStackSetAdministrationRole.yml \
    --capabilities CAPABILITY_NAMED_IAM \
    --tags Key=Environment,Value=test Key=Project,Value=CloudFormationDemo
```

## Deploy AWSCloudFormationStackSetExecutionRole

**N.B. this is an admin permissions role and more permissive than may be desired in a production environment**

In a production environment this would typically be in other accounts

Export environment variable to avoid exposing our AWS account ID in source code:

```bash
export AWS_ACCOUNT_ID="account-id"
```

```bash
aws cloudformation create-stack \
    --stack-name AWSCloudFormationStackSetExecutionRole \
    --template-body file://AWSCloudFormationStackSetExecutionRole.yml \
    --parameters ParameterKey=AdministratorAccountId,ParameterValue="$AWS_ACCOUNT_ID" \
    --capabilities CAPABILITY_NAMED_IAM \
    --tags Key=Environment,Value=test Key=Project,Value=CloudFormationDemo
```

## Deploy StackSet

```bash
aws cloudformation create-stack-set \
  --stack-set-name "S3StackSet" \
  --template-body file://s3-stack.yaml \
  --execution-role-name "AWSCloudFormationStackSetExecutionRole"
```

## Add Stack Instances to the StackSet to deploy our actual resources

```bash
aws cloudformation create-stack-instances \
  --stack-set-name "S3StackSet" \
  --accounts "[\"$AWS_ACCOUNT_ID\"]" \
  --regions "[\"eu-west-1\", \"eu-west-2\"]"
  ```

  ## Clean-up

  Firstly we will update our stackset to enable deletion of resources so that are deleted with the respective stack instances:

  ```bash
aws cloudformation update-stack-set \
  --stack-set-name "S3StackSet" \
  --template-body file://s3-stack-teardown.yaml \
  --execution-role-name "AWSCloudFormationStackSetExecutionRole"
```

We wait for this to complete. We can then delete our stack instances. As per [AWS documentation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stackinstances-delete.html)_Because --retain-stacks is a required parameter of delete-stack-instances, if you don't want to retain (save) stacks, add --no-retain-stacks. In this walkthrough, we add the --no-retain-stacks parameter, because we aren't retaining any stacks._ 

```bash
aws cloudformation delete-stack-instances \
  --stack-set-name "S3StackSet" \
  --accounts "[\"$AWS_ACCOUNT_ID\"]" \
  --regions "[\"eu-west-1\", \"eu-west-2\"]" \
  --operation-preferences FailureToleranceCount=0,MaxConcurrentCount=1 \
  --no-retain-stacks
  ```

We can then delete the StackSet:

```bash
aws cloudformation delete-stack-set \
  --stack-set-name "S3StackSet"
```

Lastly we delete the two Stacks we used to deploy the self-managed permissions roles for CloudFormation:

```bash
aws cloudformation delete-stack \
    --stack-name AWSCloudFormationStackSetAdministrationRole
```  

```bash
aws cloudformation delete-stack \
    --stack-name AWSCloudFormationStackSetExecutionRole
  ```

We have now returned to the state we were in at the start of our exercise (we ignore the automatically created `cf-templates-...` S3 bucket!)