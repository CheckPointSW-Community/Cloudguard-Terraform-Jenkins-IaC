# Cloudguard-Terraform-Jenkins-IaC
Jenkins Pipeline for Check Point Terraform IaC
The following components will be covered:
Source Code Management Repository: Git and GitHub.
Terraform: HashiCorp Terraform has an integration with Cloud providers and Check Point using Providers plugins. We will be using the Check Point modules located on the Check Point CloudGuard IaaS GitHub repository or you can use the ones on my repository for this tutorial.
 https://github.com/chkp-dhouari/Cloudguard-Terraform-Jenkins-IaC.git


 https://github.com/CheckPointSW/CloudGuardIaaS/tree/master/terraform

Jenkins: CI/CD management server for the creation and management of the Security Infrastructure as continuous pipelines.
SourceGuard: Check Point static code analysis tool . More information at

CloudGuard IaaS: Check Point Multi Cloud Cloud Infrastructure Security.
The CloudGuard network security gateways will be configured as EC2 instances in an AWS autoscaling group in a VPC. The management server responsible for security policy management and configuration the security gateways will be configured as an EC2 instance as well in the same VPC. Terraform allow for the use of modules in order to embody the DRY software principle.

PreRequesites:
AWS account and AWS cli setup / https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html
Install Virtual Box - https://www.virtualbox.org/ for the Jenkins VM unless you are a cloud VM.
Install Git on your local machine - https://git-scm.com/
ATOM IDE editor and install the terraform plugin to manage Terraform code templates - https://atom.io/
Architecture:
No alt text provided for this image
>Let's get started by Terraforming CloudGuard as Code..
Lets get started and understand the structure of the Terraform modules defining the Check Point CloudGuard IaaS security gateways and management server configuration in code. Terraform modules have been created in line with the software DRY (Do Not Repeat Yourself) principle which can be set by security team and reused by App and Infrastructure teams without having to configure everything to scratch or having to become Terraform experts. Per Terraform documentation, Modules in Terraform are self-contained packages of Terraform configurations that are managed as a group. Modules are used to create reusable components in Terraform as well as for basic code organization. Simply put, no need to rewrite the whole code every time you want to provision something like a Firewall, server, function, load balancer etc..

Modules directory should have 3 files per Terraform best practices as follow: Note that all Terraform configuration files are named with the .tf file extension.
main.tf: configuration of all resource objects to be provisioned in the cloud provider to create the infrastructure.
output.tf: value of results of the calling module that can used in other modules using <module.module_name.output_name>
variables.tf: values of input variables used by modules using var.variable_name that will define your infrastructure
There are 3 Check Point CloudGuard modules and let's explore them:
amis: This module defines all the Check Point amis to used in Cloudguard security GWs and management server per region,
instance_type: This module allow to chose the type of amazon instance type (t2,c5, m5 etc..) and size (micro, large, xlarge..) to be used for the CloudGuard cloud security gateway.
autoscale: This module defines all the configuration blocks of all the objects to be provisioned such as the cloudguard VMs. the autoscaling group, security groups, launch policies, load balancer etc.. using following structure:
You can refer to the Terraform documentation for more information on resources, arguments and attributes. https://www.terraform.io/docs/index.html

// calling upon the amis module//

module "amis" {
	  source = "../amis"	

	  version_license = var.version_license
	}

//The resource type is autoscaling group in aws and the user given name in this terraform template is "asg"//
  
  resource "aws_autoscaling_group" "asg" {

// the actual of the autoscaling group when deployed in aws as asg_name and defined in the local definition//	

 name_prefix = local.asg_name


//defines which lauch configuration to use with the value format as <TYPE>.<NAME>.<ARGUMENT> and in this case the argument is the id of the launch config//	  

 launch_configuration= aws_launch_configuration.asg_launch_configuration.id

//min size of the autoscaling group//	 

 min_size = var.minimum_group_size

//max size of the autoscaling group//
	 
 max_size = var.maximum_group_size

//ELB associated with the autoscaling group. Note the splat article or * that iterates the list over many ELB names.//	
  
   load_balancers = aws_elb.proxy_elb.*.name 
   
....

}

Let us now have a look at the actual main.tf file for this lab to deploy our CloudGuard Security Hub. we first define the provider as part of the main.tf or as a separate provider.tf file. This will allow Terraform to download the aws plugin when initializing the terraform project root directory. It is important to define your aws region. AWS user id and key is required but should be configured as secrets, environment variable or with the aws configure command as Terraform will read the credentials file in the .aws directory:

provider "aws" {
	  region = var.region
	}

Below is the configuration or provisioning our security hub.You can see how simple and how easy to use the code becomes when using Terraform modules. All the variables values are coded in the variable.tf and terraform.tvars files.

Calling the autoscale module
module "autoscale" {
	  source = "../../modules/autoscale"
	
Environment
	  prefix = var.prefix
	  asg_name = var.asg_name
	

VPC Network Configuration
	  vpc_id = var.vpc_id
	  subnet_ids = var.subnet_ids
	
Gateway configuration
	  gateways_provision_address_type = var.gateways_provision_address_type
	  managementServer = var.managementServer
	  configurationTemplate = var.configurationTemplate
	

EC2 Instances Configuration
	  instances_name = var.instances_name
	  instance_type = var.instance_type
	  key_name = var.key_name
	

Auto Scaling Configuration
	  minimum_group_size = var.minimum_group_size
	  maximum_group_size = var.maximum_group_size
	  
	

CloudGuard Parameters Configuration
	  version_license = var.version_license
	  admin_shell = var.admin_shell
	  password_hash = var.password_hash
	  SICKey = var.SICKey
	  enable_instance_connect = var.enable_instance_connect
	  allow_upload_download = var.allow_upload_download
	  enable_cloudwatch = var.enable_cloudwatch
	  bootstrap_script = var.bootstrap_script
	

Outbound Load Balancer Configuration (optional)
	  proxy_elb_type = var.proxy_elb_type
	  proxy_elb_clients = var.proxy_elb_clients
	  proxy_elb_port = var.proxy_elb_port

}
Remote State
With Terraform, immutability is possible via a state file which allows it to track and compare the configuration with the actual of state of the resources provisioned in aws. This state file is located by default in the local root directory. However when using a CICD pipeline, it is important to configure a remote state backend so that the state file can be accessed from multiple environment. Terraform supports multiple backends and we will use S3 for this tutorial

Create and use a S3 bucket to store the state file:

$ aws s3 mb s3://your_bucket_name
Create backend.tf file with the following configuration:

terraform {
  backend "s3" {
	 bucket         = "your_bucket_name" 
	 key            = "remote.tfstate" <<-- name for your remote state file
	 region         = "us-east-1"
	 dynamodb_table = "terraform-state-lock" <<-- name of state lock DB
	  
}
 	}
In order to ensure the lock state to prevent multiple users doing configuration changes simultaneously, we will have to create a dynamoDB database with a LockID keyword..This only required when using S3 as backend.

No alt text provided for this image

We are now ready to configure Jenkins for the continuous Integration and Deployment:
Lets setup a Jenkins server for the configuration of the CICD pipeline and I will be installing it on an ubuntu Linux VM on my local machine using VBox and Vagrant...Jenkins requires to have Java installed as prerequisite. You can find the vagrant file on the tutorial GitHub repository.

##installing Java####

apt install default-jre            

apt install openjdk-8-jre-headless 

##installing Jenkins####

wget -q -O - https://jenkins-ci.org/debian/jenkins-ci.org.key | sudo apt-key add -


sudo echo deb http://pkg.jenkins-ci.org/debian binary/ > /etc/apt/sources.list.d/jenkins.list'


