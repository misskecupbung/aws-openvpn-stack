# AWS OpenVPN Stack

Simple CloudFormation template that builds a VPC (public subnet), Internet access, security controls, S3 bucket, IAM role and an EC2 instance which installs OpenVPN (using the angristan headless script) and uploads a first client profile (`client1.ovpn`) to S3.

## Files
- `cloudformation-openvpn-vpc.yaml` – Main CloudFormation template.
- `.github/workflows/deploy-openvpn.yml` – GitHub Actions workflow to deploy the stack on push or manual dispatch.

## Prerequisites
- AWS account & credentials (prefer OIDC role assumption). Set secret `AWS_ROLE_TO_ASSUME` or static `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`.
- Existing EC2 key pair (parameter `KeyPairName`).
- Globally unique S3 bucket name (parameter `BucketName`).

## Quick Deploy (CLI)
```bash
aws cloudformation deploy \
  --template-file cloudformation-openvpn-vpc.yaml \
  --stack-name openvpn-dev \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    EnvironmentName=dev \
    VpcCidr=10.20.0.0/16 \
    PublicSubnetCidr=10.20.1.0/24 \
    AllowedClientCidr=YOUR_PUBLIC_IP/32 \
    InstanceType=t3.micro \
    KeyPairName=YOUR_KEYPAIR \
    BucketName=your-unique-bucket \
    Architecture=arm64 \
    LatestAmiId=/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-arm64
```

## GitHub Actions (Manual Dispatch)
Trigger the workflow in the Actions tab, provide inputs (stack name, CIDRs, key pair, bucket name). It will validate and deploy the stack, then show outputs.

## Retrieve Client Profile
Wait a few minutes for bootstrap, then:
```bash
aws s3 cp s3://your-unique-bucket/client1.ovpn ./client1.ovpn
```
Import `client1.ovpn` into your OpenVPN client.

## Important Parameters
- `AllowedClientCidr`: Restrict to your IP (/32) for SSH and VPN access.
- `BucketName`: Must be globally unique; never made public.
- `Architecture`: `arm64` or `x86_64` (match instance type and AMI).

## Security Notes (Minimal)
- Change `AllowedClientCidr` from default `0.0.0.0/0` before production use.
- Add encryption (EBS & S3) and logging for production hardening.
- Rotate client certs; consider adding more clients with the script (`./openvpn-install.sh --addclient`).

## Clean Up
```bash
aws cloudformation delete-stack --stack-name openvpn-dev
```
(Remember: deleting the stack removes the EC2 instance; add `DeletionPolicy: Retain` to bucket if you want to keep profiles.)

## License
Use at your own risk; review and adapt for production security requirements.
