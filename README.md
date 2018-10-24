# Break a Monolith Application into Microservices with Amazon Elastic Container Service, Docker, and Amazon EC2

In this demo, I will show you how to use Elastic Container Services to break
a monolithic application into micro services.

We will start out by running a monolith ECS and then we will deploy new microservices
side by side with the Monolith and finally we will divert traffic to the microservices
with zero downtime.

To start what is a monolith vs microservices and why do we want to migrate from one to another?

Choosing monolith vs micrse

Many companies go through this process, they realize

Shift must be handled carefully specially

To demonstrate.. let's look

small Rest API for a forum

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

## Part 1: Containerized the Monolith
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

## Part 2: Deploy the Monolith
###  2.1 Launch an ECS Cluster using AWS CloudFormation
First, you will create a an Amazon ECS cluster, deployed behind an Application Load Balancer.

1. Navigate to the AWS CloudFormation console.
2. Select Create Stack.
3. Select 'Upload a template to Amazon S3' and choose the ecs.yml file from the GitHub project at amazon-ecs-nodejs-microservice/2-containerized/infrastructure/ecs.yml Select Next.
4. For stack name, enter BreakTheMonolith-Demo. Keep the other parameter values the same:
- Desired Capacity = 2
- InstanceType = t2.micro
- MaxSize = 2
5. Select Next.
6. It is not necessary to modify any options on this page. Select Next.
7. Check the box at the bottom of the next page and select Create. You will see your stack with the orange CREATE_IN_PROGRESS. You can select the refresh button at the top right of the screen to check on the progress. This process typically takes under 5 minutes.

OR

You can also use the AWS CLI to deploy AWS CloudFormation Stacks. Just add in your region to this code and run in the terminal from the folder amazon-ecs-nodejs-microservices/3-microservices on your computer.

```
$ aws cloudformation deploy \
   --template-file infrastructure/ecs.yml \
   --region <region> \
   --stack-name Nodejs-Microservices \
   --capabilities CAPABILITY_NAMED_IAM
```

### 2.2 Check your Cluster is Running
- Navigate to the Amazon ECS console. Your cluster should appear in the list.
- Clicking into the cluster, select the 'Tasks' tab, no tasks will be running. 
- Select the 'ECS Instances' tab, you will see the two EC2 Instances the AWS CloudFormation template created.

### 2.3 Write a Task Definition
The task definition tells Amazon ECS how to deploy your application containers across the cluster.
1. Navigate to the 'Task Definitions' menu on the left side of the Amazon ECS console.
2. Select Create new Task Definition.
3. Task Definition Name = api.
4. Select Add Container.
5. Specify the following parameters.
- If a parameter is not defined, leave it blank or with the default settings: Container name = api image = [account-id].dkr.ecr.[region].amazonaws.com/api:v1 (this is the URL of your ECR repository image from the previous step).
- Be sure the tag :v1 matches the value you used in module 1 to tag and push the image. Memory = Hard limit: 256 Port mappings = Host port:0, Container port:3000 CPU units = 256
6. Select Add.
7. Select Create.
8. Your Task Definition will now show up in the console.

### 2.4 Configure the Application Load Balancer: Target Group
The Application Load Balancer (ALB) lets your service accept incoming traffic. The ALB automatically routes traffic to container instances running on your cluster using them as a target group.

Check your VPC Name: If this is not your first time using this AWS account, you may have multiple VPCs. It is important to configure your Target Group with the correct VPC.

- Navigate to the Load Balancer section of the EC2 Console.
- You should see a Load Balancer already exists named demo.
- Select the checkbox to see the Load Balancer details.
- Note the value for the VPC attribute on the details page.

Configure the ALB Target Group

1. Navigate to the Target Group section of the EC2 Console.
2. Select Create target group.
3. Configure the Target Group (do not modify defaults if they are not specified here): 
- Name = api 
- Protocol = HTTP 
- Port = 80 
- VPC = select the VPC that matches your Load Balancer from the previous step. This is most likely NOT your default VPC. 
- Advanced health check settings: Healthy threshold = 2 Unhealthy threshold = 2 Timeout = 5 Interval = 6.
4. Select Create.

### 2.5 Configure the Application Load Balancer: Listener
The listener checks for incoming connection requests to your ALB.

Add a Listener to the ALB

1. Navigate to the Load Balancer section of the EC2 Console.
2. You should see a Load Balancer already exists named demo.
3. Select the checkbox to see the Load Balancer details.
4. Select the Listeners tab.
5. Select Create Listener:
- Protocol = HTTP
- Port = 80
- Default target group = api
6. Click Create.

