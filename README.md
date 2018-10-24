# Break a Monolith Application into Microservices with Amazon Elastic Container Service, Docker, and Amazon EC2

In this demo, I will show you how to use Elastic Container Services to break
a monolithic application into microservices.

We will start out by running a monolithic appplication using ECS and then we will deploy new microservices
side by side with the Monolith and finally we will divert traffic over to our microservices
with zero downtime.

Let's take a look at our server.js file
```js
const app = require('koa')();
const router = require('koa-router')();
const db = require('./db.json');

// Log requests
app.use(function *(next){
  const start = new Date;
  yield next;
  const ms = new Date - start;
  console.log('%s %s - %s', this.method, this.url, ms);
});

router.get('/api/users', function *(next) {
  this.body = db.users;
});

router.get('/api/users/:userId', function *(next) {
  const id = parseInt(this.params.userId);
  this.body = db.users.find((user) => user.id == id);
});

router.get('/api/threads', function *() {
  this.body = db.threads;
});

router.get('/api/threads/:threadId', function *() {
  const id = parseInt(this.params.threadId);
  this.body = db.threads.find((thread) => thread.id == id);
});

router.get('/api/posts/in-thread/:threadId', function *() {
  const id = parseInt(this.params.threadId);
  this.body = db.posts.filter((post) => post.thread == id);
});

router.get('/api/posts/by-user/:userId', function *() {
  const id = parseInt(this.params.userId);
  this.body = db.posts.filter((post) => post.user == id);
});

router.get('/api/', function *() {
  this.body = "API ready to receive requests";
});

router.get('/', function *() {
  this.body = "Ready to receive requests";
});

app.use(router.routes());
```

## Part 1: Containerize the Monolith
### 1.1 Get Setup
1. Install Docker
```
$ docker --version
Docker version 18.06.1-ce, build e68fc7a
```
2. Install AWS CLI
```
$ aws --version
aws-cli/1.16.8 Python/2.7.10 Darwin/16.7.0 botocore/1.11.8
```
3. Text Editor
4. AWS Account

### 1.2 Download & Open the Project
Download Code from Github
```
$ git clone https://github.com/awslabs/amazon-ecs-nodejs-microservices
```
Open the Project Files using Visual Studio

Run the Monolith locally

So first, let's verify this application will run on my local machine
```
$ cd ~/amazon-ecs-nodejs-microservices/1-no-container
$ npm install koa
$ npm install koa-router
$ npm start

> @ start /Users/jrdalino/Projects/amazon-ecs-nodejs-microservices/1-no-container
> node server.js

Worker started
```
To test
```
$ curl localhost:3000/api/users
$ curl localhost:3000/api/users/1
$ curl localhost:3000/api/threads
$ curl localhost:3000/api/threads/1
$ curl localhost:3000/api/posts/by-user/1
```

### 1.3 Provision a Repository
Create the Repository
- Navigate to the Amazon Elastic Container Registry (Amazon ECR).
- Select Create Repository.
- Name your repository. For this step, keep it simple and call this repository api.

Record Repository Information
- After you hit next, you should get a message that looks like this:
- The repository address follows a simple format: [account-id].dkr.ecr.[region].amazonaws.com/[repo-name].

Mine looks like this:
```
505265941169.dkr.ecr.ap-southeast-1.amazonaws.com/api
```

### 1.4 Build & Push the Docker Image
Open your terminal, and set your path to the 2-containerized/services/api section of the GitHub code in the directory you have it cloned or downloaded it into: ~/amazon-ecs-nodejs-microservices/2-containerized/services/api.
```
$ /Users/jrdalino/Projects/amazon-ecs-nodejs-microservices/2-containerized/services/api

$ ls
Dockerfile	db.json		package.json	rule.json	server.js
```
Authenticate Docker Login with AWS:

1. Run aws ecr get-login --no-include-email --region [region]. Example: aws ecr get-login --no-include-email --region ap-southeast-1 If you have never used the AWS CLI before, you may need to configure your credentials.
```
$(aws ecr get-login --no-include-email --region ap-southeast-1)
```

