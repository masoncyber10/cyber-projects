# Cross Account Lambda to S3 Access
**Visual Diagram**

<img width="650" height="536" alt="image" src="https://github.com/user-attachments/assets/6b1035fe-3e08-4e76-8c27-bf1f8eee414a" />

**Project Overview**

**Objective:** 

Build a secure cross-account IAM access pattern where a Lambda function in Account A can read/write to an S3 bucket in Account B using temporary credentials.

**What I've learned:**
- IAM roles and policies
  - Trust and Permission Policy (Trust Relationship)
- STS (Security Token Service) temporary credentials
- Lambda execution
- API Gateway integration
- Cross-account access patterns

I had learned that putting credentials on Lambda code can be risky especially when testing this lab in a sandbox environment. 
STS (Security Token Service) should always be put in place and credential timer should be one hour max. During the process of
making IAM roles for Lambda and S3, I realized that there is always a set of policies in place to give the certain role, user, or group
depending on the situation limited access.

**Step 1: Create Account A (Lambda) role**
1. Go to IAM --> Roles --> Create role
2. Select Lambda as a Service
3. Name: LambdaAssumeRole
4. Create the role without adding policy

<img width="768" height="33" alt="image" src="https://github.com/user-attachments/assets/d3bda26d-b5e9-42a8-9e46-61c74eb48df4" />


**Step 2 Create Account B (S3) role**
1. Same step as Step 1
2. Name: S3AccessRole
   
<img width="157" height="32" alt="image" src="https://github.com/user-attachments/assets/83363a17-182a-46de-8353-30e109958dde" />

**Step 3: Add permission to assume Account B**
1. Add an inline policy under LambaAssumeRole
2. Include Security Token Service (STS) in the policy
3. Resource: Add the Account ID and Account B (S3AccessRole)
4. Create the policy
5. Copy the LambdaAssumeRole's ARN, which will later pasted in Account B trust policy
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::ACCOUNT-ID:role/S3AccessRole"
    }
  ]
}
```
<img width="935" height="201" alt="image" src="https://github.com/user-attachments/assets/fb407fc3-fd07-4027-9bf8-a2a1a3d3c033" />

**Step 4: Create S3 Bucket**
1. Go to S3 --> Create bucket
2. Name: my-test-bucket-cloud#
   - The bucket will be used in the Lambda code


**Step 5: Create the trust policy (S3AccessRole)**
1. Go to the Trust Relationship and edit the policy
2. Replace the default policy and implement the LambdaAssumeRole's ARN to establish trust Account A

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT-ID:role/LambdaAssumeRole"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```
**Step 6: Adding S3 Permissions**
1. Under permissions in S3AccessRole, add an inline policy
   - The permission policy needs to allow S3 actions onto the bucket
   - Actions needed: _s3:GetObject_, _s3:PutObject_, _s3:ListBucket_'
   - Resources: bucket ARN and bucket content (bucket/* pattern) --> Only edit the read/write actions
   - Copy the S3AccessRole's ARN into a notepad (Lambda code)

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::test-bucket-cloud",
        "arn:aws:s3:::test-bucket-cloud/*"
      ]
    }
  ]
}
```

**Step 7: Lambda Function (did not write the code but needed to work on my coding implementation and analysis)**
1. Create a function
2. Name: CrossAccountS3Test
3. Runtime: Python: 3.12
4. Execution Role: LambdaAccessRole

**Code**
```
import boto3
import json

def lambda_handler(event, context):
    # Create an STS client
    sts_client = boto3.client('sts')

    # Call assume_role() to get temporary credentials for S3AccessRole
    assume_role = sts_client.assume_role(
        RoleArn="arn:aws:iam::ACCOUNT-ID:role/S3AccessRole",
        RoleSessionName="CrossAccountSession"
    )
    
    # Store the response with temporary credentials
    credentials = assume_role['Credentials']

    # Extract the three temporary credentials
    access_key = credentials['AccessKeyId']
    secret_access_key = credentials['SecretAccessKey']
    session_token = credentials['SessionToken']

    # Create S3 client using the temporary credentials
    s3_client = boto3.client('s3',
        aws_access_key_id=access_key,
        aws_secret_access_key=secret_access_key,
        aws_session_token=session_token
    )

    # Write a file to S3
    s3_client.put_object(
       Bucket='your-bucket-name',
       Key='my-test-file.txt',
       Body='Hello from Cross-Account Access' 
    )

    # Read the file back from S3
    response = s3_client.get_object(
        Bucket='your-bucket-name',
        Key='my-test-file.txt',
    )

    # Extract and decode the file content
    content = response['Body'].read().decode('utf-8')
    
    # Return success response
    return {
        'statusCode': 200,
        'body': json.dumps(f'Success! File content: {content}')
    }
```
Analysis of the Lambda code: This is a test code to see whether the Lambda and S3 bucket policies work.