### 2.6 Deploy the Monolith as a Service
Now, you will deploy the monolith as a service onto the cluster.

1. Navigate to the 'Clusters' menu on the left side of the Amazon ECS console.
2. Select your cluster: BreakTheMonolith-Demo-ECSCluster.
3. Under the services tab, select Create.
4. Configure the service (do not modify any default values): Service name = api Number of tasks = 1
5. Select Configure ELB:
- ELB Type = Application Load Balancer.
- For IAM role, select BreakTheMonolith-Demo-ECSServiceRole.
- Select your Load Balancer ELB name = demo.
- Select Add to ELB.
6. Add your service to the target group:
- Listener port = 80:HTTP
- Target group name = select your group: api.
7. Select Save.
8. Select Create Service.
9. Select View Service

### 2.7 Test your Monolith
Validate your deployment by checking if the service is available from the internet and pinging it.

To Find your Service URL:

Navigate to the Load Balancers section of the EC2 Console.
Select your load balancer demo.
Copy and paste the value for DNS name into your browser.
You should see a message 'Ready to receive requests'.

See Each Part of the Service: The node.js application routes traffic to each worker based on the URL. To see a worker, simply add the worker name api/[worker-name] to the end of your DNS Name like this:

http://[DNS name]/api/users
http://[DNS name]/api/threads
http://[DNS name]/api/posts
You can also add a record number at the end of the URL to drill down to a particular record. Like this: http://[DNS name]/api/posts/1 or http://[DNS name]/api/users/2

## Part 3: Break the Monolith
### 3.1 Provision the ECR Repositories
In the previous two steps, you deployed your application as a monolith using a single service and a single container image repository. To deploy the application as three microservices, you will need to provision three repositories (one for each service) in Amazon ECR.

Our three services are:
- users
- threads
- posts

Create the repository:

1. Navigate to the Amazon ECR Console.
2. Select Create Repository
3. Repository name:
- users
- threads
- posts

Record the repositories information: [account-id].dkr.ecr.[region].amazonaws.com/[service-name]
♻ Repeat these steps for each microservice.

You should now have four repositories in Amazon ECR.

### 3.2 Authenticate Docker with AWS (optional)
You may skip this step if you recently completed Module 1 of this workshop.

Run aws ecr get-login --no-include-email --region [region]
Example: aws ecr get-login --no-include-email --region ap-southeast-1
You are going to get a massive output starting with docker login -u AWS -p ... Copy this entire output, paste, and run it in the terminal.

You should see Login Succeeded

### 3.3 Build and Push Images for Each Service
In the project folder amazon-ecs-nodejs-microservices/3-microservices/services, you will have folders with files for each service. Notice how each microservice is essentially a clone of the previous monolithic service.

You can see how each service is now specialized by comparing the file db.json in each service and in the monolithic api service. Previously posts, threads, and users were all stored in a single database file. Now, each is stored in the database file for its respective service.

Open your terminal, and set your path to the 3-microservices/services section of the GitHub code. ~/amazon-ecs-nodejs-microservices/3-microservices/services

Build and Tag Each Image

- In the terminal, run docker build -t [service-name] ./[service-name] Example: docker build -t posts ./posts
- After the build completes, tag the image so you can push it to the repository: docker tag [service-name]:latest [account-id].dkr.ecr.[region].amazonaws.com/[service-name]:latest example: docker tag posts:latest [account-id].dkr.ecr.ap-southeast-1.amazonaws.com/posts:latest
- Run docker push to push your image to ECR: docker push [account-id].dkr.ecr.[region].amazonaws.com/[service-name]:latest

If you navigate to your ECR repository, you should see your images tagged with latest. 

♻ Repeat these steps for each microservice image.  

⚐ NOTE: Be sure to build and tag all three images.

## Part 4: Deploy Microservices
### 4.1 Write Task Definitions for your Services
### 4.2 Configure the Application Load Balancer: Target Groups
### 4.3 Configure Listernet Rules
### 4.4 Deploy your Microservices
### 4.5 Switch Over Traffic to your Microservices
### 4.5 Validate you Deployment

## Part 5: Clean Up
### 5.1 Turn of Services
### 5.2 Delete Listerners
### 5.3 Delete Target Groups
### 5.4 Rollback your CloudFormation Stack
### 5.5 Deregister Amazon ECR Repositories
