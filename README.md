## **Summary**: In this high-level document, I am sharing how I used AWS console services to create AWS services to register a docker container image on an ECR repo, deploy an ECS cluster running the image with Auto Scaling Group and an Application Load Balancer.

 

## Create IAM ECS Instance and Task roles

- These roles are needed for     the ECS container instance and ECS Task to access the necessary ECS     services such as ECR and CloudWatch (https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html).
- Create “ecsInstanceRole”     based on AWS Managed policy [AmazonEC2ContainerServiceforEC2Role](https://us-east-1.console.aws.amazon.com/iam/home#/policies/arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceforEC2Role)     with for EC2 service trusted entity
- Create role     “ecsTaskExecutionRole” based on AWS Managed policy [AmazonECSTaskExecutionRolePolicy](https://us-east-1.console.aws.amazon.com/iam/home#/policies/arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy)     for ecs-tasks trusted entity
       
       

## Create A Repository 

- Use the     Amazon Elastic Container Registry Console to create a repo for an     application, “autopics” in my setup.
- Once the     repo is created, follow the “View push commands” to push the docker image     to the registry. For tutorial purpose, one can use an image like     nginx:latest from docker hub and tagging it with your ECR domain (see     instructions on the “View push commands”).

 

## Create an EC2 Task Definition

- In the ECS console,     create an EC2 task definition for the ECR image

![img](./images/clip_image002-1677612425942-80.png)

![img](./images/clip_image004-1677612425942-81.png)

- Add a container docker     image from the ECR uploaded earlier

- Since multiple tasks can      run on one ECS container instance, leave the host port 0 enabling dynamic      forwarding. The autopics application listens on port 3000. The ALB      manages the traffic from external access forwarding requests to the right      host port.

![img](./images/clip_image006-1677612425942-82.png)

- Create the task definition

## Create Security Groups for the Application Load Balancer and EC2 instance.

- EC2 instance allows HTTP     traffic from the ALB
- Create a security, my-alb,     allowing external traffic on HTTP. The load balancer of the ECS service     will use this security group.
- Create a security group,     my-ecs, allowing inbound traffic to instance hosts only on port-range 32768     – 65535 from source my-alb security group of the load balancer. 

 

## Create the Application Load Balancer

·     Create an Internet facing ALB

![img](./images/clip_image008-1677612425942-83.png)

- Create a default target as it’s required for the ALB, but the target created by the ECS service created in the following section is used instead. Specify the my-alb created earlier as the security group

![img](./images/clip_image010-1677612425942-85.png)

## Create an ECS EC2 Cluster

- Create EC2 cluster with an     initial of two t2.micro instances to start. Since this is a dev     environment, I am creating these instances in the public VPC’s for ease of     troubleshooting via SSH. For production, put the instances in private VPC’s     per best practice. 

![img](./images/clip_image011-1677612425942-84.png)

- I specified Key pair for     SSH access for troubleshooting if necessary.
- For the Networking below,     select “my-ecs” security group created above for the instances.

 

- ![img](./images/clip_image013-1677612425942-86.png)Specify the ecsInstanceRole     IAM role created earlier, so the container instances can call the     necessary services to run the cluster.

![img](./images/clip_image015-1677612425942-87.png)

- When creating the cluster,     ASG and Launch Configuration will be automatically created also. The auto     created ASG will be used when the Capacity Provider is created for the     cluster.

##  

## Create ECS Service on the Cluster.

- Service component is needed to scale Tasks and integrated with Capacity Provider to auto scale and load balance the container instances. One can run the task on the cluster without a Service/Capacity Provider but the setup cannot be scaled or load balanced.

![img](./images/clip_image017-1677612425942-88.png)

- Attach the ALB during the service creation as ALB cannot be added later.

![img](./images/clip_image019-1677612425942-89.png)

- Note that the AWSServiceRoleForECS role is created automatically if it does not exist already.

![img](./images/clip_image021-1677612425942-90.png)

- A new target group, my-ecs-service-target, should be created and attached to the ALB listener. Verify that’s done.

- I am not doing the Service Task Auto Scaling. This can be done later on Service tab of the cluster. We can also manually change the desired task count on service to test the container instance auto scaling (capacity provider added next section).

![img](./images/clip_image023-1677612425942-91.png)

- I requested 4 tasks to be started, but only 2 started. Due to cpu/memory constraints, only one task can start on each instance.  Each task takes 1 vCPU and 500 MB RAM, so only tasks can run for the given 2 instances. 

- Our cluster was setup only two instances. We can manually increase 2 more instances or setup Capacity Provider to dynamically scale up/down as needed to meet the desired 4 tasks running. We can setup the Capacity Provider with the service next.

![img](./images/clip_image025-1677612425943-92.png)

![img](./images/clip_image027-1677612425943-93.png)

## Create Capacity Provider to Auto Scale the Service Container Instances

![img](./images/clip_image029-1677612425943-95.png)

- Use the Auto Scaling group was automatically generated when the cluster was created which associated with a Launch configuration (auto created based on the cluster instance setup).

- Set the Target capacity 50% which should trigger the auto scaling to scale out based on the current metrics. CloudWatch alarm for the Target capacity will be created.

- If the capacity provider does not scale out to meet 4 tasks desired, check the auto scale settings to ensure they are set correctly (e.g. maximum should be >= 4):

![img](./images/clip_image031-1677612425943-94.png)

- Auto scaling triggered, and we now have 4 tasks running on for instances:

![img](./images/clip_image033-1677612425943-96.png)

![img](./images/clip_image035-1677612425944-98.png)

![img](./images/clip_image037-1677612425944-97.png)

 

## Create a Route53 Alias to the ALB

- I have a domain kylemma.com which I will create an alias for long ALB DNS below.

- I can also create SSL using the AWS Certificate Manager to register SSL/TLS certificate for my domain which I will skip.

![img](./images/clip_image039-1677612425944-99.png)

- Use domain to access the application instead of the ALB DNS.

![img](./images/clip_image041-1677612425944-100.png)

## Tear Down the Cluster

- Delete the Capacity Provider

- Delete the cluster Service

- Delete the Cluster which will automatically delete the ASG, CloudWatch alarms (high/low triggers), corresponding Launch configuration and its registered instances

- Delete the ALB, repo and other services created earlier if not needed anymore

 