sudo apt-get update


sudo apt-get install jenkins

Note: In case you get a GPG key error when running apt-get update, please use the following cli with the last 8 digit of the public key that is printed out in the error message:

sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv <last 8 digits of the PUBLIC KEY>

Once Jenkins is installed, please verify that the Jenkins service is Active with the following command..Otherwise use the service jenkins start or restart command

$ service jenkins status

● jenkins.service - LSB: Start Jenkins at boot time

   Loaded: loaded (/etc/init.d/jenkins; generated)

   Active: active (exited) since Wed 2020-07-15 08:53:59 UTC; 1min 2s ago

     Docs: man:systemd-sysv-generator(8)

    Tasks: 0 (limit: 4915)

   CGroup: /system.slice/jenkins.service




Jul 15 08:53:47 minion-1 systemd[1]: Starting LSB: Start Jenkins at boot time...

Jul 15 08:53:48 minion-1 jenkins[14590]: Correct java version found
.time...

Please open your browser to the IP address of your VM at port 8080. Jenkins listens by default to port 8080 and it can be changed by replacing the value of <HTTP_PORT= > at /etc/default/jenkins. You can find your IP address using ifconfig. IF you are not using a VM then it would be localhost or 127.0.0.1. You will see the page below and please unlock Jenkins with the password below:

# cat /var/lib/jenkins/secrets/initialAdminPassword

<your_password>

No alt text provided for this image
No alt text provided for this image
Please install suggested plugin. You will be able to start using Jenkins and start creating devops pipelines on your way to be a DevSecOps master:

Please go to configure Jenkins and add the following AWS Plugins in order to be able to pass aws credentials to Terraform as we will be deploying the security infrastructure to

AWS SDK
AWS step credentials
Docker
Jenkins Pipeline:
We are now ready to create our Jenkins CICD pipeline and we will call it "security_as_code". in the jenkins page click on < new item> ,enter the name of the pipeline, select Pipeline and then click OK.

No alt text provided for this image


No alt text provided for this image
We will then define the pipeline details such as our GitHub repository as source control or the source of our versioned code. The code being the Jenkinsfile which is the definition of the Pipeline as code and the terraform templates for the CloudGuard cloud security gateways to be deployed in AWS.

In order to manage disc space used by Jenkins to select discard old builds and select number of builds based on your environment.

No alt text provided for this image
Scroll down to the Pipeline section and define your Source Code Management or Github in this case where my repository is located. copy and paste your repo address. If the repository is public then there is no need to enter any credentials. I am using only one branch or master. The Jenkinsfile defines the pipeline as code and then click Save.

No alt text provided for this image
Credentials:
It is critical to store any credentials such as usernames, password, API keys in an encrypted store. Jenkins provides that with the credentials configuration menu. Please configure your SouceGuard and AWS API keys as Jenkins credentials with the SourceGuard as secret text option and AWS as aws credentials in the Kind drop down menu:

No alt text provided for this image
Jenkinsfile or Pipeline as Code:
Your pipeline is ready and lets have a look at the Jenkinsfile to understand the various stages according the architecture picture of this CICD pipeline..Jenkinsfile is written in Groovy which is a weird spin-off of JavaScript that is also declarative. As explained above it is important to use declarative code.

Lets breakdown the pipeline with each stage..More stages can added notifications via email or slack..I am using human approval stages and will show more advanced stages in the Part 2 of this tutorial: The code starts by declaring that it is a declarative Jenkins pipeline and that we will not be specifying a particular agent for starters. We are also declaring the SourceGuard API keys as environment variables for SourceGuard to run.

pipeline {
	    agent any

// define the env variables for this pipeline and add the API ID and Key for the Sourceguard SAST tool

	     environment {
	           SG_CLIENT_ID = credentials("SG_CLIENT_ID")
	           SG_SECRET_KEY = credentials("SG_SECRET_KEY")
	           }

Stages configuration block will allow to configure each stage of this pipeline. We will start with Checking out all our code from the GitHub repository.."scm" stands for source code management.

   stages {
	

	     stage('checkout Terraform files to deploy infra') {
	      steps {
	        checkout scm
	       }
	

	     }

The second stage is to do static code analysis of all the Terraform code for credentials, vulnerabilities, malicious IPs, etc...using SourceGuard. If anything is found, SourceGuard will flag the scan result as BLOCK and the Jenkins pipeline will fail. However, I will allow the pipeline to continue as I will add an approval stage based on the scan result review:

    
      stage('Terraform Code SourceGuard SAST Scan') { 
          agent {
               docker { image 'dhouari/devsecops'
                         args '--entrypoint=' }
                       }
          steps { 
             script {      
                 try {
                     
                     sh 'sourceguard-cli --src .'
           
                   } catch (Exception e) {
    
                   echo "Request for Code Review Approval"  
                  }
               }
            }
         }
       
Note: I have added deleted AWS credentials to illustrate the importance of performing static code analysis. This is the result of the scan flagging the AWS credentials:

....

+ sourceguard-cli --src .
19-07-2020 07:53:24.931 SourceGuard Started

19-07-2020 07:53:28.676 Project name: Cloudguard-Terraform-Jenkins-IaC path: .
19-07-2020 07:53:28.676 Scan id: 56ee5e5a1b890ea04587e663a0e86640e1054894546f4a30c94d52cad8b856d8-ISWAsS

19-07-2020 07:53:36.443 Scanning ...

19-07-2020 07:53:56.035 Analyzing ...

19-07-2020 07:54:16.755 Action: BLOCK
Content Findings:
	- ID: 10000000-0000-0000-0000-000000000010
	  Name: "aws_access_key_id"
	  Description: "Possible AWS access key ID"
		- SHA: 452b416204d3854a842aa0d3fa32975961d80642dfee55fa09e2aabb2abb9da9 Path: IaC1@2/provider.tf
			- SHA: bde537f15e2943f9b057d60ae9e4bad5a8de7e9b14de30edec8291c609e0a1ed
			  Payload: AKIAVXXATFULLESP****
			  Lines: [4]
19-07-2020 07:54:16.755 Please see full analysis: https://portal.checkpoint.com/Dashboard/SourceGuard#/scan/56ee5e5a1b890ea04587e663a0e86640e1054894546f4a30c94d52cad8b856d8-ISWAsS
[Pipeline] echo
Request for Code Review Approval
[Pipeline] }
[Pipeline] // script
....

This next stage is important and will pause the pipeline and request for human admin approval based on the code scan result review.

         stage('Code approval request') {
	     
	           steps {
	             script {
	               def userInput = input(id: 'confirm', message: 'Code Approval Request?', 
       parameters: [ [$class: 'BooleanParameterDefinition', defaultValue: false, description: 'Approve to Proceed', name: 'approve'] ])
	              }
	            }
	          }


No alt text provided for this image


In this stage, we will do the terraform init and then the terraform plan in order to review what Terraform is planning to deploy or change.

    stage('init Terraform') {
	       agent {
	               docker { image 'dhouari/devsecops'
	                         args '--entrypoint=' }
	                       }
	        steps {
	            withAWS(credentials: 'awscreds', region: 'us-east-1'){
	               
	             sh "terraform init && terraform plan"
	            
	            }
	         }
	     }  


In this next stage, we will request approval from a human operator that the plan is valid and is OK to deploy the CloudGuard Security as Code configuration to the AWS cloud provider

   stage('Terraform plan approval request') {
	      steps {
	        script {
	          def userInput = input(id: 'confirm', message: 'Terraform Plan Approval Request?', 
        parameters: [ [$class: 'BooleanParameterDefinition', defaultValue: false,      description: 'Approve to Proceed', name: 'approve'] ])
	        }
	      }
	    }
	        
No alt text provided for this image


The final stage of this pipeline will do a terraform apply to deploy the CloudGuard cloud security to AWS. The auto-approve flag is setup since this has been approved in the previous stage as not to interrupt the deployment.

stage('Deploy the Terraform infra') {
	       agent {
	               docker { image 'dhouari/devsecops'
	                         args '--entrypoint=' }
	                       }
	        steps {
	            withAWS(credentials: 'awscreds', region: 'us-east-1'){
	               
	             sh "terraform apply --auto-approve"
	            
	          }
	       }
	    }


TIP: I am using my Check Point DevSecOps toolkit Docker container as stage agent so we won't have to install Terraform, SourceGuard and aws cli in the Jenkins workspace of the pipeline. It includes the SourceGuard cli, the Terraform cli and the AWS cli. The Check Point DevSecOps toolkit is located on Docker Hub under dhouari/devsecops

Note: It is good practice and actually important to clean up your Jenkins workspace once the pipeline is executed.

post { 
	  always { 
	    cleanWs()
	   }
	  
   }

}
We are almost done and it is time to trigger our pipeline by clicking on Build Now and please note that you can use webhooks on GitHub to trigger the pipeline automatically anytime any change to the code is committed to your repository.