2. You're going to get a massive output starting with docker login -u AWS -p ... Copy this entire output, paste, and run it in the terminal.
```
docker login -u AWS -p eyJwY...== https://505265941169.dkr.ecr.ap-southeast-1.amazonaws.com
```

You should see Login Succeeded.

4. Build the Image: In the terminal, run docker build -t api . NOTE, the . is important here.
```
docker build -t api .
```

5. Tag the Image: After the build completes, tag the image so you can push it to the repository: docker tag api:latest [account-id].dkr.ecr.[region].amazonaws.com/api:latest
```
docker tag api:latest 505265941169.dkr.ecr.ap-southeast-1.amazonaws.com/api:latest
```

6. Push the image to ECR: Run docker push to push your image to ECR: 
docker push [account-id].dkr.ecr.[region].amazonaws.com/api:latest
```
docker push 505265941169.dkr.ecr.ap-southeast-1.amazonaws.com/api:latest
```

Now that my image is stored in the respository, it can be pulled back down wherever i need to run it, including Amazon Elastic Container Service. 

## Part 2: Deploy the Monolith
###  2.1 Launch an ECS Cluster using AWS CloudFormation
In order to run my my container in ECS, i need a little bit of setup though. I prepared a CloudFormation Template that creates a fresh VPC, a cluster of Docker Hosts and an Application Load Balancer

First, you will create a an Amazon ECS cluster, deployed behind an Application Load Balancer. Let's use AWS CLI to deploy AWS CloudFormation Stack. Just add in your region to this code and run in the terminal from the folder amazon-ecs-nodejs-microservices/3-microservices on your computer.
```
$ cd /Users/jrdalino/Projects/amazon-ecs-nodejs-microservices/3-microservices
```

Let's deploy our cloudformation template. Make sure you use the correct region.
```
$ aws cloudformation deploy \
   --template-file infrastructure/ecs.yml \
   --region ap-southeast-1 \
   --stack-name Nodejs-Microservices \
   --capabilities CAPABILITY_NAMED_IAM
```

Output
```
Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - Nodejs-Microservices
```

### 2.2 Check your Cluster is Running
- Navigate to the Amazon ECS console. Your cluster should appear in the list.
- Clicking into the cluster, select the 'Tasks' tab, no tasks will be running. 
- Select the 'ECS Instances' tab, you will see the two EC2 Instances the AWS CloudFormation template created.

### 2.3 Write a Task Definition
The task definition tells Amazon ECS how to deploy your application containers across the cluster.
1. Navigate to the 'Task Definitions' menu on the left side of the Amazon ECS console.
2. Select Create new Task Definition and Select Launch type compatibility = EC2
3. Task Definition Name = api.
4. Select Add Container.
5. Specify the following parameters. If a parameter is not defined, leave it blank or with the default settings: 
- Container name = api 
- Image = 505265941169.dkr.ecr.ap-southeast-1.amazonaws.com/api:latest
- Memory = Hard limit: 256 
- Port mappings = Host port:0, Container port:3000 
- CPU units = 256
6. Select Add.
7. Select Create.
8. Your Task Definition will now show up in the console.

### 2.4 Configure the Application Load Balancer: Target Group
The Application Load Balancer (ALB) lets your service accept incoming traffic. The ALB automatically routes traffic to container instances running on your cluster using them as a target group.

**Check your VPC Name:** If this is not your first time using this AWS account, you may have multiple VPCs. It is important to configure your Target Group with the correct VPC.

- Navigate to the Load Balancer section of the EC2 Console.
- You should see a Load Balancer already exists named demo.
- Select the checkbox to see the Load Balancer details.
- Note the value for the VPC attribute on the details page.

**Configure the ALB Target Group**

1. Navigate to the Target Group section of the EC2 Console.
2. Select Create target group.
3. Configure the Target Group (do not modify defaults if they are not specified here): 
- Name = api 
- Protocol = HTTP 
- Port = 80 
- VPC = select the VPC that matches your Load Balancer from the previous step. This is most likely NOT your default VPC. 
Advanced health check settings: 
- Healthy threshold = 2 
- Unhealthy threshold = 2 
- Timeout = 5 
- Interval = 6
4. Select Create.

