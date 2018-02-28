# Build

Assuming that you have valid credentials on aws to run packer (~/.aws/credentials)

```
VPC_ID=
SUBNET_ID=
packer build \
  -var "vpc_id=$VPC_ID" \
  -var "subnet_id=$SUBNET_ID" \
  kube-<kube-version>.json
```

This will create a custom AMI on your AWS account.

> **Important:** The packer template is pre-selected to run on a specific VPC and subnet

More Info: https://www.packer.io/intro/getting-started/build-image.html#your-first-image

