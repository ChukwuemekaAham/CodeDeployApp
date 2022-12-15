## Setting up a CI/CD pipeline by integrating Jenkins with AWS CodeBuild and AWS CodeDeploy

In this post, I explain how to use the [Jenkins](https://jenkins.io/) open-source automation server to deploy [AWS CodeBuild](https://aws.amazon.com/codebuild/) artifacts with [AWS CodeDeploy](https://aws.amazon.com/codedeploy/), creating a functioning CI/CD pipeline. When properly implemented, the CI/CD pipeline is triggered by code changes pushed to your GitHub repo, automatically fed into CodeBuild, then the output is deployed on CodeDeploy.

Solution overview
The functioning pipeline creates a fully managed build service that compiles your source code. It then produces code artifacts that can be used by CodeDeploy to deploy to your production environment automatically.

![img](solution.png)



The deployment workflow starts by placing the application code on the GitHub repository. To automate this scenario, I added source code management to the Jenkins project under the Source Code section. I chose the GitHub option, which by design clones a copy from the GitHub repo content in the Jenkins local workspace directory.

In the second step of my automation procedure, I enabled a trigger for the Jenkins server using an “Poll SCM” option. This option makes Jenkins check the configured repository for any new commits/code changes with a specified frequency. In this testing scenario, I configured the trigger to perform every two minutes. The automated Jenkins deployment process works as follows:

1) Jenkins checks for any new changes on GitHub every two minutes.

2) Change determination:
If Jenkins finds no changes, Jenkins exits the procedure.
If it does find changes, Jenkins clones all the files from the GitHub repository to the Jenkins server workspace directory.

3) The File Operation plugin deletes all the files cloned from GitHub. This keeps the Jenkins workspace directory clean.