### 2.5 Configure the Application Load Balancer: Listener
The listener checks for incoming connection requests to your ALB.

**Add a Listener to the ALB**

1. Navigate to the Load Balancer section of the EC2 Console.
2. You should see a Load Balancer already exists named demo.
3. Select the checkbox to see the Load Balancer details.
4. Select the Listeners tab.
5. Select Create Listener:
- Protocol = HTTP
- Port = 80
- Default target group = Forward to: api
6. Click Create.

### 2.6 Deploy the Monolith as a Service
Now that the Task Definition has been created, I can launch this Task Definition as a Service. This is basically a way to tell ECS to run one or more copies of this container at all times and connect the running containers to a load balancer.

Let's deploy the monolith as a service onto the cluster.

1. Navigate to the 'Clusters' menu on the left side of the Amazon ECS console.
2. Select your cluster: BreakTheMonolith-Demo-ECSCluster.
3. Under the services tab, select Create.
4. Configure the service (do not modify any default values): 
- Service name = api 
- Number of tasks = 2 (I specify how many copies of your container I want to run)
5. Click on Next Step
6. Under Load Balancing
- Load Balancer Type = Application Load Balancer.
- For IAM role, select BreakTheMonolith-Demo-ECSServiceRole.
- Load Balancer name = demo should already be selected.
- Select Add to Load Balancer.
7. Add your service to the target group:
- Listener port = 80:HTTP
- Target group name = api.
8. Select Save.
9. Select Create Service.
10. Select View Service.

### 2.7 Test your Monolith
Validate your deployment by checking if the service is available from the internet and pinging it.

**To Find your Service URL:**

Navigate to the Load Balancers section of the EC2 Console.
Select your load balancer demo.
Copy and paste the value for DNS name into your browser.
You should see a message 'Ready to receive requests'.

**See Each Part of the Service:** The node.js application routes traffic to each worker based on the URL. To see a worker, simply add the worker name api/[worker-name] to the end of your DNS Name like this:

- http://[DNS name]/api/users
- http://[DNS name]/api/threads
- http://[DNS name]/api/posts

You can also add a record number at the end of the URL to drill down to a particular record. Like this: http://[DNS name]/api/posts/1 or http://[DNS name]/api/users/2

So at this point I have my class-style monolithic application up and running in Elastic Container Service but i'm not done yet. My goal is to take this monolith and split it up into microservices. If we look at the base code for the monolith, you can see the HTTP routes relating to users, threads and posts. A sensible way to split this application up into microservices would be to create three microservices, one for users, one for threads and one for posts.

So what i'm gonna do is i'm going to repeat the steps that I did to deplot the monolithic application but instead I'll build and deploy three different microservices that will run in parallel with the monolith.

## Part 3: Break the Monolith
### 3.1 Provision the ECR Repositories
In the previous two steps, you deployed your application as a monolith using a single service and a single container image repository. To deploy the application as three microservices, you will need to provision three repositories (one for each service) in Amazon ECR. Our three services are users, threads and posts.

**Create the repository:**

1. Navigate to the Amazon ECR Console.
2. Select Create Repository
3. Repository name:
- users
- threads
- posts

Record the repositories information: [account-id].dkr.ecr.[region].amazonaws.com/[service-name]
```
505265941169.dkr.ecr.ap-southeast-1.amazonaws.com/users
505265941169.dkr.ecr.ap-southeast-1.amazonaws.com/threads
505265941169.dkr.ecr.ap-southeast-1.amazonaws.com/posts
```

You should now have **four** repositories in Amazon ECR.

### 3.2 Re-authenticate Docker with AWS if needed

### 3.3 Build and Push Images for Each Service
In the project folder amazon-ecs-nodejs-microservices/3-microservices/services, you will have folders with files for each service. Notice how each microservice is essentially a clone of the previous monolithic service.

You can see how each service is now specialized by comparing the file db.json in each service and in the monolithic api service. Previously posts, threads, and users were all stored in a single database file. Now, each is stored in the database file for its respective service.

