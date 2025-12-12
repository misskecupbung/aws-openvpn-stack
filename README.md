# AWS OpenVPN Stack

CloudFormation template that deploys a VPC with an EC2 instance running OpenVPN. The client config (`.ovpn`) is automatically uploaded to S3.

Uses [angristan/openvpn-install](https://github.com/angristan/openvpn-install) for headless OpenVPN setup.

## Resources Created

| Resource | Type | Description |
|----------|------|-------------|
| VPC | `AWS::EC2::VPC` | Virtual Private Cloud with DNS support |
| Internet Gateway | `AWS::EC2::InternetGateway` | Enables internet access for the VPC |
| Public Subnet | `AWS::EC2::Subnet` | Subnet with auto-assign public IP |
| Route Table | `AWS::EC2::RouteTable` | Routes traffic to the Internet Gateway |
| Network ACL | `AWS::EC2::NetworkAcl` | Allows SSH (22), OpenVPN (1194/UDP), and ephemeral ports |
| Security Group | `AWS::EC2::SecurityGroup` | Controls inbound/outbound traffic for EC2 |
| S3 Bucket | `AWS::S3::Bucket` | Stores the generated `.ovpn` client config |
| S3 Bucket Policy | `AWS::S3::BucketPolicy` | Allows EC2 instance to upload files |
| SNS Topic | `AWS::SNS::Topic` | Notification topic for setup completion |
| SNS Subscription | `AWS::SNS::Subscription` | Email subscription for notifications |
| IAM Role | `AWS::IAM::Role` | Grants EC2 access to S3 and SNS |
| Instance Profile | `AWS::IAM::InstanceProfile` | Attaches IAM role to EC2 |
| EC2 Instance | `AWS::EC2::Instance` | Ubuntu 24.04 server running OpenVPN |
| Elastic IP | `AWS::EC2::EIP` | Static public IP for the VPN server |

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