4) The AWS CodeBuild plugin zips the files and sends them to a predefined Amazon S3 bucket location then initiates the CodeBuild project, which obtains the code from the S3 bucket. The project then creates the output artifact zip file, and stores that file again on the S3 bucket.
5) The HTTP Request plugin downloads the CodeBuild output artifacts from the S3 bucket.
I edited the S3 bucket policy to allow access from the Jenkins server IP address. See the following example policy:
{
  "Version": "2012-10-17",
  "Id": "S3PolicyId1",
  "Statement": [
    {
      "Sid": "IPAllow",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": "arn:aws:s3:::examplebucket/*",
      "Condition": {
         "IpAddress": {"aws:SourceIp": "x.x.x.x/x"},  <--- IP of the Jenkins server
      } 
    } 
  ]
}

JSON
This policy enables the HTTP request plugin to access the S3 bucket. This plugin doesn’t use the IAM instance profile or the AWS access keys (access key ID and secret access key).

6) The output artifact is a compressed ZIP file. The CodeDeploy plugin by design requires the files to be unzipped to zip them and send them over to the S3 bucket for the CodeDeploy deployment. For that, I used the File Operation plugin to perform the following:
Unzip the CodeBuild zipped artifact output in the Jenkins root workspace directory. At this point, the workspace directory should include the original zip file downloaded from the S3 bucket from Step 5 and the files extracted from this archive.
Delete the original .zip file, and leave only the source bundle contents for the deployment.

7) The CodeDeploy plugin selects and zips all workspace directory files. This plugin uses the CodeDeploy application name, deployment group name, and deployment configurations that you configured to initiate a new CodeDeploy deployment. The CodeDeploy plugin then uploads the newly zipped file according to the S3 bucket location provided to CodeDeploy as a source code for its new deployment operation.


## Walkthrough

In this post, I walk you through the following steps:

Creating resources to build the infrastructure, including the Jenkins server, CodeBuild project, and CodeDeploy application.
Accessing and unlocking the Jenkins server.
Creating a project and configuring the CodeDeploy Jenkins plugin.
Testing the whole CI/CD pipeline.


## Create the resources
In this section, I show you how to launch an AWS CloudFormation template, a tool that creates the following resources:

*Amazon S3 bucket*—Stores the GitHub repository files and the CodeBuild artifact application file that CodeDeploy uses.

*IAM S3 bucket policy*—Allows the Jenkins server access to the S3 bucket.

*JenkinsRole*—An IAM role and instance profile for the Amazon EC2 instance for use as a Jenkins server. This role allows Jenkins on the EC2 instance to access the S3 bucket to write files and access to create CodeDeploy deployments.

*CodeDeploy application* and *CodeDeploy deployment group*.

*CodeDeploy service role*—An IAM role to enable CodeDeploy to read the tags applied to the instances or the EC2 Auto Scaling group names associated with the instances.

*CodeDeployRole*—An IAM role and instance profile for the EC2 instances of CodeDeploy. This role has permissions to write files to the S3 bucket created by this template and to create deployments in CodeDeploy.

*CodeBuildRole*—An IAM role to be used by CodeBuild to access the S3 bucket and create the build projects.

*Jenkins server*—An EC2 instance running Jenkins.

*CodeBuild project*—This is configured with the S3 bucket and S3 
artifact.

*Auto Scaling group*—Contains EC2 instances running Apache and the CodeDeploy agent fronted by an Elastic Load Balancer.
Auto Scaling launch configurations—For use by the Auto Scaling group.

*Security groups*—For the Jenkins server, the load balancer, and the CodeDeploy EC2 instances.
 

 https://us-west-2.console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/create?templateURL=https://s3.us-west-2.amazonaws.com/cf-templates-1w3b7bfbw1d7d-us-west-2/2022-12-11T103442.690Z15w-CodeDeploy.cfn.yaml

S3 URL: https://s3.eu-central-1.amazonaws.com/cf-templates-1w3b7bfbw1d7d-eu-central-1/2022-12-12T141344.838Zj5l-CodeDeploy.cfn.json

 https://us-west-2.console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/quickcreate?templateURL=https%3A%2F%2Fs3.us-west-2.amazonaws.com%2Fcf-templates-1w3b7bfbw1d7d-us-west-2%2F2022-12-11T130854.743Zpt3-CodeDeploy.cfn.json&stackName=CICD-Jenkins-CodeBuild-CodeDeploy&param_KeyName=adminkey&param_InstanceCount=3&param_YourIPRange=0.0.0.0%2F0&param_CodedeployInstanceType=t2.medium&param_VpcId=vpc-5d8e8b25&param_PublicSubnet2=subnet-7e64f206&param_JenkinsInstanceType=t2.medium&param_CodeBuildProject=CodeBuild-S3&param_PublicSubnet1=subnet-b8cb78f2

 aws cloudformation create-stack --stack-name S2 --template-body example template --parameters ParameterKey=ImageId,ParameterValue=myLatestAMI


 Browse to the ELBDNSName http://cicd-jenkins-c-elb-432di5sslbhs-974005596.us-west-2.elb.amazonaws.com/ value from the Outputs tab, verifying that you can see the Sample page. You should see a congratulatory message.
Your Jenkins server should be ready to deploy.


Access and unlock your Jenkins server
In this section, I discuss how to access, unlock, and customize your Jenkins server.

Copy the JenkinsServerDNSName http://ec2-54-70-123-171.us-west-2.compute.amazonaws.com/ value from the Outputs tab of the CloudFormation stack, and paste it into your browser.
To unlock the Jenkins server, SSH to the server using the IP address and key pair, following the instructions from Unlocking Jenkins.
Use the root user to Cat the log file (/var/log/jenkins/jenkins.log) and copy the automatically generated alphanumeric password (between the two sets of asterisks). Then, use the password to unlock your Jenkins server, as shown in the following screenshots.


ami-06ce824c157700cd2 ubuntu22

ami-076309742d466ad69 AMI Linux 2