Open your terminal, and set your path to the 3-microservices/services section of the GitHub code. ~/amazon-ecs-nodejs-microservices/3-microservices/services

Build, Tag and Push Each Image

users
```
$ docker build -t users users/.
$ docker tag users:latest 505265941169.dkr.ecr.ap-southeast-1.amazonaws.com/users:latest
$ docker push 505265941169.dkr.ecr.ap-southeast-1.amazonaws.com/users:latest
```
threads
```
$ docker build -t threads threads/.
$ docker tag threads:latest 505265941169.dkr.ecr.ap-southeast-1.amazonaws.com/threads:latest
$ docker push 505265941169.dkr.ecr.ap-southeast-1.amazonaws.com/threads:latest
```
posts
```
$ docker build -t posts posts/.
$ docker tag posts:latest 505265941169.dkr.ecr.ap-southeast-1.amazonaws.com/posts:latest
$ docker push 505265941169.dkr.ecr.ap-southeast-1.amazonaws.com/posts:latest
```

## Part 4: Deploy Microservices
### 4.1 Write Task Definitions for your Services
You will deploy three new microservices onto the same cluster you have running from Module 2. Like in Module 2, you will write Task Definitions for each service.

⚐ **NOTE:** It is possible to add multiple containers to a task definition - so feasibly you could run all three microservices as different containers on a single service. This however, would still be monolithic as every container would need to scale linearly with the service. Your goal is to have three independent services and each service requires its own task definition running a container with the image for that respective service.

You can either these Task Definitions in the console UI, or speed things up by writing them as JSON. To write the task definition as a JSON file, select Configure via JSON at the bottom of the new Task Definition screen.

The parameters for the task definition are:

- Name = [service-name] 
- Image = [service ECR repo URL]:latest 
- cpu = 256 
- memory = 256 
- Container Port = 3000 
- Host Post = 0

Or with JSON:
```json
{
    "containerDefinitions": [
        {
            "name": "[service-name]",
            "image": "[account-id].dkr.ecr.ap-southeast-1.amazonaws.com/[service-name]:[tag]",
            "memoryReservation": "256",
            "cpu": "256",
            "essential": true,
            "portMappings": [
                {
                    "hostPort": "0",
                    "containerPort": "3000",
                    "protocol": "tcp"
                }
            ]
        }
    ],
    "volumes": [],
    "networkMode": "bridge",
    "placementConstraints": [],
    "family": "[service-name]"
}
```

♻ Repeat this process to create a task definition for each service:
- posts
- threads
- users

### 4.2 Configure the Application Load Balancer: Target Groups
Like in Module 2, you will be configuring target groups for each of your services. Target groups allow traffic to correctly reach each service.

**Check your VPC Name:** The AWS CloudFormation stack has its own VPC, which is most likely not your default VPC. It is important to configure your Target Groups with the correct VPC.

- Navigate to the Load Balancer section of the EC2 Console.
- You should see a Load Balancer already exists named demo.
- Select the checkbox to see the Load Balancer details.
- Note the value for the VPC attribute on the details page.
 
**Configure the Target Groups**

1. Navigate to the Target Group section of the EC2 Console.
2. Select Create target group.
3. Configure the Target Group (do not modify defaults if they are not specified here): 
- Name = [service-name] 
- Protocol = HTTP 
- Port = 80 VPC = select the VPC that matches your Load Balancer from the previous step. 
Advanced health check settings: 
- Healthy threshold = 2 
- Unhealthy threshold = 2 
- Timeout = 5 
- Interval = 6
4. Select Create.
 
♻ Repeat this process to create a target group for each service:
- posts
- threads
- users
 
**Finally, Create a Fourth Target Group**
- drop-traffic

This target group is a 'dummy' target. You will use it to keep traffic from reaching your monolith after your microservices are fully running. You should have 5 target groups total in your table.

### 4.3 Configure Listernet Rules
The listener checks for incoming connection requests to your ALB in order to route traffic appropriately.

Right now, all four of your services (monolith and your three microservices) are running behind the same load balancer. To make the transition from monolith to microservices, you will start routing traffic to your microservices and stop routing traffic to your monolith.