SUCCESS
All the pipeline stages ran successfully and your Cloud Security code has been deployed and provisioned in your AWS region

No alt text provided for this image
TIP: //You can verify the provisioning using the Terraform show command and below is the Jenkins console logs for the pipeline

Thank you for reading and You are now ready to provision and manage all your cloud native Infrastructure Security using code!
Any changes, updates and deleting of cloud objects can be done from your code to trigger the pipeline to run again to update your cloud infrastructure in the same way a developer manages his applications

Started by user daboss
Obtained Jenkinsfile from git https://github.com/chkp-dhouari/Cloudguard-Terraform-Jenkins-IaC.git
Running in Durability level: MAX_SURVIVABILITY
[Pipeline] Start of Pipeline
[Pipeline] node
Running on Jenkins in /var/lib/jenkins/workspace/IaC1
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Declarative: Checkout SCM)
[Pipeline] checkout
No credentials specified
 > git rev-parse --is-inside-work-tree # timeout=10
Fetching changes from the remote Git repository
 > git config remote.origin.url https://github.com/chkp-dhouari/Cloudguard-Terraform-Jenkins-IaC.git # timeout=10
Fetching upstream changes from https://github.com/chkp-dhouari/Cloudguard-Terraform-Jenkins-IaC.git
 > git --version # timeout=10
 > git fetch --tags --progress -- https://github.com/chkp-dhouari/Cloudguard-Terraform-Jenkins-IaC.git +refs/heads/*:refs/remotes/origin/* # timeout=10
 > git rev-parse refs/remotes/origin/master^{commit} # timeout=10
 > git rev-parse refs/remotes/origin/origin/master^{commit} # timeout=10
Checking out Revision 9023b77f39703e80f4df55d66a2a33ab8d35578e (refs/remotes/origin/master)
 > git config core.sparsecheckout # timeout=10
 > git checkout -f 9023b77f39703e80f4df55d66a2a33ab8d35578e # timeout=10
Commit message: "Update Jenkinsfile"
 > git rev-list --no-walk 6999aa4f80fae58e68d6fbaf000eb483a652af43 # timeout=10
