
#Requirements:
* A VPC with 3 public and 3 private subnets
  * the private subnets must be behind a NAT gateway (or multiple)
* AMI: It's recommended that you use the AMIs supplied in the template, which are CentOS 7 based and come from here: [github.com/irvingpop/packer-chef-highperf-centos7-ami](https://github.com/irvingpop/packer-chef-highperf-centos7-ami)
  * If you use your own, the following things need to be installed:
    - awscli (`aws` command)
    - Cloudformation Helper Scripts (`cfn-init` and `cfn-signal` commands)
    - NTP (installed and enabled)


#Creating a VPC to spec
```bash
# create the VPCs
aws cloudformation create-stack --stack-name myname-vpc --template-body file://vpc/vpc-3azs.yaml --capabilities CAPABILITY_IAM --parameters ParameterKey=ClassB,ParameterValue=42

# create the NAT gateway
aws cloudformation create-stack --stack-name myname-vpc-natgw --template-body file://vpc/vpc-nat-gateway.yaml --capabilities CAPABILITY_IAM --parameters ParameterKey=ParentVPCStack,ParameterValue=myname-vpc

# create the bastion host
aws cloudformation create-stack --stack-name myname-vpc-bastion --template-body file://vpc/vpc-ssh-bastion.yaml --capabilities CAPABILITY_IAM --parameters ParameterKey=ParentVPCStack,ParameterValue=myname-vpc ParameterKey=KeyName,ParameterValue=my_ssh_key
```

# Fire up the backendless chef server stack
```bash
aws cloudformation create-stack \
  --stack-name irving-backendless-chef \
  --template-body file://backendless_chef.yaml \
  --capabilities CAPABILITY_IAM \
  --on-failure DO_NOTHING \
  --parameters \
  ParameterKey=SSLCertificateARN,ParameterValue=arn:aws:iam::862552916454:server-certificate/ip-ub-backend1-trusty-aws-1164570181.us-west-2.elb.amazonaws.com \
  ParameterKey=LicenseCount,ParameterValue=999999 \
  ParameterKey=DBUser,ParameterValue=chefadmin \
  ParameterKey=DBPassword,ParameterValue=VerySecure \
  ParameterKey=KeyName,ParameterValue=irving@getchef.com \
  ParameterKey=VPC,ParameterValue=vpc-0012f067 \
  ParameterKey=SSHSecurityGroup,ParameterValue=sg-bf53c1c6 \
  'ParameterKey=LoadBalancerSubnets,ParameterValue="subnet-ff2f279b,subnet-6c30121a,subnet-0b61fa53"' \
  'ParameterKey=ChefServerSubnets,ParameterValue="subnet-fe2f279a,subnet-6d30121b,subnet-0c61fa54"' \
  'ParameterKey=NatGatewayIPs,ParameterValue="35.160.121.138"' \
  ParameterKey=InstanceType,ParameterValue=c4.xlarge \
  ParameterKey=DBInstanceClass,ParameterValue=db.m4.xlarge
```

# SSH to your hosts

If you're using a bastion host:
```bash
ssh -o ProxyCommand="ssh -W %h:%p -q ec2-user@35.160.211.71" -l centos <chef server private ip>
```

otherwise just login as `centos` to the private IPs of the chef servers