**Open your a listener**

- Navigate to the Load Balancer section of the EC2 Console.
- You should see a Load Balancer already exists named demo.
- Select the checkbox to see the Load Balancer details.
- Select the Listeners tab.
 
**Update Listener Rules**

1. Select View/edit rules > for the listener.
2. Select the + and insert rule.
3. The rule criteria are:
- IF Path = /api/[service-name]* THEN Forward to [service-name]
- For example: Path = /api/posts* forward to posts
4. Create four new rules, one to maintain traffic to the monolith, and one for each service. You will have a total of five rules, including the default. Ensure you add your rules in this order:
- api: /api* forwards to api
- users: /api/users* forwards to users
- threads: /api/threads* forwards to threads
- posts: /api/posts* forwards to posts
5. Select the back arrow at the top left of the page to return to the load balancer console.

### 4.4 Deploy your Microservices
Now, you will deploy your three services onto your cluster. Repeat these steps for each of your three services:

1. Navigate to the 'Clusters' menu on the left side of the Amazon ECS console.
2.  Select your cluster: BreakTheMonolith-Demo-ECSCluster.
3.  Under the services tab, select Create.
4. Configure the service (do not modify any default values) Task definition = select the highest value for X: [service-name]:X (X should = 1 for most cases) Service name = [service-name] Number of tasks = 1
5.  Select Configure ELB
- ELB Type = Application Load Balancer
- For IAM role, select BreakTheMonolith-Demo-ECSServiceRole
- Select your Load Balancer demo
- Select Add to ELB
6. Add your service to the target group:
- Listener port = 80:HTTP
- Target group name = select your group: [service-name]
7. Select Save.
8. Select Create Service.
9. Select View Service.

It should only take a few seconds for all your services to start. Double check that all services and tasks are running and healthy before you proceed.

### 4.5 Switch Over Traffic to your Microservices
Right now, your microservices are running, but all traffic is still flowing to your monolith service.

**Update Listener Rulers to Re-Route Traffic to the Microservices:**

- Navigate to the Load Balancer section of the EC2 Console
- Select View/edit rules > for the listener on the demo load balancer.
- Delete the first rule (/api* forwards to api).
- Update the default rule to forward to drop-traffic.

**Turn off the Monolith:** Now, traffic is flowing to your microservices and you can spin down the Monolith service.

- Navigate back to your Amazon ECS cluster BreakTheMonolith-Demo-ECSCluster.
- Select the api service and then Update.
- Change Number of Tasks to 0.
- Select Update Service.
 
Amazon ECS will now drain any connections from containers the service has deployed on the cluster then stop the containers. If you refresh the Deployments or Tasks lists after about 30 seconds, you will see that the number of tasks will drop to 0. The service is still active, so if you needed to roll back for any reason, you could simply update it to deploy more tasks.

- Select the api service and then Delete and confirm delete.

**You have now fully transitioned your node.js from the monolith to microservices, without any downtime!**


### 4.5 Validate you Deployment

**Find your service URL:** This is the same URL that you used in Module 2 of this tutorial.

- Navigate to the Load Balancers section of the EC2 Console.
- Select your load balancer demo-microservices.
- Copy and paste the value for DNS name into your browser.
- You should see a message 'Ready to receive requests'.
 
**See the Values for each Microservice:** Your ALB routes traffic based on the request URL. To see each service, simply add the service name to the end of your DNS Name like this:

- http://[DNS name]/api/users
- http://[DNS name]/api/threads
- http://[DNS name]/api/posts

⚐ **NOTE:** These URLs perform exactly the same as when the monolith is deployed. This is very important because any APIs or consumers that would expect to connect to this app will not be affected by the changes you made. Going from monolith to microservices required no changes to other parts of your infrastructure.

You can also use tools such as Postman for testing your APIs.

## Part 5: Clean Up
### 5.1 Turn of Services
### 5.2 Delete Listerners
### 5.3 Delete Target Groups
### 5.4 Rollback your CloudFormation Stack
### 5.5 Deregister Amazon ECR Repositories
