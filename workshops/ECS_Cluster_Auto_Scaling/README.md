## EC2 Spot Workshop : ECS_Cluster_Auto_Scaling

##  Overview of lab

Understand different options to Cost optimize container workloads on ECS using Spot at scale. The workshop will cover ECS Capacity providers, Fargate Spot and ECS Cluster Auto scaling. Learn the reason why you'd use one configuration over the other and what are the consequences as a result.


Components of the workshop

### Step1 :  Login to the AWS Workshop Portal

a.	Connect to the portal by browsing to https://dashboard.eventengine.run/
b.	Enter the Hash in the text box, and click Proceed
c.	In the User Dashboard screen, click AWS Console
d.	In the popup page, click Open Console

### Step2 :  Setting up the Cloud9 workspace environment 

•	Select any region of your choice. For this lab, North Virginia (us-east-1) is used
•	Select Services and type cloud9 or (https://console.aws.amazon.com/cloud9)
•	Name it ecsspotworkshop. Click “Next Step”, keep all other defaults and click “Next Step”.  Keep all other defaults and click “Create Environment”

Install JQ and envsubst 

```
sudo yum -y install jq gettext
```
Download the Github repo to your Cloud9 Workshop

```
git clone 
```

### Step3 :  Create IAM roles for your Workspace
In order to work with ECS from our workstation, we will need the appropriate permissions for our developer workstation instance. In this section we will create 3 Roles for ECS. Your existing account may have some roles already setup. But for this workshop, please create them as stated below.
1.	ecsspotworkshop-admin
2.	ecslabinstanceprofile
3.	AutoScalingServiceRolePolicy

Go to the IAM Console, Roles > Create New Role > AWS Service > EC2. We will later assign this role to our workstation instance.
Click Next: Permissions. Confirm that AdministratorAccess is checked
Click Next:Tags Use the defaults, and click Next: Review to review.
Enter ecsspotworkshop-admin for the Name, and click Create role.

Again, Go to the IAM Console, Roles > Create New Role > AWS Service > EC2.
Create another new role so that EC2 instances in the ECS cluster have appropriate permissions to access the container registry, auto-scale, etc. 
We will later assign this role to the EC2 instances in our ECS cluster.
In the Create Role screen, enter AmazonEC2ContainerServiceforEC2Role AmazonEC2ContainerServiceAutoscaleRole in the text field (without a comma) and select the two policies.

In the Review screen, enter ecslabinstanceprofile for the Role name and click Create Role.
Note: By default, the ECS first run wizard creates ecsInstanceRole for you to use. However, it's a best practice to create a specific role for your use so that we can add more policies in the future when we need to. If you are creating ecsInstanceRole then as AdministratorAccess to it.

Use the same process to create another new role so that EC2 Auto scaling will have necessary permissions to launch/terminate resources on your behalf.  
Under the section “Or select a service to view its use cases”, select ‘EC2 Auto scaling’ for the service which will use this role. 

Under the section “Select your use case”, select ‘EC2 Auto scaling’ and click on “Next: Permissions”

We will later use this role when we create auto scaling groups. In the Create Role screen, enter AutoScalingServiceRolePolicy
In the optional suffix, enter “ec2” as shown below so that role become AWSServiceRoleForAutoScaling_ec2


### Step4 :  Attach the IAM role to your Workspace
Now we will be attaching a role to the EC2 Instance which is hosting the Cloud9 environment. Follow this deep link to find your Cloud9 EC2 instance
Select the instance, then choose Actions / Instance Settings / Attach/Replace IAM Role

Now, Return to your Cloud 9 workspace and click the sprocket, or launch a new tab to open the Preferences tab 
Select AWS SETTINGS 
Turn off AWS managed temporary credentials 
Close the Preferences tab

To ensure temporary credentials aren’t already in place we will also remove any existing credentials file:

```
rm -vf ${HOME}/.aws/credentials
```

We should configure our aws cli with our current region as default:

```
export ACCOUNT_ID=$(aws sts get-caller-identity --output text --query Account)
export AWS_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')
echo "export ACCOUNT_ID=${ACCOUNT_ID}" >> ~/.bash_profile
echo "export AWS_REGION=${AWS_REGION}" >> ~/.bash_profile
aws configure set default.region ${AWS_REGION}
aws configure get default.region
```

Now, Use the GetCallerIdentity CLI command to validate that the Cloud9 IDE is using the correct IAM role.

```aws sts get-caller-identity```

The output assumed-role name should contain:

```
{
    "Account": "000474600478", 
    "UserId": "AROAQAHCJ2QPAONSHPAXY:i-01ad7d6cd53ba8945", 
    "Arn": "arn:aws:sts::000474600478:assumed-role/ecsspotworkshop-admin/i-01ad7d6cd53ba8945"
}
```


### Step5 :  Create Launch Template

In this section let us first create a Launch Template (LT) which contain the common configuration including the following. 
•	ECS optimized AMI
•	Instance user data required to bootstrap the instance into the ECS Cluster
•	Instance Profile / Role
•	Instance Types ( Optional )
•	EBS Storage configuration
•	Instance Tags
•	Security Group Ids

Set the following global variables for the names of resources be created in this workshop.

```
export WORKSHOP_NAME=ecs-spot-workshop
export LAUNCH_TEMPLATE_NAME=ecs-spot-workshop-lt
export ASG_NAME_OD=ecs-spot-workshop-asg-od
export ASG_NAME_SPOT=ecs-spot-workshop-asg-spot
export OD_CAPACITY_PROVIDER_NAME=od-capacity_provider
export SPOT_CAPACITY_PROVIDER_NAME=ec2spot-capacity_provider
export ECS_FARGATE_CLUSTER_NAME=EcsSpotWorkshopCluster
export LAUNCH_TEMPLATE_VERSION=1

```

Set the ARN of the IAM role ecslabinstanceprofile created in Section 2. Replace the <AWS Acount ID> with your AWS account 
  
```
export IAM_INSTANT_PROFILE_ARN=arn:aws:iam::<AWS Account ID>:instance-profile/ecslabinstanceprofile
```

You get your account id with below command

```
echo $ACCOUNT_ID
```

Set the default security group in your default VPC

```
export SECURITY_GROUP=$( aws ec2 describe-security-groups --group-names default | jq -r '.SecurityGroups[0].GroupId')
echo "Default Security group is $SECURITY_GROUP"

```

The output from above command looks like below.
Default Security group is sg-xxxxxxxx
Set the latest ECS Optimized AMI 

```
export AMI_ID=$(aws ssm get-parameters --names /aws/service/ecs/optimized-ami/amazon-linux-2/recommended | jq -r 'last(.Parameters[]).Value' | jq -r '.image_id')

echo "Latest ECS Optimized Amazon AMI_ID is $AMI_ID"
```

The output from above command looks like below.
Latest ECS Optimized Amazon AMI_ID is ami-07a63940735ae....

Make sure in the folder there is a file a templates/user-data.txt for the EC2 user data with below code. 

```
#!/bin/bash
echo ECS_CLUSTER=ECS_FARGATE_CLUSTER_NAME >> /etc/ecs/ecs.config

```

Find the launch-template-data.json in templates folder for the Launch Template configuration. 

```

{
  "IamInstanceProfile": {
    "Arn": "%instanceProfile%"
  },
  "ImageId": "%ami-id%",
  "SecurityGroupIds": [
    "%instanceSecurityGroup%"
  ],
  "InstanceType": "t3.micro",
  "BlockDeviceMappings": [
        {
            "DeviceName": "/dev/xvdcz",
            "Ebs": {
                "VolumeSize": 22,
                "VolumeType": "gp2",
                "DeleteOnTermination": true,
                "Encrypted": true
                }
        }
    ],
  "TagSpecifications": [
    {
      "ResourceType": "instance",
      "Tags": [
        {
          "Key": "Name",
          "Value": "%workshopName%"
        },
        {
          "Key": "Env",
          "Value": "Test"
        }
      ]
    }
  ],
  "Monitoring": {
    "Enabled": true
  },
  
  "UserData": "%UserData%"
}
```

Copy the template files to the current directory

```
cp -Rfp templates/*.json .
cp -Rfp templates/*.txt .

```


Run the following commands to substitute the template with actual values from the global variables. 

```
sed -i.bak -e "s#ECS_FARGATE_CLUSTER_NAME#$ECS_FARGATE_CLUSTER_NAME#g"  user-data.txt
sed -i.bak -e "s#%instanceProfile%#$IAM_INSTANT_PROFILE_ARN#g"  launch-template-data.json
sed -i.bak -e "s#%instanceSecurityGroup%#$SECURITY_GROUP#g"  launch-template-data.json
sed -i.bak -e "s#%workshopName%#$WORKSHOP_NAME#g"  launch-template-data.json
sed -i.bak  -e "s#%ami-id%#$AMI_ID#g" -e "s#%UserData%#$(cat user-data.txt | base64 --wrap=0)#g" launch-template-data.json

```

Now let is create the launch template

```
LAUCH_TEMPLATE_ID=$(aws ec2 create-launch-template --launch-template-name $LAUNCH_TEMPLATE_NAME --version-description $LAUNCH_TEMPLATE_VERSION --launch-template-data file://launch-template-data.json | jq -r '.LaunchTemplate.LaunchTemplateId')

echo "Amazon LAUCH_TEMPLATE_ID is $LAUCH_TEMPLATE_ID"

```
The output from above command looks like this
Amazon LAUCH_TEMPLATE_ID is lt-023e2e52afc...
You can view this Launch Template in the Console.


### Step6 : Create Capacity Provider for On-Demand using Auto Scaling Group

In this section, let us first create an Auto Scaling Group for EC2 On-Demand Instances using the Launch Template created in last section

Set the following variables for auto scaling configuration

```
export ASG_NAME=$ASG_NAME_OD
export OD_BASE=0
export OD_PERCENTAGE=100
export MIN_SIZE=0
export MAX_SIZE=10
export DESIREDS_SIZE=0
export INSTANCE_TYPE_1=c4.large
export INSTANCE_TYPE_2=c5.large
export INSTANCE_TYPE_3=m4.large
export INSTANCE_TYPE_4=m5.large
export INSTANCE_TYPE_5=r4.large
export INSTANCE_TYPE_6=r5.large

```

Set the auto scaling service linked role (created in section 2) ARN
Replace the <AWS Acount ID> with your AWS account 
  
``` 
export SERVICE_ROLE_ARN="arn:aws:iam::<AWS Account ID>:role/aws-service-role/autoscaling.amazonaws.com/AWSServiceRoleForAutoScaling_ec2"
```

Select all the default subnets in your default VPC to launch EC2 instances
Run below command to list all the default subnets in your default VPC

```
aws ec2 describe-subnets | jq -r '.Subnets[].SubnetId'
```

Construct and run the below command using the above subnet id values.

```
export PUBLIC_SUBNET_LIST="subnet-764d7d11,subnet-a2c2fd8c,subnet-cb26e686,subnet-7acbf626,subnet-93d490ad,subnet-313ad03f"
```



In this workshop, you will deploy the following:

### Step1 :  Create a Launch Template with ECS optimized AMI and with user data configuring ECS Cluster  
TBD
### Step2 : Create a ASG for only OD instances i.e. ecs-fargate-cluster-autoscale-asg-od with MIN 0 and MAX 10
TBD
### Step3 : Create a Capacity Provider using this ASG i.e. od-capacity_provider_3  with Managed Scaling Enabled with target capacity of 100
TBD
### Step4 : Create a ASG for pnly Spot instances i.e. ecs-fargate-cluster-autoscale-asg-spot)  with MIN 0 and MAX 10
TBD

### Step5 : Create a Capacity Provider using this ASG i.e. spot-capacity_provider_3 with Managed Scaling Enabled with target capacity of 100


### Step6 : Create an ECS cluster (i.e. EcsFargateCluster) with above two capacity providers and with a default capacity provider strategy

The default strategy is od-capacity_provider_3,base=1,weight=1  which means any tasks/services will be deployed in OD if strategy is not explicitly specified while launching them

### Step7 : Add default fargate capacity providers i.e. FARGATE and FARGATE-SPOT to the above cluster
TBD
### Step8 :Create a task definition for fargate i.e. webapp-fargate-task
TBD
### Step9 :Create a task definition for EC2 i.e. webapp-ec2-task
TBD
### Step10 : Deployed 6 Services as follows
TBD
Deploy a service i.e. webapp-ec2-service-od (with 2 tasks) to launch tasks ONLY on OD Capacity Providers
a.	2 tasks gets deployed on OD instances launched from od-capacity_provider_3  
Deploy a service i.e. webapp-ec2-service-spot (with 2 tasks) to launch tasks ONLY on Spot Capacity Providers
a.	2 tasks gets deployed on Spot instances launched from spot-capacity_provider_3
Deploy a service i.e. webapp-ec2-service-mix (with 6 tasks) to launch tasks on both OD(weight=1)  and Spot (weight=3) Capacity Providers
a.	2 tasks on OD instances from od-capacity_provider_3  and 4 tasks on Spot instances from spot-capacity_provider_3
Deploy a service i.e. webapp-fargate-service-fargate (with 2 tasks) to launch tasks ONLY on FARGATE Capacity Provider
a.	2 tasks gets deployed on FARGATE
Deploy a service i.e. webapp-fargate-service-fargate-spot (with 2 tasks) to launch tasks ONLY FARGATE-SPOT Capacity Provider
a.	2 tasks gets deployed on FARGATE
Deploy a service i.e. webapp-fargate-service--mix (with 4 tasks) to launch tasks on both FARGATE(weight=3) and FARGATE-SPOT (weight=1)
a.	3 tasks gets deployed on FARGATE  and 1 tasks on FARGATE-SPOT

### Workshop Cleanup
