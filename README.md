# Break a Monolith Application into Microservices with Amazon Elastic Container Service, Docker, and Amazon EC2

## Part 1: Containerized the Monolith
- 1.1 Get Setup
- 1.2 Downliad & Open the Project
- 1.3 Provision a Repository
- 1.4 Build & Push the Docker Image

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
