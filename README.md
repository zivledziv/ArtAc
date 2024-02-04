# ArtAc

## part 1 - Manually deploy with ECS

### steps of deployment:

### Create an application and Dockerize it:
1. Write an application. it can be a a flask web app. (app.py)
2. Write a compatible Dockerfile to build an image for that app. (Dockerfile)
3. Build the image and tag it. (if using Dockerhub: repoName/imageName:Tag)
4. Push it to your repo. (if using Dockerhub: docker push repoName/imageName:Tag)

#### Networking:
in AWS: Configure VPC, subnets (auto asssign ipv4 for ssh connection for debugging)
and security groups (one for ecs and one for alb)
ecs - inbound: allow ssh from any + allow any from alb
alb - inbound: allow all

#### ECS:
1. Go to the ECS dashboard in the AWS Management Console.
2. Click "Create Cluster": Choose a cluster name, select "EC2", OS image (amazon linux 2), instance type (t2.micro), Set the number of instances (2-3), set key pair, choose vpc, attach subnets. 
3. create task definition: choose name, select "EC2", Network mode: bridge (for easy port mapping), cpu and memory (0.5 unit each), container: name, image (example for dockerhub: docker.io/zivlederer/artac-app:latest), enter relevant port mappings (5000:5000).
4. create ecs service inside the new cluster: choose newly created task definition family, service name, how many tasks you desire (2).
Wanted result: 2 instances of ec2 should be created and running your application.  

#### Application load balancer:
1. navigate to "Target groups" and then create one: set only vpc, leave the rest as default.
2. navigate to "Load Balancers" and then "Create Load Balancer".
3. Choose application LB type: set name, vpc, attach minimum 2 subnets, attach alb security group we created, attach newly created target group. leave the rest as default.
Wanted result: copy the dns from the new alb and try to run it on your browser. you should access and see your app.

### challenges faced and how they were resolved:
1. at first I wanted to use ecr but I encounter some problem with not being able to push images to it because of auth formatting. some time was burned on this and it didnt work so i moved to dockerhub.
2. It was hard to know what is free to use. Burned time on this issues. Like at first it said to run ecs fargate and i had to make sure its free, turnes out it isnt free so i moved to ec2. i had to make sure everything is free before using it.
3. some bugs that took time to find out about:
    - container was exiting - i had to correct Werkzeug library version to be in the compatible version to flask. 
    - port mapping wasn't good - forgot to switch network mode to bridge for easy port mapping between the host and container.


## part 2 - Automate deployment with Terraform and Github Actions
after completing all stages in this part, you can push a new commit to remote and see the magic happens.
after github actions completes you can grab the load balancer's dns and run it on your browser. you may need to wait a couple more minutes for changes to take affect.
### steps of deployment:

#### Terraform:
explanation about all terraform files:
- variables.tf : some variables that are being used in other resources.
- vpc.tf: creating all vpc, subnets etc.. and networking configurations.
- ec2.tf: creating: ec2 instance autoscaling for ecs and apllication load balancer.
**keep in mind that in resource: aws_launch_template you need to give it a pre created key pair!**
- ecs.sh: The “ecs.sh” file contains a command to create an environment variable in “/etc/ecs/ecs.config” file on each EC2 instance that will be created. Without setting this, the ECS service will not be able to deploy and run containers on our EC2 instance.
- main.tf: creating the ECS itself including: cluster, capacity provider, task definition, service etc...
- provider.tf: setting providers with versions
- backend.tf: setting terraform backend inside an s3 bucket that were pre-created. I chose to create this s3 bucket as part as github actions for full automation but the normal procedure will be to create it manually before.

#### challenges faced and how they were resolved:
1. I worked in different regions. and every region has its own ami id's so I got errors about not finding the write image. had to run this command to get good ami id: `aws ssm get-parameters --names /aws/service/ecs/optimized-ami/amazon-linux-2/recommended --region us-west-2 --query Parameters[0].Value`
2. setting up wrong port number by accident. correcting


#### Github Actions:
- First head over to github secrets and add those 3 secrets: AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, DOCKERHUB_TOKEN. We use them when setting environment vaiables at the start of the actions file.

- create .github/workflows/main.tf actions file. the main steps are:
    1. checkout the repository code (fetches the latest version of the repository code)
    2. prepare the environment for Terraform (configure the environment for Terraform. This includes installing the correct Terraform version and setting up necessary variables)
    3. Logs into Dockerhub using the provided credentials and token.
    4. Builds a Docker image from the 'app' directory and pushes it to the specified Dockerhub repository.
    5. (Because I wanted everything to be automatic) I included the creation/Ensuring of Terraform backend s3 bucket in aws before using Terraform. This step is totaly optionl and might not be that efficient. Normally you would prefer manually create the s3 bucket because its only one time.
    6. step 6 is creating s3 bucket only if its not found in step 5.
    7. Terraform init
    8. Terraform apply
    9. Forces a new deployment of the ECS service in the specified cluster, using AWS CLI.

#### challenges faced and how they were resolved:
1. Trying to push to dockerhub without auth. Didn’t work. Adding dockerhub token as secret.
2. Deciding wether to add s3 bucket creation for terraform backend in the actions file to make everything automated or include a prerequisite. I know its not very efficient for the long term but decided to include it for fun as optional.
.
