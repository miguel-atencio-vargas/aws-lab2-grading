# AWS Lab 2 - Scalable Infrastructure with Application Load Balancer

This project demonstrates a scalable AWS infrastructure using CloudFormation, featuring VPC networking, Auto Scaling Groups, Application Load Balancer, and automated testing via GitHub Actions.

## Architecture Overview

```
Internet
    │
    ├─► Application Load Balancer (port 80)
    │       │
    │       ├─► Target Group
    │       │       │
    │       │       ├─► EC2 Instance (AZ 1) - Apache Web Server
    │       │       └─► EC2 Instance (AZ 2) - Apache Web Server
    │       │
    │       └─► Auto Scaling Group (2-4 instances)
    │
    └─► VPC (10.0.0.0/16)
            ├─► Public Subnet 1 (10.0.1.0/24) - AZ 1
            └─► Public Subnet 2 (10.0.2.0/24) - AZ 2
```

### Key Features

- **VPC**: Custom VPC with Internet Gateway
- **High Availability**: Resources deployed across 2 Availability Zones
- **Load Balancing**: Internet-facing Application Load Balancer
- **Auto Scaling**: EC2 Auto Scaling Group (min: 2, max: 4 instances)
- **Security**: Security groups restrict access (EC2 only accessible via ALB)
- **Automated Testing**: GitHub Actions workflow with pytest validation

## Prerequisites

