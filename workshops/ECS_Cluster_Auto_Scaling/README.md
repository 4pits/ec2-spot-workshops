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

Find file templates/asg.json for the Auto scaling group configuration. 

```
{
  "AutoScalingGroupName": "%ASG_NAME%",
  "MixedInstancesPolicy": {
    "LaunchTemplate": {
      "LaunchTemplateSpecification": {
        "LaunchTemplateName": "%LAUNCH_TEMPLATE_NAME%",
        "Version": "%LAUNCH_TEMPLATE_VERSION%"
      },
      "Overrides": [
        {
          "InstanceType": "%INSTANCE_TYPE_1%"
        },
        {
          "InstanceType": "%INSTANCE_TYPE_2%"
        },
        {
          "InstanceType": "%INSTANCE_TYPE_3%"
        },
        {
          "InstanceType": "%INSTANCE_TYPE_4%"
        },
        {
          "InstanceType": "%INSTANCE_TYPE_5%"
        },
        {
          "InstanceType": "%INSTANCE_TYPE_6%"
        }
      ]
    },
    "InstancesDistribution": {
      "OnDemandAllocationStrategy": "prioritized",
      "OnDemandBaseCapacity": %OD_BASE%,
      "OnDemandPercentageAboveBaseCapacity": %OD_PERCENTAGE%,
      "SpotAllocationStrategy": "lowest-price",
      "SpotInstancePools": 4
    }
  },
  "MinSize": %MIN_SIZE%,
  "MaxSize": %MAX_SIZE%,
  "DesiredCapacity": %DESIREDS_SIZE%,
  "DefaultCooldown": 300,
  "HealthCheckGracePeriod": 300, 
  "HealthCheckType": "EC2",
  "VPCZoneIdentifier": "%PUBLIC_SUBNET_LIST%",
      "TerminationPolicies": [ 
        "DEFAULT" 
  ],
  "NewInstancesProtectedFromScaleIn": true, 
  "ServiceLinkedRoleARN": "%SERVICE_ROLE_ARN%"
}

```
Copy this Template file to the current directory

```
cp -Rfp templates/asg.json .
```

Run the following commands to substitute the template with actual values from the global variables

```
sed -i.bak -e "s#%ASG_NAME%#$ASG_NAME#g"  asg.json
sed -i.bak -e "s#%LAUNCH_TEMPLATE_NAME%#$LAUNCH_TEMPLATE_NAME#g"  asg.json
sed -i.bak -e "s#%LAUNCH_TEMPLATE_VERSION%#$LAUNCH_TEMPLATE_VERSION#g"  asg.json
sed -i.bak -e "s#%INSTANCE_TYPE_1%#$INSTANCE_TYPE_1#g"  asg.json
sed -i.bak -e "s#%INSTANCE_TYPE_2%#$INSTANCE_TYPE_2#g"  asg.json
sed -i.bak -e "s#%INSTANCE_TYPE_3%#$INSTANCE_TYPE_3#g"  asg.json
sed -i.bak -e "s#%INSTANCE_TYPE_4%#$INSTANCE_TYPE_4#g"  asg.json
sed -i.bak -e "s#%INSTANCE_TYPE_5%#$INSTANCE_TYPE_5#g"  asg.json
sed -i.bak -e "s#%INSTANCE_TYPE_6%#$INSTANCE_TYPE_6#g"  asg.json
sed -i.bak -e "s#%OD_BASE%#$OD_BASE#g"  asg.json
sed -i.bak -e "s#%OD_PERCENTAGE%#$OD_PERCENTAGE#g"  asg.json
sed -i.bak -e "s#%MIN_SIZE%#$MIN_SIZE#g"  asg.json
sed -i.bak -e "s#%MAX_SIZE%#$MAX_SIZE#g"  asg.json
sed -i.bak -e "s#%DESIREDS_SIZE%#$DESIREDS_SIZE#g"  asg.json
sed -i.bak -e "s#%OD_BASE%#$OD_BASE#g"  asg.json
sed -i.bak -e "s#%PUBLIC_SUBNET_LIST%#$PUBLIC_SUBNET_LIST#g"  asg.json
sed -i.bak -e "s#%SERVICE_ROLE_ARN%#$SERVICE_ROLE_ARN#g"  asg.json

```

