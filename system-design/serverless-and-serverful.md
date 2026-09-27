# Serverless vs. Serverful Architectures

## Serverful Architecture

- You are responsible for provisioning and maintaining the underlying compute resources.
- Key responsibilities include:
  - Operating system patching and maintenance
  - Manual or auto-scaling configurations
  - Managing load balancers
  - Forecasting capacity and reserving instances
- Common Platforms:
  - Amazon EC2
  - DigitalOcean Droplets

## Serverless Architecture

- The underlying infrastructure is entirely abstracted away and managed by the cloud vendor.
- Developers only focus on writing code, and the platform scales it automatically based on events.
- Common Platforms:
  - AWS Lambda
  - Azure Functions
  - Google Cloud Functions
