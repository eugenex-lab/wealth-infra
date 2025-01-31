Introduction
This document provides information concerning the infrastructure as a code for Wealth Services on the Amazon AWS platform

Compute Setup
This section give you a guide on how to setup a compute instance that will be used to manage the infrastructure

If you don't have an SSH key already, generate a new SSH key and associate with the bit bucket account

ssh-keygen -t rsa -b 4096 && \
cat ~/.ssh/id_rsa.pub
mkdir $HOME/infra-code && \
    cd $HOME/infra-code && \
    git clone git@bitbucket.org:sankoredevelopers/wealth-infrastructure-code.git && \
    profile_location=$(./wealth-infrastructure-code/tools/infrastructure_pre_setup | tail -n 1) && \
    source $profile_location && ./wealth-infrastructure-code/tools/infrastructure_setup
Environment Setup
Prerequisite
This documentation makes references and uses code available for Wealth staging. project root would refer to the directory

<repo_root>/terraform/aws/staging
Setup
To get started, there are certain prerequist that needs to be completed. The requirements are dividend into two

One-time Setup
Individual User Setup
One-Time Setup
The one time setup contains actions that must be executed just once. And these actions are to be carried out on the AWS account where this infrastructure will be deployed into.

Terraform Backend Setup
As Dev-Ops engineers interact with Terraform scripts to deploy and update infrastructure, terraform keeps a state of each execution which enables terraform know what was done at each time, what was updated, what was created etc. With this, terraform can easily execute a rollback on created or update infrastructure. A problem arise therefore when there are now multiple users making changes to the terraform infrastructure. Each state will reside locally on the user's machine. With this, if user A makes a change and execute that code on his system, should user B want to rollback that change on his own system, terraform will not be able to, because the state for the change made by user A resides in user A's laptop.

Note Terraform manages infrastructure state using what it calls a Backend

To help solve this problem, Terraform can use a remote backend. With a remote backend, the state of each user's infrastructure changes do not reside on the user's system, but on a central remote service. One of this remote backend system on Terraform is AWS S3

So here we will have to configure and AWS S3 account that will be used by Terraform to manage state, as well as an AWS DynamoDB that will be used for Locking.

Note Locking is important because it prevents two users from updating the state of an infrastructure at the same time

Create your S3 Bucket
Visit the AWS S3 console to create an S3 bucket for terraform state management. We recommend creating a new bucket instead of using an already existing one
Your bucket should have the following attributes
Disable Bucket ACL. It's advisable to not use ACL but to instead have it disabled, thus making the Bucket Creator the owner of objects created on the bucket
Ensure the created bucket is Private (block all public access to the bucket)
Enable Bucket Versioning
And finally enable server-side encryption for the bucket
Note that any Dev-Ops user who is going to use the Terraform infrastructure would have to be given the necessary Permission to access the bucket. The S3 policy need for terraform to work are
s3:ListBucket
s3:GetObject
s3:PutObject
s3:DeleteObject
Create your DynamoDB Table
In your AWS management console, navigate to the DynamoDB service click on Create Table
Provide a name for your new table (name will be used later on)
Your table should have partition key called LockID and should be of type String. Terraform requires that your Table has a partition key with the name "LockID" and of type String.
Updating the backend configuration
On your project root, update the file backend.tf and change the value of bucket and dynamodb_table with the bucket name and dynamodb table that was created above and issue the following command from your project root

$ terraform init
CodeBuild Bitbucket
Go to Codebuild on aws, attempt to create a new codebuild project In the source select BitBucket Click on the Connect BitBucket Account to start the Oauth flow which allows codebuild to interact with bitbucket.

Add Database Peering Security Group
To enable our compute instances to be able to connect with the database in another VPC, a security group with the Name "Database-Peering-Into-DBVPC-{env}" is created. Add that security group to the database instance

jq tool is required for json parsing

create secrets on secret manager (let's do this using terraform)

Environment Variables
The Following environment variables should be set

export secrets to secret manager. The name of the secret can be found in the wealthng output file using the key project_info.application_secretmanager_key SSO_SERVICE_URL: WEALTH_PAYINTEGRATE_DB_HOST: The WEALTH_PAYINTEGRATE_DB: WEALTH_PAYINTEGRATE_USER: WEALTH_PAYINTEGRATE_PASS:

WEALTH_DB_HOST WEALTH_DB_NAME: WEALTH_DB_USER WEALTH_DB_PASS

AEROSPIKE_HOST:

service_url_ls_ligare: service_url_ls_enquiry: service_url_ls_investment: service_url_ls_approvalasaservice: service_url_ls_otp: service_url_ls_messaging: service_url_ls_oms: service_url_ls_transaction: service_url_ls_userprofile: service_url_ls_thirdpartyintegration: service_url_ls_paymentintegration: service_url_ls_advisor: service_url_ls_configuration: service_url_wealth_portal service_url_black_middleware

Unknown services service_url_ls_ixtracsearch: service_url_ls_media: service_url_ls_urlshortner: service_url_ls_approvalworkflow: service_url_ls_fixconnect: service_url_ls_botnotification:

ensure to add a security group rule in the database security group to allow connection from the new VPC After installing wealthng on terraform, add values to the generated aws secret so the volume mounting does not fail in kubernets For some reason after installing the ingress controller, the created target group overrides port for health check. Change that port to use the default target port

sankore-terraform --target core --environment staging --terraform "init" sankore-terraform --target core --environment staging --terraform "apply"

sankore-terraform --target k8_cluster --environment staging --terraform "init" sankore-terraform --target k8_cluster --environment staging --terraform "apply"

sankore-terraform --target k8_workload --environment staging --terraform "init" sankore-terraform --target k8_workload --environment staging --terraform "apply"

wealth-kubernetes setup_newly_created_cluster

sankore-terraform --target wealthng --environment staging --terraform "init"sankore-terraform --target wealthng --environment staging --terraform "apply" wealth-codebuild build-all-others --product "wealthng" wealth-codebuild build_all_libraries --product wealthng wealth-kubernetes setup_after_project_terraform_install --product wealthng wealth-kubernetes publish_helm_chart --product wealthng --chart_folder "wealth-portal" --release_name "wealth-portal" --chart_version "0.1.1-alpha.1.1+0926e72" --docker_image_version "0926e72-1"

wealth-kubernetes publish_helm_chart --product wealthng --chart_folder "ligare-service" --release_name "ls-messaging" --chart_version "0.1.8-alpha.8+227d339" --docker_image_version "x227d339-8"

wealth-kubernetes publish_helm_chart --product wealthng --chart_folder "black-middleware" --release_name "black-middleware" --chart_version "0.1.8-alpha.8+227d339" --docker_image_version "x227d339-8" --dry_run_only "yes"

wealth-codebuild build_library --library "ls-ligare-stage"

Install Kafka wealth-kubernetes install_kafka

Install prometus stack wealth-kubernetes install_independent_chart --action "install" --chart "kube-prometheus-stack" --dry_run_only "no"

Other Commands wealth-kubernetes uninstall_setup_newly_created_cluster