- **AWS Account** with appropriate permissions
- **AWS CLI** installed and configured ([Installation Guide](https://aws.amazon.com/cli/))
- **Python 3.8+** for running tests locally
- **GitHub Account** (for automated testing)

## Infrastructure Components

### Security Groups

1. **ALB Security Group**
   - Inbound: HTTP (port 80) from `0.0.0.0/0`
   - Outbound: All traffic

2. **EC2 Security Group**
   - Inbound: HTTP (port 80) from ALB Security Group **only**
   - Outbound: All traffic
   - **Important**: EC2 instances are NOT directly accessible from the internet

### EC2 Configuration

- **AMI**: Latest Amazon Linux 2 (dynamically resolved)
- **Instance Type**: t2.micro
- **Web Server**: Apache httpd serving "Hello from the app"
- **User Data**: Automatic installation and configuration on launch

## Deployment Instructions

### Option 1: Manual Deployment (AWS CLI)

1. **Clone the repository**
   ```bash
   git clone <your-repo-url>
   cd aws-lab2-grading
   ```

2. **Deploy the CloudFormation stack**
   ```bash
   aws cloudformation deploy \
     --stack-name coderoad-lab-2 \
     --template-file infrastructure.yaml \
     --region us-east-1
   ```

3. **Wait for deployment to complete** (typically 5-10 minutes)
   ```bash
   aws cloudformation wait stack-create-complete \
     --stack-name coderoad-lab-2 \
     --region us-east-1
   ```

4. **Get the ALB DNS name**
   ```bash
   aws cloudformation describe-stacks \
     --stack-name coderoad-lab-2 \
     --query 'Stacks[0].Outputs[?OutputKey==`LoadBalancerDNSName`].OutputValue' \
     --output text
   ```

5. **Test the endpoint**
   ```bash
   curl http://<ALB_DNS_NAME>/
   ```
   
   Expected output:
   ```html
   <html><body><h1>Hello from the app</h1></body></html>
   ```

### Option 2: AWS Management Console

1. Navigate to **CloudFormation** in AWS Console
2. Click **Create Stack** → **With new resources**
3. Upload `infrastructure.yaml`
4. Stack name: `coderoad-lab-2`
5. Click **Next** → **Next** → **Create Stack**
6. Wait for `CREATE_COMPLETE` status
7. Check **Outputs** tab for `LoadBalancerDNSName`

## Testing

### Local Testing (Manual)

1. **Install Python dependencies**
   ```bash
   pip install boto3 pytest requests
   ```

2. **Configure AWS credentials**
   ```bash
   export AWS_ACCESS_KEY_ID=your-access-key
   export AWS_SECRET_ACCESS_KEY=your-secret-key
   export AWS_DEFAULT_REGION=us-east-1
   ```

3. **Run pytest tests**
   ```bash
   pytest test_lab2.py -v
   ```

### Test Coverage

The `test_lab2.py` file includes:

- ✅ `test_stack_exists_and_has_alb_output`: Validates stack and output
- ✅ `test_alb_http_returns_expected_text`: Tests ALB endpoint returns "Hello from the app"
- ✅ `test_target_group_has_healthy_targets`: Confirms healthy instances in target group
- ✅ `test_instances_not_publicly_reachable_on_port_80`: Security validation (instances not directly accessible)

## GitHub Actions Setup

### Prerequisites

You need to set up AWS OIDC (OpenID Connect) authentication for GitHub Actions.

### Step 1: Create OIDC Provider in AWS

1. **Navigate to IAM** → **Identity Providers** → **Add Provider**
2. **Provider Type**: OpenID Connect
3. **Provider URL**: `https://token.actions.githubusercontent.com`
4. **Audience**: `sts.amazonaws.com`
5. Click **Add Provider**

### Step 2: Create IAM Role for GitHub Actions

1. **Navigate to IAM** → **Roles** → **Create Role**
2. **Trusted Entity Type**: Web Identity
3. **Identity Provider**: `token.actions.githubusercontent.com`
4. **Audience**: `sts.amazonaws.com`
5. **Add Condition** (replace with your GitHub org/repo):
   ```
   StringEquals: token.actions.githubusercontent.com:sub = repo:YOUR_ORG/YOUR_REPO:ref:refs/heads/main
   ```
6. **Attach Policies**:
   - `AWSCloudFormationFullAccess`
   - `AmazonVPCFullAccess`
   - `AmazonEC2FullAccess`
   - `ElasticLoadBalancingFullAccess`
   - `AutoScalingFullAccess`
   
   Or create a custom policy with least-privilege access

7. **Role Name**: `github-actions-lab2-role`
8. **Copy the Role ARN** (looks like `arn:aws:iam::123456789012:role/github-actions-lab2-role`)

### Step 3: Configure GitHub Secrets

1. Navigate to your GitHub repository
2. Go to **Settings** → **Secrets and variables** → **Actions**
3. Add the following secrets:
   - `AWS_ROLE_ARN`: The ARN from Step 2
   - (Optional) `AWS_REGION`: Default is `us-east-1` in workflow

### Step 4: Run the Workflow

The workflow automatically runs on:
- Push to `main` branch
- Pull requests to `main` branch
- Manual trigger via **Actions** tab → **Run workflow**

**Manual Trigger Options**:
- Choose whether to delete the stack after testing (useful for cost management)

## Security Considerations

### Network Security

- ✅ EC2 instances in public subnets (have public IPs for NAT-free internet access)
- ✅ Security group rules restrict EC2 to accept traffic **only from ALB**
- ✅ Users cannot directly access EC2 instances via public IP
- ✅ All traffic routed through ALB on port 80

### Best Practices Implemented

- VPC isolation
- Security group-based access control
- Health checks for high availability
- Multi-AZ deployment for fault tolerance
- Auto Scaling for resilience

## Cost Optimization

### Running Costs (Approximate)

- Application Load Balancer: ~$16/month + data processing
- EC2 t2.micro instances (2): ~$14/month
- Data transfer: Variable
- **Total**: ~$30-40/month

### Cost Savings Tips

1. **Delete stack when not in use**:
   ```bash
   aws cloudformation delete-stack --stack-name coderoad-lab-2
   ```

2. **Use smaller instance types** (already using t2.micro)

3. **Reduce Auto Scaling capacity** during testing

## Cleanup

### Delete the Stack

**AWS CLI**:
```bash
aws cloudformation delete-stack --stack-name coderoad-lab-2
aws cloudformation wait stack-delete-complete --stack-name coderoad-lab-2
```

**AWS Console**:
1. Navigate to CloudFormation
2. Select `coderoad-lab-2` stack
3. Click **Delete**
4. Confirm deletion

All resources (VPC, subnets, ALB, EC2 instances, security groups) will be automatically deleted.

## Troubleshooting

### Issue: ALB returns 503 Service Unavailable

**Cause**: Instances are still launching or unhealthy

**Solution**: Wait 2-3 minutes for instances to pass health checks
```bash
aws elbv2 describe-target-health \
  --target-group-arn <YOUR_TARGET_GROUP_ARN>
```

### Issue: CloudFormation deployment fails

**Cause**: Insufficient permissions or resource limits

**Solution**: 
- Check IAM permissions
- Verify VPC limits not exceeded
- Check CloudFormation events:
  ```bash
  aws cloudformation describe-stack-events --stack-name coderoad-lab-2
  ```

### Issue: Tests fail with "instances publicly reachable"

**Cause**: Security group misconfiguration

**Solution**: Verify EC2 security group only allows traffic from ALB SG:
```bash
aws ec2 describe-security-groups --group-ids <EC2_SG_ID>
```

## Resources

- [AWS CloudFormation Documentation](https://docs.aws.amazon.com/cloudformation/)
- [Application Load Balancer Guide](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/)
- [Auto Scaling Documentation](https://docs.aws.amazon.com/autoscaling/)
- [GitHub Actions OIDC Guide](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)

## License

This project is for educational purposes as part of AWS Lab 2.