•	Create the auto scaling group for the On Demand Instances

```
aws autoscaling create-auto-scaling-group --cli-input-json file://asg.json
ASG_ARN=$(aws autoscaling describe-auto-scaling-groups --auto-scaling-group-name $ASG_NAME_OD | jq -r '.AutoScalingGroups[0].AutoScalingGroupARN')

echo "$ASG_NAME_OD ARN=$ASG_ARN"

```

The output of the above command looks like below
ecs-spot-workshop-asg-od ARN=arn:aws:autoscaling:us-east-1:000474600478:autoScalingGroup:1e9de503-068e-4d78-8272-82536fc92d14:autoScalingGroupName/ecs-spot-workshop-asg-od 


•	Create a template file templates/ecs-capacityprovider.json for the Capacity Provider Configuration. 

```
{
    "name": "%CAPACITY_PROVIDER_NAME%", 
    "autoScalingGroupProvider": {
    "autoScalingGroupArn": "%ASG_ARN%",
            "managedScaling": {
                "status": "ENABLED",
                "targetCapacity": %TARGET_CAPACITY%,
                "minimumScalingStepSize": 1,
                "maximumScalingStepSize": %MAX_SIZE%
            },
            "managedTerminationProtection": "ENABLED"
    }
}

```

Copy the template file to the current folder

```
cp -Rfp  templates/ecs-capacityprovider.json .
```
Replace the place holder values in the template file

```
export TARGET_CAPACITY=100
export CAPACITY_PROVIDER_NAME=$OD_CAPACITY_PROVIDER_NAME
sed -i.bak -e "s#%CAPACITY_PROVIDER_NAME%#$CAPACITY_PROVIDER_NAME#g"  ecs-capacityprovider.json
sed -i.bak -e "s#%ASG_ARN%#$ASG_ARN#g"  ecs-capacityprovider.json
sed -i.bak -e "s#%MAX_SIZE%#$MAX_SIZE#g"  ecs-capacityprovider.json
sed -i.bak -e "s#%TARGET_CAPACITY%#$TARGET_CAPACITY#g"  ecs-capacityprovider.json
```

Create the On-Demand Capacity Provider with Auto scaling group

```
CAPACITY_PROVIDER_ARN=$(aws ecs create-capacity-provider --cli-input-json file://ecs-capacityprovider.json | jq -r '.capacityProvider.capacityProviderArn')

echo "$OD_CAPACITY_PROVIDER_NAME ARN=$CAPACITY_PROVIDER_ARN"

```

The output of the above command looks like

od-capacity_provider ARN=arn:aws:ecs:us-east-1:000474600478:capacity-provider/od-capacity_provider 


Create Capacity Provider for EC2 Spot using Auto Scaling Group 

In this section, let us first create an Auto Scaling Group for EC2 Spot Instances using the Launch Template created in section earlier
This procedure is exactly same Section 6 except the change in below configuration for EC2 Spot instances



### Step7 : Create Capacity Provider for EC2 Spot using Auto Scaling Group 

•	Copy the Template file and replace the values for EC2 spot configuration

```
cp -Rfp templates/asg.json .
```

•	Replace the values for Auto Scaling Group configuration  for EC2 spot

