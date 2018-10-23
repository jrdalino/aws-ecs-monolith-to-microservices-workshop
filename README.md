# Break a Monolith Application into Microservices with Amazon Elastic Container Service, Docker, and Amazon EC2

## Part 1: Containerized the Monolith
### 1.1 Get Setup
1. Have an AWS Account
2. Install Docker
3. Install AWS CLI
4. Have a Text Editor

### 1.2 Download & Open the Project
Download Code from Github
```
$ git clone https://github.com/awslabs/amazon-ecs-nodejs-microservices
```
Open the Project Files using Visual Studio

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
