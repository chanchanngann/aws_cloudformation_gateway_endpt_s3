# EC2 in private subnet to access S3 using Gateway VPC Endpoint

## Concepts

- **What is VPC Endpoint?**

  A VPC endpoint is a virtual scalable networking component you create in a VPC and 
  use as a private entry point to supported AWS services and third-party applications. 

- **How to access S3 without entering the public internet?**

  To privately access Amazon S3, we can use either interface VPC endpoint or gateway VPC endpoint.

- **Interface VPC endpoint vs gateway VPC endpoint for S3**

  Both endpoints ensure network traffic to remain on the AWS network.

  **Interface VPC endpoint** 
    - Enable connectivity to a range of services powered by AWS PrivateLink
    - Use private IP from the VPC to access S3
    - Allow access from on-premise
    - Allow access from VPC in another AWS region using VPC peering or Transit Gateway
    - Billed

  **Gateway endpoint**
    - Use route table to route the traffic and support only S3 & Dynamo DB (No private link involved)
    - Use S3 public IP 
    - Do not allow access from on-premise
    - Allow access from only the SAME AWS region
    - FREE of charge

=> In this exercise, I tried to access S3 from an EC2 instance in a private subnet using Gateway VPC endpoint.
This route allows the private EC2 to access S3 without going through internet gateway or NAT gateway.
Also, by adding bucket policy to the destination S3 bucket, we deny all traffic except the allowed VPC endpoint. 

## Architecture

1. Set up a Bastion host in a public subnet which can access internet using internet gateway,
and another intance in a private subnet. One has to connect to the bastion host first in order to access the private instance.

2. Set up Gateway VPC endpoint for AWS S3 in the same VPC as the bastion host & private instance.

3. Add bucket policy to the destination S3 bucket to restrict access.
The policy will deny all traffic except the allowed gateway endpoint plus my source IP.
I added my IP to the bucket policy so that I can also access the bucket via console.
Otherwise, the bucket will be locked to be only accessible from the private EC2.

![architecture](/s3_endpt_architecture_v1.PNG)

## Cloudformation template

I have enclosed the cloudformation template for the above architecture.
The stack will be built accordingly by running the AWS CLI command below.
```
aws cloudformation create-stack --stack-name s3-gateway-endpoint --template-body file:///path/to/file/s3_gateway_endpoint.yaml --capabilities CAPABILITY_NAMED_IAM
```
### Issue

Permission denied when trying to connect to the private EC2 from the bastion host.
It's because I didn't have the key pair saved on bastion host.

### How to solve the issue
1. At the Bastion host, create the key pair using editor

    `nano my_keypair.pem`

2. Change the permission of the key pair

    `chmod 0400 my_keypair.pem`

## Testing access to S3

1. Connect to the bastion host:

    `ssh -i /filepath/to/my_keypair.pem ec2-user@public-ip-of-the-bastion-host`
    
    `aws s3 ls destination-bucket`

=> Result: failed to access the destination-bucket as the bucket policy denied public access

2. Connect to the private EC2 from the bastion host:

    `ssh -i my_keypair.pem ec2-user@private-ip-of-the-private-instance`

    `aws s3 ls --region ap-northeast-2`

    `aws s3 ls destination-bucket`

=> Result: SUCCESSFULLY accessed the destination-bucket

**__References__**
- https://aws.amazon.com/blogs/architecture/choosing-your-vpc-endpoint-strategy-for-amazon-s3/
- https://docs.aws.amazon.com/vpc/latest/privatelink/getting-started.html#test-vpc-endpoint
- https://jayendrapatil.com/aws-vpc-gateway-endpoints/
- https://templates.cloudonaut.io/en/v6.11.0/vpc/#vpc-endpoint-to-s3
- https://repost.aws/knowledge-center/block-s3-traffic-vpc-ip