```
export ASG_NAME=$ASG_NAME_SPOT
export OD_PERCENTAGE=0

sed -i.bak -e "s#%ASG_NAME%#$ASG_NAME#g"  asg.json
sed -i.bak -e "s#%LAUNCH_TEMPLATE_NAME%#$LAUNCH_TEMPLATE_NAME#g"  asg.json
sed -i.bak -e "s#%LAUNCH_TEMPLATE_VERSION%#$LAUNCH_TEMPLATE_VERSION#g"  asg.json
sed -i.bak -e "s#%INSTANCE_TYPE_1%#$INSTANCE_TYPE_1#g"  asg.json
sed -i.bak -e "s#%INSTANCE_TYPE_2%#$INSTANCE_TYPE_2#g"  asg.json
sed -i.bak -e "s#%INSTANCE_TYPE_3%#$INSTANCE_TYPE_3#g"  asg.json
sed -i.bak -e "s#%INSTANCE_TYPE_4%#$INSTANCE_TYPE_4#g"  asg.json
sed -i.bak -e "s#%INSTANCE_TYPE_5%#$INSTANCE_TYPE_5#g"  asg.json
sed -i.bak -e "s#%INSTANCE_TYPE_6%#$INSTANCE_TYPE_6#g"  asg.json
sed -i.bak -e "s#%OD_BASE%#$OD_BASE#g"  asg.json
sed -i.bak -e "s#%OD_PERCENTAGE%#$OD_PERCENTAGE#g"  asg.json
sed -i.bak -e "s#%MIN_SIZE%#$MIN_SIZE#g"  asg.json
sed -i.bak -e "s#%MAX_SIZE%#$MAX_SIZE#g"  asg.json
sed -i.bak -e "s#%DESIREDS_SIZE%#$DESIREDS_SIZE#g"  asg.json
sed -i.bak -e "s#%OD_BASE%#$OD_BASE#g"  asg.json
sed -i.bak -e "s#%PUBLIC_SUBNET_LIST%#$PUBLIC_SUBNET_LIST#g"  asg.json
sed -i.bak -e "s#%SERVICE_ROLE_ARN%#$SERVICE_ROLE_ARN#g"  asg.json

```

•	Create the Auto scaling group for EC2 spot

```
aws autoscaling create-auto-scaling-group --cli-input-json file://asg.json
ASG_ARN=$(aws autoscaling describe-auto-scaling-groups --auto-scaling-group-name $ASG_NAME_SPOT | jq -r '.AutoScalingGroups[0].AutoScalingGroupARN')

echo "$ASG_NAME_SPOT ARN=$ASG_ARN"

```

The output for the above command looks like this

ecs-spot-workshop-asg-spot ARN=arn:aws:autoscaling:us-east-1:000474600478:autoScalingGroup:dd7a67e0-4df0-4cda-98d7-7e13c36dec5b:autoScalingGroupName/ecs-spot-workshop-asg-spot


•	Copy the template file for Capacity Provider to the current directory.

```
cp -Rfp templates/ecs-capacityprovider.json .
```

•	Run the following commands to substitute the template with actual values from the global variables

```
export TARGET_CAPACITY=100
export CAPACITY_PROVIDER_NAME=$SPOT_CAPACITY_PROVIDER_NAME
sed -i.bak -e "s#%CAPACITY_PROVIDER_NAME%#$CAPACITY_PROVIDER_NAME#g"  ecs-capacityprovider.json
sed -i.bak -e "s#%ASG_ARN%#$ASG_ARN#g"  ecs-capacityprovider.json
sed -i.bak -e "s#%MAX_SIZE%#$MAX_SIZE#g"  ecs-capacityprovider.json
sed -i.bak -e "s#%TARGET_CAPACITY%#$TARGET_CAPACITY#g"  ecs-capacityprovider.json

```

•	Create the EC2 Spot Capacity Provider with Auto scaling group

```
CAPACITY_PROVIDER_ARN=$(aws ecs create-capacity-provider --cli-input-json file://ecs-capacityprovider.json | jq -r '.capacityProvider.capacityProviderArn')

echo "$SPOT_CAPACITY_PROVIDER_NAME ARN=$CAPACITY_PROVIDER_ARN"

```

The output of the above command looks like

spot-capacity_provider ARN=arn:aws:ecs:us-east-1:000474600478:capacity-provider/ec2spot-capacity_provider



### Step8 :  Create ECS Cluster with Capacity Providers


Now let’s create the ECS cluster with above 2 Capacity Providers

```
aws ecs create-cluster --cluster-name $ECS_FARGATE_CLUSTER_NAME \
       --capacity-providers $OD_CAPACITY_PROVIDER_NAME $SPOT_CAPACITY_PROVIDER_NAME \
       --default-capacity-provider-strategy capacityProvider=$OD_CAPACITY_PROVIDER_NAME,base=1,weight=1 \
         capacityProvider=$SPOT_CAPACITY_PROVIDER_NAME,weight=0

sleep 10

```
The sleep statement is to allow the ECS cluster status to change to ACTIVE

Go to AWS > Services > ECS
Select the ECS Cluster created in last Step

