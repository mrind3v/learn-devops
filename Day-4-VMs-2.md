Cloud providers lets us create VMs according to our needs.

One can use the cloud provider ui (like AWS or Azure) to manually create VMs. But DevOps engineers don't do that since its manual work again and DevOps is all about efficiency. They write scripts that send request to API endpoints written by the developers of the services provided by the cloud provider to automate the process of creating VMs (in this case) and all other services.

Eg: Some ways to create EC2 instance in AWS:
1. AWS CLI
2. AWS CDK
3. AWS UI
---- the above three ways are preferred by companies deeply into AWS ecosystem for priority support---
4. Terraform (open source and not AWS specific)