CloudFormation example to create Ansible build server and automate Drupal Installation on target EC2 instance using Ansible Playbook.
--------
This main objective of this example is to install Ansible build server on an EC2 instance from scratch using a custom CloudFormation template that also creates a new VPC, Subnets, Elastic IP, Security Group, and other resources. We will then use this new Ansible build instance to automate the installation of basic Drupal website in the target EC2 instance running on Amazon Linux using Ansible Playbook. 


### Assumptions:

It is assumed that the CloudFormation template will use t2.micro instances running on us-east-1 region. Also, the CloudFormation template will create a new VPC with CidrBlock 10.0.0.0/16 along with the required resources needed to setup the ansible build server.

### Ansible build server:

Ansible is a simple, agentless way to automate your infrastructure. If you find yourself deploying Drupal or WordPress over and over again, Ansible could save you a lot of time. In this example, with a few lines of YAML (a straighforward markup language), we will automate the typically tedious process of setting up Drupal website on a fresh Amazon Linux AMI.

### Installation (source ansible-build-server):

The first step is to create an EC2 KeyPair and create all the resources that we need using these custom cloudformation template cloudformation-ansible-build-server.json (with EIP assigned) or cloudformation-ansible-build-server-no-eip.json (With Public IP address assigned) files.

```
aws ec2 create-key-pair --key-name MyKeyPair --query 'KeyMaterial' --output text > MyKeyPair.pem

aws cloudformation create-stack --stack-name MyAnsibleCF --template-body file:////Documents//cloudformation-ansible-build-server.json  --parameters ParameterKey=KeyName,ParameterValue=MyKeyPair

aws ec2 describe-instances
```


### Drupal Installation (target drupal-server):

SSH to the source ansible build server and check the installation:

```
ansible --version
ansible 1.9.2
  configured module search path = None


python
Python 2.7.9 (default, Apr  1 2015, 18:18:03) 
[GCC 4.8.2 20140120 (Red Hat 4.8.2-16)] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import boto
>>> boto.Version
'2.38.0'
>>>
```

To successfully make an API call to AWS, you will need to configure Boto (the Python interface to AWS). But the simplest is just to export the following environment variables (input your ACCESS and SECRET keys below):
```
Export AWS credentials in Ansible host:
export EC2_ACCESS_KEY=""
export EC2_SECRET_KEY=""
export EC2_URL=https://ec2.amazonaws.com
export S3_URL=https://s3.amazonaws.com:443
export AWS_ACCESS_KEY_ID=${EC2_ACCESS_KEY}
export AWS_SECRET_ACCESS_KEY=${EC2_SECRET_KEY}
export AWS_DEFAULT_REGION='us-east-1'
```

You can test the AWS EC2 External Inventory Script to make sure your config is correct:

```
/ansible/contrib/inventory/ec2.py --list  
```

Create an EC2 Keypair to use with the target Drupal website

```
aws ec2 create-key-pair --key-name MyDrupalKey --query 'KeyMaterial' --output text > MyDrupalKey.pem
cp MyDrupalKey.pem ~/.ssh/ 
chmod 600 ~/.ssh/MyDrupalKey.pem
ssh-agent bash
ssh-add ~/.ssh/MyDrupalKey.pem
```

Now, edit the DrupalCloudFormation.yaml template and change **stack_name** and **KeyName** and other basic parameters

```yaml
---
- name: Launch a Drupal single instance CloudFormation stack
  hosts: localhost
  connection: local
  gather_facts: false

  tasks:
  - name: Launch a CloudFormation stack
    cloudformation: >
      stack_name="DrupalWebsite"
      state=present
      region=us-east-1
      disable_rollback=true
      template=Drupal_Single_Instance.template
    args:
      template_parameters:
        KeyName: MyDrupalKey
        SiteName: "My Drupal website"
    register: stack
  - name: Show stack outputs
    debug: msg="My stack outputs are {{stack.stack_outputs}}"
``` 

Also, edit the Drupal_Single_Instance.template and set the KeyName if created with a different name other than the example used MyDrupalKey.

Finally, automate the installation of a new Drupal instance using Ansible Playbook

```
ansible-playbook DrupalCloudFormation.yaml
```

Check the information about the relevant EC2 instances with the following commands:

```
aws cloudformation describe-stacks --stack-name DrupalWebsite
aws ec2 describe-instances
```

Test your Drupal website installation and change details such as the admin password and other configuration options.