[Pipeline] }
[Pipeline] // stage
[Pipeline] withEnv
[Pipeline] {
[Pipeline] withCredentials
Masking supported pattern matches of $privatekey or $SG_SECRET_KEY or $SG_CLIENT_ID
[Pipeline] {
[Pipeline] stage
[Pipeline] { (checkout Terraform files to deploy infra)
[Pipeline] checkout
No credentials specified
 > git rev-parse --is-inside-work-tree # timeout=10
Fetching changes from the remote Git repository
 > git config remote.origin.url https://github.com/chkp-dhouari/Cloudguard-Terraform-Jenkins-IaC.git # timeout=10
Fetching upstream changes from https://github.com/chkp-dhouari/Cloudguard-Terraform-Jenkins-IaC.git
 > git --version # timeout=10
 > git fetch --tags --progress -- https://github.com/chkp-dhouari/Cloudguard-Terraform-Jenkins-IaC.git +refs/heads/*:refs/remotes/origin/* # timeout=10
 > git rev-parse refs/remotes/origin/master^{commit} # timeout=10
 > git rev-parse refs/remotes/origin/origin/master^{commit} # timeout=10
Checking out Revision 9023b77f39703e80f4df55d66a2a33ab8d35578e (refs/remotes/origin/master)
 > git config core.sparsecheckout # timeout=10
 > git checkout -f 9023b77f39703e80f4df55d66a2a33ab8d35578e # timeout=10
Commit message: "Update Jenkinsfile"
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Terraform Code SourceGuard SAST Scan)
[Pipeline] node
Running on Jenkins in /var/lib/jenkins/workspace/IaC1@2
[Pipeline] {
[Pipeline] checkout
No credentials specified
 > git rev-parse --is-inside-work-tree # timeout=10
Fetching changes from the remote Git repository
 > git config remote.origin.url https://github.com/chkp-dhouari/Cloudguard-Terraform-Jenkins-IaC.git # timeout=10
Fetching upstream changes from https://github.com/chkp-dhouari/Cloudguard-Terraform-Jenkins-IaC.git
 > git --version # timeout=10
 > git fetch --tags --progress -- https://github.com/chkp-dhouari/Cloudguard-Terraform-Jenkins-IaC.git +refs/heads/*:refs/remotes/origin/* # timeout=10
 > git rev-parse refs/remotes/origin/master^{commit} # timeout=10
 > git rev-parse refs/remotes/origin/origin/master^{commit} # timeout=10
Checking out Revision 9023b77f39703e80f4df55d66a2a33ab8d35578e (refs/remotes/origin/master)
 > git config core.sparsecheckout # timeout=10
 > git checkout -f 9023b77f39703e80f4df55d66a2a33ab8d35578e # timeout=10
Commit message: "Update Jenkinsfile"
[Pipeline] withEnv
[Pipeline] {
[Pipeline] isUnix
[Pipeline] sh
+ docker inspect -f . dhouari/devsecops
.
[Pipeline] withDockerContainer
Jenkins does not seem to be running inside a container
$ docker run -t -d -u 0:0 --entrypoint= -w /var/lib/jenkins/workspace/IaC1@2 -v /var/lib/jenkins/workspace/IaC1@2:/var/lib/jenkins/workspace/IaC1@2:rw,z -v /var/lib/jenkins/workspace/IaC1@2@tmp:/var/lib/jenkins/workspace/IaC1@2@tmp:rw,z -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** dhouari/devsecops cat
$ docker top 83ccff4f33fff554032efa4dd51c85b508c4649aeb79baa3623091a6c423d50d -eo pid,comm
[Pipeline] {
[Pipeline] script
[Pipeline] {
[Pipeline] sh
+ sourceguard-cli --src .
19-07-2020 07:53:24.931 SourceGuard Started
19-07-2020 07:53:28.676 Project name: Cloudguard-Terraform-Jenkins-IaC path: .
19-07-2020 07:53:28.676 Scan id: 56ee5e5a1b890ea04587e663a0e86640e1054894546f4a30c94d52cad8b856d8-ISWAsS
19-07-2020 07:53:36.443 Scanning ...
19-07-2020 07:53:56.035 Analyzing ...
19-07-2020 07:54:16.755 Action: BLOCK
Content Findings:
	- ID: 10000000-0000-0000-0000-000000000010
	  Name: "aws_access_key_id"
	  Description: "Possible AWS access key ID"
		- SHA: 452b416204d3854a842aa0d3fa32975961d80642dfee55fa09e2aabb2abb9da9 Path: IaC1@2/provider.tf
			- SHA: bde537f15e2943f9b057d60ae9e4bad5a8de7e9b14de30edec8291c609e0a1ed
			  Payload: AKIAVXXATFULLESP****
			  Lines: [4]
19-07-2020 07:54:16.755 Please see full analysis: https://portal.checkpoint.com/Dashboard/SourceGuard#/scan/56ee5e5a1b890ea04587e663a0e86640e1054894546f4a30c94d52cad8b856d8-ISWAsS
[Pipeline] echo
Request for Code Review Approval
[Pipeline] }
[Pipeline] // script
[Pipeline] }
$ docker stop --time=1 83ccff4f33fff554032efa4dd51c85b508c4649aeb79baa3623091a6c423d50d
$ docker rm -f 83ccff4f33fff554032efa4dd51c85b508c4649aeb79baa3623091a6c423d50d
[Pipeline] // withDockerContainer
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // node
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Code approval request)
[Pipeline] script
[Pipeline] {
[Pipeline] input
Input requested
Approved by daboss
[Pipeline] }
[Pipeline] // script
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (init Terraform)
[Pipeline] node
Running on Jenkins in /var/lib/jenkins/workspace/IaC1@2
[Pipeline] {
[Pipeline] checkout
No credentials specified
 > git rev-parse --is-inside-work-tree # timeout=10
Fetching changes from the remote Git repository
 > git config remote.origin.url https://github.com/chkp-dhouari/Cloudguard-Terraform-Jenkins-IaC.git # timeout=10
Fetching upstream changes from https://github.com/chkp-dhouari/Cloudguard-Terraform-Jenkins-IaC.git
 > git --version # timeout=10
 > git fetch --tags --progress -- https://github.com/chkp-dhouari/Cloudguard-Terraform-Jenkins-IaC.git +refs/heads/*:refs/remotes/origin/* # timeout=10
 > git rev-parse refs/remotes/origin/master^{commit} # timeout=10
 > git rev-parse refs/remotes/origin/origin/master^{commit} # timeout=10
Checking out Revision 9023b77f39703e80f4df55d66a2a33ab8d35578e (refs/remotes/origin/master)
 > git config core.sparsecheckout # timeout=10
 > git checkout -f 9023b77f39703e80f4df55d66a2a33ab8d35578e # timeout=10
Commit message: "Update Jenkinsfile"
[Pipeline] withEnv
[Pipeline] {
[Pipeline] isUnix
[Pipeline] sh
+ docker inspect -f . dhouari/devsecops
.
[Pipeline] withDockerContainer
Jenkins does not seem to be running inside a container
$ docker run -t -d -u 0:0 --entrypoint= -w /var/lib/jenkins/workspace/IaC1@2 -v /var/lib/jenkins/workspace/IaC1@2:/var/lib/jenkins/workspace/IaC1@2:rw,z -v /var/lib/jenkins/workspace/IaC1@2@tmp:/var/lib/jenkins/workspace/IaC1@2@tmp:rw,z -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** dhouari/devsecops cat
$ docker top 6e2bb525697cf7937076b6d5b3dddec31bcd10934781ddcc6343ad57648e2d96 -eo pid,comm
[Pipeline] {
[Pipeline] withAWS
Constructing AWS CredentialsSetting AWS region us-east-1 
[Pipeline] {
[Pipeline] sh
+ terraform init --upgrade
[0m[1mUpgrading modules...[0m
- autoscale in modules/autoscale
- autoscale.amis in modules/amis
- autoscale.validate_instance_type in modules/instance_type

[0m[1mInitializing the backend...[0m

[0m[1mInitializing provider plugins...[0m
- Checking for available provider plugins...
- Downloading plugin for provider "random" (hashicorp/random) 2.3.0...
- Downloading plugin for provider "http" (hashicorp/http) 1.2.0...
- Downloading plugin for provider "aws" (hashicorp/aws) 2.70.0...

The following providers do not have any version constraints in configuration,
so the latest version was installed.

To prevent automatic upgrades to new major versions that may contain breaking
changes, it is recommended to add version = "..." constraints to the
corresponding provider blocks in configuration, with the constraint strings
suggested below.

* provider.http: version = "~> 1.2"
* provider.random: version = "~> 2.3"

[0m[1m[32mTerraform has been successfully initialized![0m[32m[0m
[0m[32m
You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.[0m
+ terraform plan
Acquiring state lock. This may take a few moments...
[0m[1mRefreshing Terraform state in-memory prior to plan...[0m
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.
[0m
[0m[1mmodule.autoscale.module.amis.data.http.amis_json_http: Refreshing state...[0m
[0m[1mmodule.autoscale.data.aws_iam_policy_document.assume_role_policy_document: Refreshing state...[0m
[0m[1mmodule.autoscale.data.aws_iam_policy_document.policy_document: Refreshing state...[0m
[0m[1mmodule.autoscale.module.amis.data.aws_region.current: Refreshing state...[0m

------------------------------------------------------------------------

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  [32m+[0m create
 [36m<=[0m read (data resources)
[0m
Terraform will perform the following actions:

[1m  # data.aws_instances.asg_instances[0m will be read during apply
  # (config refers to values not yet known)[0m[0m
[0m[36m <=[0m [0mdata "aws_instances" "asg_instances"  {
      [32m+[0m [0m[1m[0mid[0m[0m            = (known after apply)
      [32m+[0m [0m[1m[0mids[0m[0m           = (known after apply)
      [32m+[0m [0m[1m[0minstance_tags[0m[0m = (known after apply)
      [32m+[0m [0m[1m[0mprivate_ips[0m[0m   = (known after apply)
      [32m+[0m [0m[1m[0mpublic_ips[0m[0m    = (known after apply)

      [32m+[0m [0mfilter {
          [32m+[0m [0m[1m[0mname[0m[0m   = "tag:aws:autoscaling:groupName"
          [32m+[0m [0m[1m[0mvalues[0m[0m = [
              [32m+[0m [0m(known after apply),
            ]
        }
    }

[1m  # module.autoscale.aws_autoscaling_group.asg[0m will be created[0m[0m
[0m[32m  +[0m [0mresource "aws_autoscaling_group" "asg" {
      [32m+[0m [0m[1m[0marn[0m[0m                       = (known after apply)
      [32m+[0m [0m[1m[0mavailability_zones[0m[0m        = (known after apply)
      [32m+[0m [0m[1m[0mdefault_cooldown[0m[0m          = (known after apply)
      [32m+[0m [0m[1m[0mdesired_capacity[0m[0m          = (known after apply)
      [32m+[0m [0m[1m[0mforce_delete[0m[0m              = false
      [32m+[0m [0m[1m[0mhealth_check_grace_period[0m[0m = 0
      [32m+[0m [0m[1m[0mhealth_check_type[0m[0m         = (known after apply)
      [32m+[0m [0m[1m[0mid[0m[0m                        = (known after apply)
      [32m+[0m [0m[1m[0mlaunch_configuration[0m[0m      = (known after apply)
      [32m+[0m [0m[1m[0mload_balancers[0m[0m            = (known after apply)
      [32m+[0m [0m[1m[0mmax_size[0m[0m                  = 10
      [32m+[0m [0m[1m[0mmetrics_granularity[0m[0m       = "1Minute"
      [32m+[0m [0m[1m[0mmin_size[0m[0m                  = 2
      [32m+[0m [0m[1m[0mname[0m[0m                      = (known after apply)
      [32m+[0m [0m[1m[0mname_prefix[0m[0m               = "TEST-autoscaling_group"
      [32m+[0m [0m[1m[0mprotect_from_scale_in[0m[0m     = false
      [32m+[0m [0m[1m[0mservice_linked_role_arn[0m[0m   = (known after apply)
      [32m+[0m [0m[1m[0mtarget_group_arns[0m[0m         = (known after apply)
      [32m+[0m [0m[1m[0mvpc_zone_identifier[0m[0m       = [
          [32m+[0m [0m"subnet-0a961057ebe4dfff5",
        ]
      [32m+[0m [0m[1m[0mwait_for_capacity_timeout[0m[0m = "10m"

      [32m+[0m [0mtag {
          [32m+[0m [0m[1m[0mkey[0m[0m                 = "Name"
          [32m+[0m [0m[1m[0mpropagate_at_launch[0m[0m = true
          [32m+[0m [0m[1m[0mvalue[0m[0m               = "TEST-asg_gateway"
        }
      [32m+[0m [0mtag {
          [32m+[0m [0m[1m[0mkey[0m[0m                 = "x-chkp-tags"
          [32m+[0m [0m[1m[0mpropagate_at_launch[0m[0m = true
          [32m+[0m [0m[1m[0mvalue[0m[0m               = "management=mgmt_env1:template=tmpl_env1:ip-address=private"
        }
    }

[1m  # module.autoscale.aws_autoscaling_policy.scale_down_policy[0m will be created[0m[0m
[0m[32m  +[0m [0mresource "aws_autoscaling_policy" "scale_down_policy" {
      [32m+[0m [0m[1m[0madjustment_type[0m[0m         = "ChangeInCapacity"
      [32m+[0m [0m[1m[0marn[0m[0m                     = (known after apply)
      [32m+[0m [0m[1m[0mautoscaling_group_name[0m[0m  = (known after apply)
      [32m+[0m [0m[1m[0mcooldown[0m[0m                = 300
      [32m+[0m [0m[1m[0mid[0m[0m                      = (known after apply)
      [32m+[0m [0m[1m[0mmetric_aggregation_type[0m[0m = (known after apply)
      [32m+[0m [0m[1m[0mname[0m[0m                    = (known after apply)
      [32m+[0m [0m[1m[0mpolicy_type[0m[0m             = "SimpleScaling"
      [32m+[0m [0m[1m[0mscaling_adjustment[0m[0m      = -1
    }

[1m  # module.autoscale.aws_autoscaling_policy.scale_up_policy[0m will be created[0m[0m
[0m[32m  +[0m [0mresource "aws_autoscaling_policy" "scale_up_policy" {
      [32m+[0m [0m[1m[0madjustment_type[0m[0m         = "ChangeInCapacity"
      [32m+[0m [0m[1m[0marn[0m[0m                     = (known after apply)
      [32m+[0m [0m[1m[0mautoscaling_group_name[0m[0m  = (known after apply)
      [32m+[0m [0m[1m[0mcooldown[0m[0m                = 300
      [32m+[0m [0m[1m[0mid[0m[0m                      = (known after apply)
      [32m+[0m [0m[1m[0mmetric_aggregation_type[0m[0m = (known after apply)
      [32m+[0m [0m[1m[0mname[0m[0m                    = (known after apply)
      [32m+[0m [0m[1m[0mpolicy_type[0m[0m             = "SimpleScaling"
      [32m+[0m [0m[1m[0mscaling_adjustment[0m[0m      = 1
    }

[1m  # module.autoscale.aws_cloudwatch_metric_alarm.cpu_alarm_high[0m will be created[0m[0m
[0m[32m  +[0m [0mresource "aws_cloudwatch_metric_alarm" "cpu_alarm_high" {
      [32m+[0m [0m[1m[0mactions_enabled[0m[0m                       = true
      [32m+[0m [0m[1m[0malarm_actions[0m[0m                         = (known after apply)
      [32m+[0m [0m[1m[0malarm_description[0m[0m                     = "Scale-up if CPU > 80% for 10 minutes"
      [32m+[0m [0m[1m[0malarm_name[0m[0m                            = (known after apply)
      [32m+[0m [0m[1m[0marn[0m[0m                                   = (known after apply)
      [32m+[0m [0m[1m[0mcomparison_operator[0m[0m                   = "GreaterThanThreshold"
      [32m+[0m [0m[1m[0mdimensions[0m[0m                            = (known after apply)
      [32m+[0m [0m[1m[0mevaluate_low_sample_count_percentiles[0m[0m = (known after apply)
      [32m+[0m [0m[1m[0mevaluation_periods[0m[0m                    = 2
      [32m+[0m [0m[1m[0mid[0m[0m                                    = (known after apply)
      [32m+[0m [0m[1m[0mmetric_name[0m[0m                           = "CPUUtilization"
      [32m+[0m [0m[1m[0mnamespace[0m[0m                             = "AWS/EC2"
      [32m+[0m [0m[1m[0mperiod[0m[0m                                = 300
      [32m+[0m [0m[1m[0mstatistic[0m[0m                             = "Average"
      [32m+[0m [0m[1m[0mthreshold[0m[0m                             = 80
      [32m+[0m [0m[1m[0mtreat_missing_data[0m[0m                    = "missing"
    }

[1m  # module.autoscale.aws_cloudwatch_metric_alarm.cpu_alarm_low[0m will be created[0m[0m
[0m[32m  +[0m [0mresource "aws_cloudwatch_metric_alarm" "cpu_alarm_low" {
      [32m+[0m [0m[1m[0mactions_enabled[0m[0m                       = true
      [32m+[0m [0m[1m[0malarm_actions[0m[0m                         = (known after apply)
      [32m+[0m [0m[1m[0malarm_description[0m[0m                     = "Scale-down if CPU < 60% for 10 minutes"
      [32m+[0m [0m[1m[0malarm_name[0m[0m                            = (known after apply)
      [32m+[0m [0m[1m[0marn[0m[0m                                   = (known after apply)
      [32m+[0m [0m[1m[0mcomparison_operator[0m[0m                   = "LessThanThreshold"
      [32m+[0m [0m[1m[0mdimensions[0m[0m                            = (known after apply)
      [32m+[0m [0m[1m[0mevaluate_low_sample_count_percentiles[0m[0m = (known after apply)
      [32m+[0m [0m[1m[0mevaluation_periods[0m[0m                    = 2
      [32m+[0m [0m[1m[0mid[0m[0m                                    = (known after apply)
      [32m+[0m [0m[1m[0mmetric_name[0m[0m                           = "CPUUtilization"
      [32m+[0m [0m[1m[0mnamespace[0m[0m                             = "AWS/EC2"
      [32m+[0m [0m[1m[0mperiod[0m[0m                                = 300
      [32m+[0m [0m[1m[0mstatistic[0m[0m                             = "Average"
      [32m+[0m [0m[1m[0mthreshold[0m[0m                             = 60
      [32m+[0m [0m[1m[0mtreat_missing_data[0m[0m                    = "missing"
    }

[1m  # module.autoscale.aws_elb.proxy_elb[0][0m will be created[0m[0m
[0m[32m  +[0m [0mresource "aws_elb" "proxy_elb" {
      [32m+[0m [0m[1m[0marn[0m[0m                         = (known after apply)
      [32m+[0m [0m[1m[0mavailability_zones[0m[0m          = (known after apply)
      [32m+[0m [0m[1m[0mconnection_draining[0m[0m         = false
      [32m+[0m [0m[1m[0mconnection_draining_timeout[0m[0m = 300
      [32m+[0m [0m[1m[0mcross_zone_load_balancing[0m[0m   = true
      [32m+[0m [0m[1m[0mdns_name[0m[0m                    = (known after apply)
      [32m+[0m [0m[1m[0mid[0m[0m                          = (known after apply)
      [32m+[0m [0m[1m[0midle_timeout[0m[0m                = 60
      [32m+[0m [0m[1m[0minstances[0m[0m                   = (known after apply)
      [32m+[0m [0m[1m[0minternal[0m[0m                    = false
      [32m+[0m [0m[1m[0mname[0m[0m                        = (known after apply)
      [32m+[0m [0m[1m[0msecurity_groups[0m[0m             = (known after apply)
      [32m+[0m [0m[1m[0msource_security_group[0m[0m       = (known after apply)
      [32m+[0m [0m[1m[0msource_security_group_id[0m[0m    = (known after apply)
      [32m+[0m [0m[1m[0msubnets[0m[0m                     = [
          [32m+[0m [0m"subnet-0a961057ebe4dfff5",
        ]
      [32m+[0m [0m[1m[0mzone_id[0m[0m                     = (known after apply)

      [32m+[0m [0mhealth_check {
          [32m+[0m [0m[1m[0mhealthy_threshold[0m[0m   = 3
          [32m+[0m [0m[1m[0minterval[0m[0m            = 30
          [32m+[0m [0m[1m[0mtarget[0m[0m              = "TCP:8080"
          [32m+[0m [0m[1m[0mtimeout[0m[0m             = 5
          [32m+[0m [0m[1m[0munhealthy_threshold[0m[0m = 5
        }

      [32m+[0m [0mlistener {
          [32m+[0m [0m[1m[0minstance_port[0m[0m     = 8080
          [32m+[0m [0m[1m[0minstance_protocol[0m[0m = "TCP"
          [32m+[0m [0m[1m[0mlb_port[0m[0m           = 8080
          [32m+[0m [0m[1m[0mlb_protocol[0m[0m       = "TCP"
        }
    }

[1m  # module.autoscale.aws_launch_configuration.asg_launch_configuration[0m will be created[0m[0m
[0m[32m  +[0m [0mresource "aws_launch_configuration" "asg_launch_configuration" {
      [32m+[0m [0m[1m[0marn[0m[0m                         = (known after apply)
      [32m+[0m [0m[1m[0massociate_public_ip_address[0m[0m = true
      [32m+[0m [0m[1m[0mebs_optimized[0m[0m               = (known after apply)
      [32m+[0m [0m[1m[0menable_monitoring[0m[0m           = true
      [32m+[0m [0m[1m[0mid[0m[0m                          = (known after apply)
      [32m+[0m [0m[1m[0mimage_id[0m[0m                    = "ami-0a84ab03e292eba9f"
      [32m+[0m [0m[1m[0minstance_type[0m[0m               = "c5.xlarge"
      [32m+[0m [0m[1m[0mkey_name[0m[0m                    = (known after apply)
      [32m+[0m [0m[1m[0mname[0m[0m                        = (known after apply)
      [32m+[0m [0m[1m[0mname_prefix[0m[0m                 = "TEST-autoscaling_group"
      [32m+[0m [0m[1m[0msecurity_groups[0m[0m             = (known after apply)
      [32m+[0m [0m[1m[0muser_data[0m[0m                   = "b551fc926dcf699bccaa986f34a7bc60832c7625"

      [32m+[0m [0mebs_block_device {
          [32m+[0m [0m[1m[0mdelete_on_termination[0m[0m = (known after apply)
          [32m+[0m [0m[1m[0mdevice_name[0m[0m           = (known after apply)
          [32m+[0m [0m[1m[0mencrypted[0m[0m             = (known after apply)
          [32m+[0m [0m[1m[0miops[0m[0m                  = (known after apply)
          [32m+[0m [0m[1m[0mno_device[0m[0m             = (known after apply)
          [32m+[0m [0m[1m[0msnapshot_id[0m[0m           = (known after apply)
          [32m+[0m [0m[1m[0mvolume_size[0m[0m           = (known after apply)
          [32m+[0m [0m[1m[0mvolume_type[0m[0m           = (known after apply)
        }

      [32m+[0m [0mroot_block_device {
          [32m+[0m [0m[1m[0mdelete_on_termination[0m[0m = (known after apply)
          [32m+[0m [0m[1m[0mencrypted[0m[0m             = (known after apply)
          [32m+[0m [0m[1m[0miops[0m[0m                  = (known after apply)
          [32m+[0m [0m[1m[0mvolume_size[0m[0m           = (known after apply)
          [32m+[0m [0m[1m[0mvolume_type[0m[0m           = (known after apply)
        }
    }

[1m  # module.autoscale.aws_load_balancer_policy.proxy_elb_policy[0][0m will be created[0m[0m
[0m[32m  +[0m [0mresource "aws_load_balancer_policy" "proxy_elb_policy" {
      [32m+[0m [0m[1m[0mid[0m[0m                 = (known after apply)
      [32m+[0m [0m[1m[0mload_balancer_name[0m[0m = (known after apply)
      [32m+[0m [0m[1m[0mpolicy_name[0m[0m        = "EnableProxyProtocol"
      [32m+[0m [0m[1m[0mpolicy_type_name[0m[0m   = "ProxyProtocolPolicyType"

      [32m+[0m [0mpolicy_attribute {
          [32m+[0m [0m[1m[0mname[0m[0m  = "ProxyProtocol"
          [32m+[0m [0m[1m[0mvalue[0m[0m = "true"
        }
    }

[1m  # module.autoscale.aws_security_group.elb_security_group[0][0m will be created[0m[0m
[0m[32m  +[0m [0mresource "aws_security_group" "elb_security_group" {
      [32m+[0m [0m[1m[0marn[0m[0m                    = (known after apply)
      [32m+[0m [0m[1m[0mdescription[0m[0m            = "ELB security group"
      [32m+[0m [0m[1m[0megress[0m[0m                 = [
          [32m+[0m [0m{
              [32m+[0m [0mcidr_blocks      = [
                  [32m+[0m [0m"0.0.0.0/0",
                ]
              [32m+[0m [0mdescription      = ""
              [32m+[0m [0mfrom_port        = 0
              [32m+[0m [0mipv6_cidr_blocks = []
              [32m+[0m [0mprefix_list_ids  = []
              [32m+[0m [0mprotocol         = "-1"
              [32m+[0m [0msecurity_groups  = []
              [32m+[0m [0mself             = false
              [32m+[0m [0mto_port          = 0
            },
        ]
      [32m+[0m [0m[1m[0mid[0m[0m                     = (known after apply)
      [32m+[0m [0m[1m[0mingress[0m[0m                = [
          [32m+[0m [0m{
              [32m+[0m [0mcidr_blocks      = [
                  [32m+[0m [0m"0.0.0.0/0",
                ]
              [32m+[0m [0mdescription      = ""
              [32m+[0m [0mfrom_port        = 8080
              [32m+[0m [0mipv6_cidr_blocks = []
              [32m+[0m [0mprefix_list_ids  = []
              [32m+[0m [0mprotocol         = "tcp"
              [32m+[0m [0msecurity_groups  = []
              [32m+[0m [0mself             = false
              [32m+[0m [0mto_port          = 8080
            },
        ]
      [32m+[0m [0m[1m[0mname[0m[0m                   = (known after apply)
      [32m+[0m [0m[1m[0mowner_id[0m[0m               = (known after apply)
      [32m+[0m [0m[1m[0mrevoke_rules_on_delete[0m[0m = false
      [32m+[0m [0m[1m[0mvpc_id[0m[0m                 = "vpc-0f176e052bb5fa932"
    }

[1m  # module.autoscale.aws_security_group.permissive_sg[0m will be created[0m[0m
[0m[32m  +[0m [0mresource "aws_security_group" "permissive_sg" {
      [32m+[0m [0m[1m[0marn[0m[0m                    = (known after apply)
      [32m+[0m [0m[1m[0mdescription[0m[0m            = "Permissive security group"
      [32m+[0m [0m[1m[0megress[0m[0m                 = [
          [32m+[0m [0m{
              [32m+[0m [0mcidr_blocks      = [
                  [32m+[0m [0m"0.0.0.0/0",
                ]
              [32m+[0m [0mdescription      = ""
              [32m+[0m [0mfrom_port        = 0
              [32m+[0m [0mipv6_cidr_blocks = []
              [32m+[0m [0mprefix_list_ids  = []
              [32m+[0m [0mprotocol         = "-1"
              [32m+[0m [0msecurity_groups  = []
              [32m+[0m [0mself             = false
              [32m+[0m [0mto_port          = 0
            },
        ]
      [32m+[0m [0m[1m[0mid[0m[0m                     = (known after apply)
      [32m+[0m [0m[1m[0mingress[0m[0m                = [
          [32m+[0m [0m{
              [32m+[0m [0mcidr_blocks      = [
                  [32m+[0m [0m"0.0.0.0/0",
                ]
              [32m+[0m [0mdescription      = ""
              [32m+[0m [0mfrom_port        = 0
              [32m+[0m [0mipv6_cidr_blocks = []
              [32m+[0m [0mprefix_list_ids  = []
              [32m+[0m [0mprotocol         = "-1"
              [32m+[0m [0msecurity_groups  = []
              [32m+[0m [0mself             = false
              [32m+[0m [0mto_port          = 0
            },
        ]
      [32m+[0m [0m[1m[0mname[0m[0m                   = (known after apply)
      [32m+[0m [0m[1m[0mname_prefix[0m[0m            = "TEST-autoscaling_group_PermissiveSecurityGroup"
      [32m+[0m [0m[1m[0mowner_id[0m[0m               = (known after apply)
      [32m+[0m [0m[1m[0mrevoke_rules_on_delete[0m[0m = false
      [32m+[0m [0m[1m[0mtags[0m[0m                   = {
          [32m+[0m [0m"Name" = "TEST-autoscaling_group_PermissiveSecurityGroup"
        }
      [32m+[0m [0m[1m[0mvpc_id[0m[0m                 = "vpc-0f176e052bb5fa932"
    }

[1m  # module.autoscale.random_id.proxy_elb_uuid[0m will be created[0m[0m
[0m[32m  +[0m [0mresource "random_id" "proxy_elb_uuid" {
      [32m+[0m [0m[1m[0mb64[0m[0m         = (known after apply)
      [32m+[0m [0m[1m[0mb64_std[0m[0m     = (known after apply)
      [32m+[0m [0m[1m[0mb64_url[0m[0m     = (known after apply)
      [32m+[0m [0m[1m[0mbyte_length[0m[0m = 5
      [32m+[0m [0m[1m[0mdec[0m[0m         = (known after apply)
      [32m+[0m [0m[1m[0mhex[0m[0m         = (known after apply)
      [32m+[0m [0m[1m[0mid[0m[0m          = (known after apply)
    }

[0m[1mPlan:[0m 11 to add, 0 to change, 0 to destroy.[0m

------------------------------------------------------------------------

Note: You didn't specify an "-out" parameter to save this plan, so Terraform
can't guarantee that exactly these actions will be performed if
"terraform apply" is subsequently run.

Releasing state lock. This may take a few moments...
[Pipeline] }
[Pipeline] // withAWS
[Pipeline] }
$ docker stop --time=1 6e2bb525697cf7937076b6d5b3dddec31bcd10934781ddcc6343ad57648e2d96
$ docker rm -f 6e2bb525697cf7937076b6d5b3dddec31bcd10934781ddcc6343ad57648e2d96
[Pipeline] // withDockerContainer
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // node
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Terraform plan approval request)
[Pipeline] script
[Pipeline] {
[Pipeline] input
Input requested
Approved by daboss
[Pipeline] }
[Pipeline] // script
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage

[Pipeline] { (Deploy the Terraform infra)
[Pipeline] node
Running on Jenkins in /var/lib/jenkins/workspace/IaC1@2
[Pipeline] {
[Pipeline] checkout
No credentials specified
 > git rev-parse --is-inside-work-tree # timeout=10
Fetching changes from the remote Git repository
 > git config remote.origin.url https://github.com/chkp-dhouari/Cloudguard-Terraform-Jenkins-IaC.git # timeout=10
Fetching upstream changes from https://github.com/chkp-dhouari/Cloudguard-Terraform-Jenkins-IaC.git
 > git --version # timeout=10
 > git fetch --tags --progress -- https://github.com/chkp-dhouari/Cloudguard-Terraform-Jenkins-IaC.git +refs/heads/*:refs/remotes/origin/* # timeout=10

 > git rev-parse refs/remotes/origin/master^{commit} # timeout=10
 > git rev-parse refs/remotes/origin/origin/master^{commit} # timeout=10
Checking out Revision 05571342996e30d4e10545bd9629e22bbffb6b0c (refs/remotes/origin/master)
 > git config core.sparsecheckout # timeout=10
 > git checkout -f 05571342996e30d4e10545bd9629e22bbffb6b0c # timeout=10
Commit message: "Update Jenkinsfile"
 > git rev-list --no-walk 6999aa4f80fae58e68d6fbaf000eb483a652af43 # timeout=10
[Pipeline] withEnv
[Pipeline] {
[Pipeline] isUnix
[Pipeline] sh
+ docker inspect -f . dhouari/devsecops
.
[Pipeline] withDockerContainer
Jenkins does not seem to be running inside a container
$ docker run -t -d -u 0:0 --entrypoint= -w /var/lib/jenkins/workspace/IaC1@2 -v /var/lib/jenkins/workspace/IaC1@2:/var/lib/jenkins/workspace/IaC1@2:rw,z -v /var/lib/jenkins/workspace/IaC1@2@tmp:/var/lib/jenkins/workspace/IaC1@2@tmp:rw,z -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** dhouari/devsecops cat

$ docker top 8e187f3378331eb0a156acf452e80a269cfbb6fdb5b29842f1927af15eb918ae -eo pid,comm

[Pipeline] {
[Pipeline] withAWS
Constructing AWS CredentialsSetting AWS region us-east-1 
[Pipeline] {
[Pipeline] sh
+ terraform apply --auto-approve

Acquiring state lock. This may take a few moments...

[0m[1mmodule.autoscale.module.amis.data.http.amis_json_http: Refreshing state...[0m

[0m[1mmodule.autoscale.module.amis.data.aws_region.current: Refreshing state...[0m
[0m[1mmodule.autoscale.data.aws_iam_policy_document.policy_document: Refreshing state...[0m
[0m[1mmodule.autoscale.data.aws_iam_policy_document.assume_role_policy_document: Refreshing state...[0m

[0m[1mmodule.autoscale.random_id.proxy_elb_uuid: Creating...[0m[0m
[0m[1mmodule.autoscale.random_id.proxy_elb_uuid: Creation complete after 0s [id=wTcAf4w][0m[0m

[0m[1mmodule.autoscale.aws_security_group.elb_security_group[0]: Creating...[0m[0m
[0m[1mmodule.autoscale.aws_security_group.permissive_sg: Creating...[0m[0m

[0m[1mmodule.autoscale.aws_security_group.elb_security_group[0]: Still creating... [10s elapsed][0m[0m
[0m[1mmodule.autoscale.aws_security_group.permissive_sg: Still creating... [10s elapsed][0m[0m

[0m[1mmodule.autoscale.aws_security_group.elb_security_group[0]: Still creating... [20s elapsed][0m[0m
[0m[1mmodule.autoscale.aws_security_group.permissive_sg: Still creating... [20s elapsed][0m[0m

[0m[1mmodule.autoscale.aws_security_group.elb_security_group[0]: Creation complete after 23s [id=sg-0463b23e5966e998e][0m[0m
[0m[1mmodule.autoscale.aws_elb.proxy_elb[0]: Creating...[0m[0m

[0m[1mmodule.autoscale.aws_security_group.permissive_sg: Creation complete after 28s [id=sg-08f886d01ab6e533e][0m[0m
[0m[1mmodule.autoscale.aws_launch_configuration.asg_launch_configuration: Creating...[0m[0m

[0m[1mmodule.autoscale.aws_elb.proxy_elb[0]: Still creating... [10s elapsed][0m[0m

[0m[1mmodule.autoscale.aws_launch_configuration.asg_launch_configuration: Creation complete after 5s [id=TEST-autoscaling_group20200719075849768500000003][0m[0m

[0m[1mmodule.autoscale.aws_elb.proxy_elb[0]: Creation complete after 16s [id=TEST-proxy-elb-c137007f8c][0m[0m
[0m[1mmodule.autoscale.aws_load_balancer_policy.proxy_elb_policy[0]: Creating...[0m[0m
[0m[1mmodule.autoscale.aws_autoscaling_group.asg: Creating...[0m[0m
[0m[1mmodule.autoscale.aws_load_balancer_policy.proxy_elb_policy[0]: Creation complete after 2s [id=TEST-proxy-elb-c137007f8c:EnableProxyProtocol][0m[0m

[0m[1mmodule.autoscale.aws_autoscaling_group.asg: Still creating... [10s elapsed][0m[0m

[0m[1mmodule.autoscale.aws_autoscaling_group.asg: Still creating... [20s elapsed][0m[0m

[0m[1mmodule.autoscale.aws_autoscaling_group.asg: Still creating... [30s elapsed][0m[0m

[0m[1mmodule.autoscale.aws_autoscaling_group.asg: Still creating... [40s elapsed][0m[0m

[0m[1mmodule.autoscale.aws_autoscaling_group.asg: Creation complete after 47s [id=TEST-autoscaling_group20200719075859046000000004][0m[0m
[0m[1mdata.aws_instances.asg_instances: Refreshing state...[0m
[0m[1mmodule.autoscale.aws_autoscaling_policy.scale_down_policy: Creating...[0m[0m
[0m[1mmodule.autoscale.aws_autoscaling_policy.scale_up_policy: Creating...[0m[0m

[0m[1mmodule.autoscale.aws_autoscaling_policy.scale_down_policy: Creation complete after 2s [id=TEST-autoscaling_group20200719075859046000000004_scale_down][0m[0m
[0m[1mmodule.autoscale.aws_cloudwatch_metric_alarm.cpu_alarm_low: Creating...[0m[0m
[0m[1mmodule.autoscale.aws_autoscaling_policy.scale_up_policy: Creation complete after 2s [id=TEST-autoscaling_group20200719075859046000000004_scale_up][0m[0m
[0m[1mmodule.autoscale.aws_cloudwatch_metric_alarm.cpu_alarm_high: Creating...[0m[0m

[0m[1mmodule.autoscale.aws_cloudwatch_metric_alarm.cpu_alarm_low: Creation complete after 2s [id=TEST-autoscaling_group20200719075859046000000004_alarm_low][0m[0m
[0m[1mmodule.autoscale.aws_cloudwatch_metric_alarm.cpu_alarm_high: Creation complete after 2s [id=TEST-autoscaling_group20200719075859046000000004_alarm_high][0m[0m

[0m[1m[32m
Apply complete! Resources: 11 added, 0 changed, 0 destroyed.[0m
Releasing state lock. This may take a few moments...

[0m[1m[32m
Outputs:

autoscale-autoscaling_group_arn = arn:aws:autoscaling:us-east-1:392332258562:autoScalingGroup:9815d991-a5d4-47f2-a036-cff83c3d6fe3:autoScalingGroupName/TEST-autoscaling_group20200719075859046000000004
autoscale-autoscaling_group_availability_zones = [
  "us-east-1d",
]
autoscale-autoscaling_group_desired_capacity = 2
autoscale-autoscaling_group_id = TEST-autoscaling_group20200719075859046000000004
autoscale-autoscaling_group_load_balancers = [
  "TEST-proxy-elb-c137007f8c",
]
autoscale-autoscaling_group_max_size = 10
autoscale-autoscaling_group_min_size = 2
autoscale-autoscaling_group_name = TEST-autoscaling_group20200719075859046000000004
autoscale-autoscaling_group_subnets = [
  "subnet-0a961057ebe4dfff5",
]
autoscale-autoscaling_group_target_group_arns = []
autoscale-iam_role_name = []
autoscale-launch_configuration_id = TEST-autoscaling_group20200719075849768500000003
autoscale-security_group = sg-08f886d01ab6e533e
autoscale_instances = {
  "filter" = [
    {
      "name" = "tag:aws:autoscaling:groupName"
      "values" = [
        "TEST-autoscaling_group20200719075859046000000004",
      ]
    },
  ]
  "id" = "terraform-20200719075946658300000005"
  "ids" = [
    "i-018e3cfb1334234f0",
    "i-04ff28ef5a7584b79",
  ]
  "private_ips" = [
    "172.31.6.149",
    "172.31.11.27",
  ]
  "public_ips" = [
    "3.236.43.222",
    "3.226.238.118",
  ]
}
autoscale_output = {
  "Deployment" = "Finalizing instances configuration may take up to 20 minutes after deployment is finished."
  "autoscale-autoscaling_group_arn" = "arn:aws:autoscaling:us-east-1:392332258562:autoScalingGroup:9815d991-a5d4-47f2-a036-cff83c3d6fe3:autoScalingGroupName/TEST-autoscaling_group20200719075859046000000004"
  "autoscale-autoscaling_group_availability_zones" = [
    "us-east-1d",
  ]
  "autoscale-autoscaling_group_desired_capacity" = 2
  "autoscale-autoscaling_group_id" = "TEST-autoscaling_group20200719075859046000000004"
  "autoscale-autoscaling_group_load_balancers" = [
    "TEST-proxy-elb-c137007f8c",
  ]
  "autoscale-autoscaling_group_max_size" = 10
  "autoscale-autoscaling_group_min_size" = 2
  "autoscale-autoscaling_group_name" = "TEST-autoscaling_group20200719075859046000000004"
  "autoscale-autoscaling_group_subnets" = [
    "subnet-0a961057ebe4dfff5",
  ]
  "autoscale-autoscaling_group_target_group_arns" = []
  "autoscale-iam_role_name" = []
  "autoscale-launch_configuration_id" = "TEST-autoscaling_group20200719075849768500000003"
  "autoscale-security_group" = "sg-08f886d01ab6e533e"
}[0m
[Pipeline] }
[Pipeline] // withAWS
[Pipeline] }
$ docker stop --time=1 8e187f3378331eb0a156acf452e80a269cfbb6fdb5b29842f1927af15eb918ae

$ docker rm -f 8e187f3378331eb0a156acf452e80a269cfbb6fdb5b29842f1927af15eb918ae
[Pipeline] // withDockerContainer
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // node
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // withCredentials
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
Finished: SUCCESS
