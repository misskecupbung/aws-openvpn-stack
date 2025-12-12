# AWS OpenVPN Stack

CloudFormation template that deploys a VPC with an EC2 instance running OpenVPN. The client config (`.ovpn`) is automatically uploaded to S3.

Uses [angristan/openvpn-install](https://github.com/angristan/openvpn-install) for headless OpenVPN setup.

## Prerequisites

- AWS CLI configured
- Globally unique S3 bucket name

## Deploy

### 1. Set Environment Variables

```bash
export AWS_REGION=ap-southeast-1
export STACK_NAME=openvpn
export KEY_PAIR=openvpn-key
export BUCKET_NAME=your-unique-bucket-name
export EMAIL=your-email@example.com
```

### 2. Create Key Pair

```bash
aws ec2 create-key-pair \
  --key-name $KEY_PAIR \
  --query 'KeyMaterial' \
  --output text \
  --region $AWS_REGION > ${KEY_PAIR}.pem

chmod 400 ${KEY_PAIR}.pem
```

### 3. Create Stack

```bash
aws cloudformation create-stack \
  --stack-name $STACK_NAME \
  --template-body file://openvpn-cstack.yaml \
  --capabilities CAPABILITY_IAM \
  --region $AWS_REGION \
  --parameters \
    ParameterKey=KeyPairName,ParameterValue=$KEY_PAIR \
    ParameterKey=BucketName,ParameterValue=$BUCKET_NAME \
    ParameterKey=NotificationEmail,ParameterValue=$EMAIL
```

> **Note:** You'll receive a subscription confirmation email. Click the link to confirm before notifications can be sent.

### 4. Get Client Config

Wait ~5 minutes for setup to complete, then:

```bash
aws s3 cp s3://$BUCKET_NAME/client1.ovpn . --region $AWS_REGION
```

Import `client1.ovpn` into your OpenVPN client.

## Clean Up

```bash
aws cloudformation delete-stack --stack-name $STACK_NAME --region $AWS_REGION
aws s3 rb s3://$BUCKET_NAME --force
aws ec2 delete-key-pair --key-name $KEY_PAIR --region $AWS_REGION
rm -f ${KEY_PAIR}.pem
```

## Security Note

Set `AllowedClientCidr` to your IP (e.g., `1.2.3.4/32`) instead of the default `0.0.0.0/0` for production use.