Click on the Update Cluster on the top right corner to see default Capacity Provider Strategy. As shown base=1 is set for OD Capacity Provider. 
That means if there is no capacity provider strategy specified during the deploying Tasks/Services, ECS by default chooses the OD Capacity Provider to launch them.

Click on Cancel as we don’t want to change the default strategy for now.

Now let’s add two more Capacity Providers i.e. FARGATE and FARGATE_SPOT

```
aws ecs put-cluster-capacity-providers   --cluster $ECS_FARGATE_CLUSTER_NAME \
     --capacity-providers FARGATE FARGATE_SPOT $OD_CAPACITY_PROVIDER_NAME $SPOT_CAPACITY_PROVIDER_NAME  \
     --default-capacity-provider-strategy capacityProvider=$OD_CAPACITY_PROVIDER_NAME,base=1,weight=1 \
         capacityProvider=$SPOT_CAPACITY_PROVIDER_NAME,weight=0  \
     --region $AWS_REGION


```
The ECS cluster should now contain 4 Capacity Providers 2 from Auto Scaling groups (1 for OD and 1 for Spot), 1 from FARGATE and 1 from FARGATE_SPOT


### Step9 :  Create ECS Tasks 

In this section, we will create two tasks definitions: one for EC2 and one for Fargate.  
Fine webapp-fargate-task.json with below contents

```
•	Create a file webapp-fargate-task.json with below contents or directly download from the link
{
    "family": "webapp-fargate-task", 
    "networkMode": "awsvpc", 
    "containerDefinitions": [
        {
            "name": "fargate-app", 
            "image": "httpd:2.4", 
            "portMappings": [
                {
                    "containerPort": 80, 
                    "hostPort": 80, 
                    "protocol": "tcp"
                }
            ], 
            "essential": true, 
            "entryPoint": [
                "sh",
		"-c"
            ], 
            "command": [
                "/bin/sh -c \"echo '<html> <head> <title>Amazon ECS Sample App</title> <style>body {margin-top: 40px; background-color: #333;} </style> </head><body> <div style=color:white;text-align:center> <h1>Amazon ECS Sample App</h1> <h2>Congratulations!</h2> <p>Your application is now running on a container in Amazon ECS.</p> </div></body></html>' >  /usr/local/apache2/htdocs/index.html && httpd-foreground\""
            ]
        }
    ], 
    "requiresCompatibilities": [
        "FARGATE"
    ], 
    "cpu": "256", 
    "memory": "512"
}

```

•	Run the below command to create the task definition

```
aws ecs register-task-definition --cli-input-json file://webapp-fargate-task.json
WEBAPP_FARGATE_TASK_DEF=$(cat webapp-fargate-task.json | jq -r '.family')

```
Make a note of the version number of the task. Ideally it should be 1 if you are registering the task for the first time.

Find a file webapp-ec2-task.json with below contents 

```
{
    "family": "webapp-ec2-task", 
    "containerDefinitions": [
        {
            "name": "fargate-app", 
            "image": "httpd:2.4", 
            "portMappings": [
                {
                    "containerPort": 80, 
                    "hostPort": 80, 
                    "protocol": "tcp"
                }
            ], 
            "essential": true, 
            "entryPoint": [
                "sh",
		"-c"
            ], 
            "command": [
                "/bin/sh -c \"echo '<html> <head> <title>Amazon ECS Sample App</title> <style>body {margin-top: 40px; background-color: #333;} </style> </head><body> <div style=color:white;text-align:center> <h1>Amazon ECS Sample App</h1> <h2>Congratulations!</h2> <p>Your application is now running on a container in Amazon ECS.</p> </div></body></html>' >  /usr/local/apache2/htdocs/index.html && httpd-foreground\""
            ]
        }
    ], 
    "requiresCompatibilities": [
        "EC2"
    ], 
    "cpu": "256", 
    "memory": "512"
}

```


•	Run the below command to create the task definition

```
aws ecs register-task-definition --cli-input-json file://webapp-ec2-task.json
WEBAPP_EC2_TASK_DEF=$(cat webapp-ec2-task.json | jq -r '.family')
```

### Step10 :  Create ECS Services 


Now we will create couple of ECS services from the Task definition which would have their own Capacity Provider Strategy

1. webapp-ec2-service-od 

