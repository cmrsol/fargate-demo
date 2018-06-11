#### Solutions considered:
A RESTful web service seemed to be an obvious way to solve the problem
presented. AWS has several patterns that could implement a RESTful
web service e.g. API Gateway / Lambda, EC2 / Autoscaling, Classic ECS etc.

At the time the project came into our backlog we had just heard of ECS Fargate. 
ECS Fargate does have a few limitations (scratch storage, CPU and memory) none of
which were a problem. So EC2 / Autoscaling and Classic ECS were ruled out because 
they would have introduced unneeded complexity. The unneeded complexity is mostly
around management of EC2 instances to either run the application or the container
needed for the solution.

When the project came into our group there had been a substantial proof-of-concept 
done. This proof-of-concept had been done with a Docker container. We are API Gateway /
Lambda proponents but there is no need to re-invent the wheel and Fargate would easily 
run the existing container. So... we had an ECS Fargate project!

#### Solution description:
**Overview:**
our group is very biased toward using existing AWS services to bring a complete project 
to the production environment. With that said here is list of the AWS services used for 
the complete solution:
* CodeCommit, CodePipeline and CodeBuild are used for the CI/CD tooling
* CloudFormation is our preferred method to describe, create and manage AWS resources
* AWS Elastic Container Registry is used to store the needed Docker container images
* AWS Systems Manager Parameter Store to hold secrets like database passwords
* and obviously ECS Fargate for the actual application stack

**CI/CD Pipeline:** 
a complete discussion of the CI/CD pipeline for the project is beyond the scope of this
post. However, in broad strokes the pipeline is:
1. Compile some C++ code wrapped in Python, create a Python wheel and publish it to an artifact store
2. Create a Docker image with that wheel installed and publish it to ECR
3. Deploy and test the new image to our test environment
4. Deploy the new image to the production environment

**The solution:**
our environments are all constructed with CloudFormation templates. Each environment is constructed
in a separate AWS account and connected back to a central utility account. The infrastructure stacks 
export a number useful bits like VPC, subnets, IAM roles, security groups etc. This scheme allows us
to move projects through the several accounts without changing the CloudFormation templates just the 
parameters that are fed into them.

For this solution we leverage an existing VPC, set of subnets, IAM role and ACM certificate in the
us-east-1 region. The solution stack describes and manages the following resources:
* **AWS::ECS::Cluster**
* AWS::EC2::SecurityGroup
* AWS::EC2::SecurityGroupIngress
* AWS::Logs::LogGroup
* **AWS::ECS::TaskDefinition**
* AWS::ElasticLoadBalancingV2::LoadBalancer
* AWS::ElasticLoadBalancingV2::TargetGroup
* AWS::ElasticLoadBalancingV2::Listener
* **AWS::ECS::Service**
* AWS::ApplicationAutoScaling::ScalableTarget
* AWS::ApplicationAutoScaling::ScalingPolicy
* AWS::ElasticLoadBalancingV2::ListenerRule
* AWS::Route53::RecordSet

Again, a complete discussion of all the resources for the solution is beyond the scope of this post. 
However, we should explore the resource definitions of the ECS Fargate specific compoonets.

**AWS::ECS::Cluster:**
```
"ECSCluster": {
    "Type":"AWS::ECS::Cluster",
    "Properties" : {
        "ClusterName" : { "Ref": "clusterName" }
    }
}
```

**AWS::ECS::TaskDefinition:**
```
"fargateDemoTaskDefinition": {
    "Type": "AWS::ECS::TaskDefinition",
    "Properties": {
        "ContainerDefinitions": [
            {
                "Essential": "true",
                "Image": { "Ref": "taskImage" },
                "LogConfiguration": {
                    "LogDriver": "awslogs",
                    "Options": {
                        "awslogs-group": {
                            "Ref": "cloudwatchLogsGroup"
                        },
                        "awslogs-region": {
                            "Ref": "AWS::Region"
                        },
                        "awslogs-stream-prefix": "fargate-demo-app"
                    }
                },
                "Name": "fargate-demo-app",
                "PortMappings": [
                    {
                        "ContainerPort": 80
                    }
                ]
            }
        ],
        "Cpu": { "Ref": "cpuAllocation" },
        "ExecutionRoleArn": {"Fn::ImportValue": "fargateDemoRoleArnV1"},
        "Family": {
            "Fn::Join": [
                "",
                [ { "Ref": "AWS::StackName" }, "-fargate-demo-app" ]
            ]
        },
        "Memory": { "Ref": "memoryAllocation" },
        "NetworkMode": "awsvpc",
        "RequiresCompatibilities" : [ "FARGATE" ],
        "TaskRoleArn": {"Fn::ImportValue": "fargateDemoRoleArnV1"}
    }
}
```

**AWS::ECS::Service:**
```
"fargateDemoService": {
     "Type": "AWS::ECS::Service",
     "DependsOn": [
         "fargateDemoALBListener"
     ],
     "Properties": {
         "Cluster": { "Ref": "ECSCluster" },
         "DesiredCount": { "Ref": "minimumCount" },
         "LaunchType": "FARGATE",
         "LoadBalancers": [
             {
                 "ContainerName": "fargate-demo-app",
                 "ContainerPort": "80",
                 "TargetGroupArn": { "Ref": "fargateDemoTargetGroup" }
             }
         ],
         "NetworkConfiguration":{
             "AwsvpcConfiguration":{
                 "SecurityGroups": [
                     { "Ref":"fargateDemoSecuityGroup" }
                 ],
                 "Subnets":[
                    {"Fn::ImportValue": "privateSubnetOneV1"},
                    {"Fn::ImportValue": "privateSubnetTwoV1"},
                    {"Fn::ImportValue": "privateSubnetThreeV1"}
                 ]
             }
         },
         "TaskDefinition": { "Ref":"fargateDemoTaskDefinition" }
     }
}
```

**The complete story:** the snippets above are instructive. If you
want explore a mostly complete sanitized version of the stack there are
two repos at GitHub you can explore. *Note: the components in the repos are
just code to show "It works!" from an ECS Fargate solution.*:

