DevOps Infrastructure Demo
===========================

This repository houses demo code for Modus Create's DevOps talks and meetups.

Originally this was targeted towards the [DevOps Wall Street](http://devopsnyc.co) talk titled _Multi-Cloud Deployment with GitHub and Terraform_. See the branch [demo-20170303](https://github.com/ModusCreateOrg/devops-infra-demo/tree/demo-20170303) for the code demonstrated at that event.

See the branch [demo-20180619](https://github.com/ModusCreateOrg/devops-infra-demo/tree/demo-20180619) for the code for the demo for the [NYC DevOps talk _Applying the CIS Baseline using Ansible & Packer_](https://www.meetup.com/nycdevops/events/fmgjmnyxjbzb/). Slides from this presentation are on [SlideShare](https://www.slideshare.net/RichardBullingtonMcG/applying-the-cis-benchmark-using-ansible-packer).

See the branch [demo-20180926](https://github.com/ModusCreateOrg/devops-infra-demo/tree/demo-20180926) for the code for the demo for the [Continuous Delivery NYC talk _Managing Expensive or Destructive Operations in Jenkins CI_](https://www.meetup.com/ContinuousDeliveryNYC/events/254036209/). Slides from this presentation are on [SlideShare](https://www.slideshare.net/RichardBullingtonMcG/managing-expensive-or-destructive-operations-in-jenkins-ci).

See the branch [demo-20181205](https://github.com/ModusCreateOrg/devops-infra-demo/tree/demo-20181205) for the code for the demo for the [Ansible NYC talk _Ansible Image Bakeries: Best Practices & Pitfalls_](https://www.meetup.com/Ansible-NYC/events/256728741/). Slides from this presentation are on [SlideShare](https://www.slideshare.net/RichardBullingtonMcG/ansible-image-bakeries-best-practices-and-pitfalls).
 
See the branch [demo-20190130](https://github.com/ModusCreateOrg/devops-infra-demo/tree/demo-20180130) for the code for the demo for the [Big Apple DevOps  talk _Monitoring and Alerting as code with Terraform and New Relic_](https://www.meetup.com/Big-Apple-DevOps/events/257744262/).
 
Instructions
------------

 To run the demo end to end, you will need:
 
* [AWS Account](https://aws.amazon.com/)
* [Google Cloud Account](https://cloud.google.com/)
* [Docker](https://docker.com/) (tested with 18.05.0-ce)
* [Packer](https://www.packer.io/) (tested with 1.0.3)
* [Terraform](https://www.terraform.io/) (tested with  v0.11.7)

Optionally, you can use Vagrant to test ansible playbooks locally and Jenkins to orchestrate creation of AMIs in conjunction with GitHub branches and pull requests.

You will also need to set a few environment variables. The method of doing so will vary from platform to platform. 

```
AWS_PROFILE
AWS_DEFAULT_PROFILE
AWS_DEFAULT_REGION
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
GOOGLE_CLOUD_KEYFILE_JSON
GOOGLE_PROJECT
GOOGLE_REGION
PACKER_AWS_VPC_ID
PACKER_AWS_SUBNET_ID
```

A [sample file](env.sh.sample) is provided as a template to customize:

```
cp env.sh.sample env.sh
vim env.sh
. env.sh
```

The AWS profile IAM user should have full control of EC2 in the account you are using.

You will need to create an application in the Google developer console, create a set of service-to-service JSON credentials, and enable the Google Cloud Storage API in the referenced Google developer application for the Google integration to work. If you don't care about that, alternately you may remove the `terraform/google.tf` file to get the demo to work without the Google part.

### Packer

Run `packer/bin/pack.sh` to initiate a Packer run. This will provision a machine on EC2, configure it using Ansible, and scan it using OpenSCAP. The results from the scan will end up in `packer/build`.

Optionally, you can use Vagrant to test ansible playbooks locally and Jenkins to orchestrate creation of AMIs in conjunction with GitHub branches and pull requests.

### Vagrant

In order to make developing the Ansible playbooks faster, a Vagrantfile is provided to provision a VM locally.

Install [Vagrant](https://www.vagrantup.com/). Change directory into the root of the repository at the command line and issue the command `vagrant up`. You can add or edit Ansible playbooks and support scripts then re-run the provisioning with `vagrant provision` to refine the remediations. This is more efficient that re-running packer and baking new AMIs for every change.

### Terraform

This Terraform setup stores its state in Amazon S3 and uses DynamoDB for locking. There is a bit of setup required to bootstrap that configuration. You can use [this repository](https://github.com/monterail/terraform-bootstrap-example) to use Terraform to do that bootstrap process. The `backend.tfvars` file in that repo should be modified as follows to work with this project:

(Replace us-east-1 and XXXXXXXXXXXX with the AWS region and your account ID)
```
bucket = "tf-state.devops-infra-demo.us-east-1.XXXXXXXXXXXX"
dynamodb_table = "TerraformStatelock-devops-infra-demo"
key = "terraform.tfstate"
profile = "terraform"
region = "us-east-1"
```
You'll also need to modify the list of operators who can modify the object in the S3 bucket. Put in the IAM user names of the user into the `setup/variables.tf` file in that project. If your Jenkins instance uses an IAM role to grant access, give it a similar set of permissions to those granted on in the bucket policy to IAM users.

These commands will then set up cloud resources using terraform:
 
    cd terraform
    terraform init
    terraform get
    # Example with values from our environment (replace with values from your environment)
    # terraform plan -var domain=modus.app -out tf.plan
    terraform plan -out tf.plan -var 'domain=example.net'
    terraform apply tf.plan
    # check to see if everything worked - use the same variables here as above
    terraform destroy -var 'domain=example.net'

Alternatively, use the wrapper script in `bin/terraform.sh` which will work interactively or from CI:

   bin/terraform.sh plan
   bin/terraform.sh apply
   bin/terraform.sh plan-destroy
   bin/terraform.sh destroy

This assumes that you already have a Route 53 domain in your AWS account created.
You need to either edit variables.tf to match your domain and AWS zone or specify these values as command line `var` parameters.

The application loads an image from Google storage. To get it loading correctly, edit the `application/assets/css/main.css` file and replace `example-media-website-storage.storage.googleapis.com` with a DNS reference for your Google storage location.

### CodeDeploy

The application enclosed in this demo is packaged and deployed using [AWS CodeDeploy](https://aws.amazon.com/codedeploy/). The script `codedeploy/bin/build.sh` will package the application so that it can be deployed on the AMI built with Ansible and Packer.

The application contains both a simple HTML web site, and a Python app that has an API endpoint of `/api/spin` that spins the CPU of the server, in order to more easily test CPU-sensing auto scaling scale-out operations.

### JMeter

A JMeter test harness that will allow testing of a the application 
### Jenkins

A `Jenkinsfile` is provided that will allow Jenkins to execute Packer and Terraform, package a CodeDeploy application, and even run JMeter performance tests. In order for Jenkins to do this, it needs to have AWS credentials set up, preferably through an IAM role, granting full control of EC2 and VPC resources in that account, and write access to the S3 bucket used for storing CodeDeploy applications. Packer needs this in order to create AMIs, key pairs, etc, Terraform needs this to create a VPC and EC2 resources, and CodeDeploy needs this to store the artifact it creates. This could be pared down further through some careful logging and role work.

The scripts here assume that Jenkins is running on EC2 and uses instance data from the Jenkins executor to infer what VPC and subnet to launch the new EC2 instance into.  The AWS profile IAM user associated with your Jenkins instance or the Jenkins user's AWS credentials should have full control of EC2 in the account you are using.

This script relies on Jenkins having a secret file containing the Google application credentials in JSON with the id "terraform-demo.json". You will need to add that to your Jenkins server's credentials.

# Modus Create

[Modus Create](https://moduscreate.com) is a digital product consultancy. We use a distributed team of the best talent in the world to offer a full suite of digital product design-build services; ranging from consumer facing apps, to digital migration, to agile development training, and business transformation.

[![Modus Create](https://res.cloudinary.com/modus-labs/image/upload/h_80/v1533109874/modus/logo-long-black.png)](https://moduscreate.com)

This project is part of [Modus Labs](https://labs.moduscreate.com).

[![Modus Labs](https://res.cloudinary.com/modus-labs/image/upload/h_80/v1531492623/labs/logo-black.png)](https://labs.moduscreate.com)

# Licensing

This project is [MIT licensed](./LICENSE).

The content in `application` is adapted from _Dimension_ by https://html5up.net/ and is [licensed under a Creative Commons Attribution 3.0 License](https://html5up.net/license)
