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

>Let's get started by Terraforming CloudGuard as Code..
Lets get started and understand the structure of the Terraform modules defining the Check Point CloudGuard IaaS security gateways and management server configuration in code. Terraform modules have been created in line with the software DRY (Do Not Repeat Yourself) principle which can be set by security team and reused by App and Infrastructure teams without having to configure everything to scratch or having to become Terraform experts. 
Per Terraform documentation, Modules in Terraform are self-contained packages of Terraform configurations that are managed as a group. Modules are used to create reusable components in Terraform as well as for basic code organization. Simply put, no need to rewrite the whole code every time you want to provision something like a Firewall, server, function, load balancer etc..

## Modules directory should have 3 files per Terraform best practices as follow: Note that all Terraform configuration files are named with the .tf file extension.

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

// The resource type is autoscaling group in aws and the user given name in this terraform template is "asg"//
  
  resource "aws_autoscaling_group" "asg" {

// the actual of the autoscaling group when deployed in aws as asg_name and defined in the local definition

 name_prefix = local.asg_name


defines which lauch configuration to use with the value format as <TYPE>.<NAME>.<ARGUMENT> and in this case the argument is the id of the launch config//	  

 launch_configuration= aws_launch_configuration.asg_launch_configuration.id

//min size of the autoscaling group//	 

 min_size = var.minimum_group_size

//max size of the autoscaling group//
	 
 max_size = var.maximum_group_size

//ELB associated with the autoscaling group. Note the splat article or * that iterates the list over many ELB names.//	
  
   load_balancers = aws_elb.proxy_elb.*.name 
   
....

}

Let us now have a look at the actual main.tf file for this lab to deploy our CloudGuard Security Hub. 
we first define the provider as part of the main.tf or as a separate provider.tf file. 
This will allow Terraform to download the aws plugin when initializing the terraform project root directory. It is important to define your aws region. AWS user id and key is required but should be configured as secrets, environment variable or with the aws configure command as Terraform will read the credentials file in the .aws directory:

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

With Terraform, immutability is possible via a state file which allows it to track and compare the configuration with the actual of state of the resources provisioned in aws. This state file is located by default in the local root directory. 
However when using a CICD pipeline, it is important to configure a remote state backend so that the state file can be accessed from multiple environment. Terraform supports multiple backends and we will use S3 for this tutorial

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

> cat /var/lib/jenkins/secrets/initialAdminPassword

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

Scroll down to the Pipeline section and define your Source Code Management or Github in this case where my repository is located. copy and paste your repo address. If the repository is public then there is no need to enter any credentials. I am using only one branch or master. The Jenkinsfile defines the pipeline as code and then click Save.


Credentials:
It is critical to store any credentials such as usernames, password, API keys in an encrypted store. Jenkins provides that with the credentials configuration menu. Please configure your SouceGuard and AWS API keys as Jenkins credentials with the SourceGuard as secret text option and AWS as aws credentials in the Kind drop down menu:


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

```

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

```

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

