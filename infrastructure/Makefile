SHELL := /bin/bash

create:
	aws cloudformation create-stack \
	--stack-name $(StackName) \
	--template-body file://$(File) \
	--parameters \
	    ParameterKey=GitHubUser,ParameterValue=$(GitHubUser) \
	    ParameterKey=GitHubToken,ParameterValue=$(GitHubToken) \
	    ParameterKey=GitHubRepoName,ParameterValue=$(GitHubRepoName) \
	    ParameterKey=GitHubBranchName,ParameterValue=$(GitHubBranchName) \
	--capabilities CAPABILITY_IAM \
	--profile $(Profile)

update:
	aws cloudformation update-stack \
	--stack-name $(StackName) \
	--template-body file://$(File) \
	--parameters \
	    ParameterKey=GitHubUser,ParameterValue=$(GitHubUser) \
	    ParameterKey=GitHubToken,ParameterValue=$(GitHubToken) \
	    ParameterKey=GitHubRepoName,ParameterValue=$(GitHubRepoName) \
	    ParameterKey=GitHubBranchName,ParameterValue=$(GitHubBranchName) \
	--capabilities CAPABILITY_IAM \
	--profile $(Profile)

delete:
	aws cloudformation delete-stack \
	--stack-name $(StackName) \
	--profile $(Profile)