```
export ECS_SERVICE_NAME=webapp-ec2-service-od
export TASK_COUNT=2

aws ecs create-service \
     --capacity-provider-strategy capacityProvider=$OD_CAPACITY_PROVIDER_NAME,weight=1 \
     --cluster $ECS_FARGATE_CLUSTER_NAME \
     --service-name $ECS_SERVICE_NAME \
     --task-definition $WEBAPP_EC2_TASK_DEF:4 \
     --desired-count $TASK_COUNT \
     --region $AWS_REGION 

```

This service deployment triggers Cloud watch alarms for Cluster Capacity. Based on the specified weightage for Capacity Providers, the OD Capacity Provider in this case (i.e. corresponding Auto scaling group) scales 2 instances to schedule 2 tasks for this service.

2. webapp-ec2-service-spot

```
export ECS_SERVICE_NAME=webapp-ec2-service-spot
export TASK_COUNT=2

aws ecs create-service \
     --capacity-provider-strategy capacityProvider=$SPOT_CAPACITY_PROVIDER_NAME,weight=1 \
     --cluster $ECS_FARGATE_CLUSTER_NAME \
     --service-name $ECS_SERVICE_NAME \
     --task-definition $WEBAPP_EC2_TASK_DEF:4 \
     --desired-count $TASK_COUNT \
     --region $AWS_REGION 
```

This service deployment triggers Cloud watch alarms for Cluster Capacity. Based on the specified weightage for Capacity Providers, the Spot Capacity Provider in this case (i.e. corresponding Auto scaling group) scales 2 instances to schedule 2 tasks for this service.

3. webapp-ec2-service-mix 

```
export ECS_SERVICE_NAME=webapp-ec2-service-mix
export TASK_COUNT=6

aws ecs create-service \
     --capacity-provider-strategy capacityProvider=$OD_CAPACITY_PROVIDER_NAME,weight=1 \
                                  capacityProvider=$SPOT_CAPACITY_PROVIDER_NAME,weight=3 \
     --cluster $ECS_FARGATE_CLUSTER_NAME \
     --service-name $ECS_SERVICE_NAME \
     --task-definition $WEBAPP_EC2_TASK_DEF:4 \
     --desired-count $TASK_COUNT \
     --region $AWS_REGION

```

This service deployment triggers Cloud watch alarms for Cluster Capacity. Based on the specified weightage (i.e. OD=1, Spot=3) for Capacity Providers, Both OD and  Spot Capacity Provider in this case (i.e. corresponding Auto scaling groups) scales 4 (Spot) and 3(OD) instances to schedule 6 tasks for this service

4. fargate-service-fargate 

Change Subnet ID here "subnet-XXXXX" and Security Group "sg-4f3f0d1e"

```
export ECS_SERVICE_NAME=fargate-service-fargate
export TASK_COUNT=2

aws ecs create-service \
     --capacity-provider-strategy capacityProvider=FARGATE,weight=1 \
     --cluster $ECS_FARGATE_CLUSTER_NAME \
     --service-name $ECS_SERVICE_NAME \
     --task-definition $WEBAPP_FARGATE_TASK_DEF:4 \
     --desired-count $TASK_COUNT \
     --region $AWS_REGION \
 	 --network-configuration "awsvpcConfiguration={subnets=[subnet-764d7d11],securityGroups=[sg-4f3f0d1e],assignPublicIp="ENABLED"}"
```
ECS Fargate provisions computing resources to deploy 2 Fargate tasks.

5. webapp-fargate-service-fargate-spot 

Change Subnet ID here "subnet-XXXXX" and Security Group "sg-4f3f0d1e"

```
export ECS_SERVICE_NAME=webapp-fargate-service-fargate-spot
export TASK_COUNT=2
aws ecs create-service \
     --capacity-provider-strategy capacityProvider=FARGATE_SPOT,weight=1 \
     --cluster $ECS_FARGATE_CLUSTER_NAME \
     --service-name $ECS_SERVICE_NAME \
     --task-definition $WEBAPP_FARGATE_TASK_DEF:4 \
     --desired-count $TASK_COUNT \
     --region $AWS_REGION \
 	 --network-configuration "awsvpcConfiguration={subnets=[subnet-764d7d11],securityGroups=[sg-4f3f0d1e],assignPublicIp="ENABLED"}"


```




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
