**Title**

Deploying a Load-Balanced Python Web Application with Amazon Web Services

**Description**

For many developers, the experience of deploying an application consists of finding a server to host the app, copying it over, and serving using some system like Nginx or Apache.
And that's ok!
It'll work for small applications that aren't meant to handle large loads.
However, if your product is growing or you're adding a new service to an existing, mature application, chances are you're going to need to learn how to scale your web application.
You can do it manually, of course, but Amazon Web Services provides great tools for managing and scaling your applications automatically.
In this tutorial we'll walk through how to set up those services in the AWS environment, enabling your web application to maintain reliability whether big or small.

**Outline**

- Walk through an explanation of the IAM user system and AWS resource addressing
    - Creating the user
    - Storing the credentials
- Set up an RDS PostgreSQL instance
    - Creating the database server
    - Storing the credentials
- Explanation and walk-through of setting up AWS Elastic Beanstalk through web interface
    - Choosing the runtime
    - Choosing load balancing optoions
    - Customizing the deployment process
    - Creating executable commands for deployment
- Deploying static resources to AWS CloudFront
    - Configuring CloudFront
    - Migrating staticfiles to CloudFront

**Who Should Attend?**

The target audience is web app developers looking to move from a single-server solution to a load-balanced, scalable solution for the applications that they support.

**Take Back to Work**

Attendees should take back to work a consistent, accessible work flow for deploying applications into a load-balanced cloud service.

**Topics Include**

Topics covered in this tutorial include

- Creating an AWS IAM user account for deploying and maintaining apps on AWS
- Creating an AWS RDS instance to host a Postgres database
- Configuring an AWS S3 bucket for static files
- Deploying a Python application to AWS Elastic Beanstalk
- Delivering static S3 resources with AWS CloudFront
