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

### 1.4 Build & Push the Docker Image
Authenticate Docker Login with AWS:
1. Run aws ecr get-login --no-include-email --region [region]. Example: aws ecr get-login --no-include-email --region us-west-2 If you have never used the AWS CLI before, you may need to configure your credentials.
2. You're going to get a massive output starting with docker login -u AWS -p ... Copy this entire output, paste, and run it in the terminal.
3. You should see Login Succeeded.

Build the Image: In the terminal, run docker build -t api . NOTE, the . is important here.
Tag the Image: After the build completes, tag the image so you can push it to the repository: docker tag api:latest [account-id].dkr.ecr.[region].amazonaws.com/api:latest
⚐ Pro tip: the :v1 represents the image build version. Every time you build the image, you should increment this version number. If you were using a script, you could use an automated number, such as a time stamp to tag the image. This is a best practice that allows you to easily revert to a previous container image build in the future.

Push the image to ECR: Run docker push to push your image to ECR: docker push [account-id].dkr.ecr.[region].amazonaws.com/api:latest

## Part 2: Deploy the Monolith
- 2.1 Launch an ECS Cluster using AWS CloudFormation
- 2.2 Check your Cluster is Running
- 2.3 Write a Task Definition
- 2.4 Configure the Application Load Balancer: Target Group
- 2.5 Configure the Application Load Balancer: Listener
- 2.6 Deploy the Monolith as a Service
- 2.7 Test your Monolith

## Part 3: Break the Monolith
- 3.1 Provision the ECR Repositories
- 3.2 Authenticate Docker with AWS (optional)
- 3.3 Build and Push Images for Each Service

## Part 4: Deploy Microservices
- 4.1 Write Task Definitions for your Services
- 4.2 Configure the Application Load Balancer: Target Groups
- 4.3 Configure Listernet Rules
- 4.4 Deploy your Microservices
- 4.5 Switch Over Traffic to your Microservices
- 4.5 Validate you Deployment

## Part 5: Clean Up
- 5.1 Turn of Services
- 5.2 Delete Listerners
- 5.3 Delete Target Groups
- 5.4 Rollback your CloudFormation Stack
- 5.5 Deregister Amazon ECR Repositories
