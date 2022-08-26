# AWS WAF Quick Start
**lith0ps**  
**Aug 21, 2022**   
**WAF, AWS-CLI**  

This is quick tutorial intended to demonstrate how to configure AWS WAF using the AWS CLI. Some knowledge of the AWS CLI, linux, and AWS in general is assumed. The given example uses one of AWS' managed rule sets. Managed rules provide some baseline functionality for common vulnerabilities. Bad inputs, bot actions, etc... In this tutorial we'll:

1. Provision an instance with an application that has known vulnerabilities. 
2. Configure a web acl that contains our rules. 
3. Test these rules to observe how the WAF rejects bad traffic.

>Disclaimer: I'm not responsible for any damages to your environment or any charges you incur following this tutorial. This guide is provided to you as is! Consider reading through the tutorial, then consulting the  [AWS pricing calculator](https://calculator.aws/#/) for an estimate of cost. 

## JQ

The output of some commands will be used as input for subsequent commands. I use some dummy values, but these should be replaced with your own. Read each command carefully! JQ is a json parser that makes dealing with the AWS CLI output a tad easier. But you don't **have** to use it! See the [documentation](https://stedolan.github.io/jq/) for platform specific installation instructions.


```bash
# for macos systems 
wget https://github.com/stedolan/jq/releases/download/jq-1.6/jq-osx-amd64
mv jq-osx-amd64 jq
jq --version
```

## Configuring a Vulnerable App

Every AWS account starts with a default VPC. For simplicity we'll use it in our example instead of creating a new one. But this is dangerous, and not necessary! It's only suitable for this example. For accounts that run production environments consider creating a new vpc for testing new technologies. To use the default VPC, start by finding the VPC ID and it's associated subnets:

```bash
aws ec2 describe-vpcs \
    --filters '[{"Name": "is-default","Values": ["true"]}]' \
    | jq '.Vpcs[0].VpcId'
```

`vpc-0b6844aabf0345867`

```bash
aws ec2 describe-subnets \
    --filters '[{"Name": "vpc-id","Values": ["vpc-0b6844aabf0345867"]}]' \
    | jq '.Subnets[0].SubnetId'
```
`subnet-0dacebb25a60c07f3`

```bash
aws ec2 describe-subnets \
--filters '[{"Name": "vpc-id","Values": ["vpc-0b6844aabf0345867"]}]' \
    | jq '.Subnets[1].SubnetId'
```
`subnet-0ef18b97079fbfe84`

There is also a default security group in every default VPC. We will not use it. Create a new one. By default, security groups allow no traffic.


```bash
aws ec2 create-security-group \
--vpc-id vpc-0b6844aabf0345867 \
--group-name Juice \
--description 'Allow http access to JUICE SHOP' | jq '.GroupId'
```

`sg-03bd3f3f85d82a435`

Only a single rule is needed to allow incoming http. Security groups are stateful, so return traffic is allowed automatically. This will prevent any reverse shells, including ones based on http. This is incase someone manages to find your insecure application during testing (unlikely). 

```
aws ec2 authorize-security-group-ingress --group-id sg-03bd3f3f85d82a435 --protocol tcp --port 80 --cidr 0.0.0.0/0
```

When creating an EC2 instance we can pass a script that will automate the installation of JUICE SHOP. The script assumes an amazon linux image will be used. AMI ids differ by [region](https://aws.amazon.com/amazon-linux-ami/). Choose one suitable for your region.


```sh
vi  userdata.txt
```

```sh
#!/bin/bash
yum update -y
yum install -y docker
service docker start
docker pull bkimminich/juice-shop
docker run -d -p 80:3000 bkimminich/juice-shop
```

```sh
aws ec2 run-instances \
    --image-id ami-051dfed8f67f095f5 \
    --count 1 \
    --instance-type t3.micro \
    --security-group-ids sg-03bd3f3f85d82a435 \
    --subnet-id subnet-0dacebb25a60c07f3 \
    --user-data file://userdata.txt \
    | jq '.Instances[0].InstanceId'
```

`i-09e426233c05c7557`


AWS WAF requires a target to associate ACLs with. While several are supported, the most popular targets are elastic load balancers and cloudfront. We'll create a load balancer as our target, which will deliver the content from our EC2 instance. 

```sh
aws elbv2 create-load-balancer \
--name juice-load-balancer  \
--subnets subnet-0dacebb25a60c07f3 subnet-0ef18b97079fbfe84 \
--security-groups sg-03bd3f3f85d82a435 \
| jq '.LoadBalancers[0].LoadBalancerArn'
```

`arn:aws:elasticloadbalancing:us-east-2:165400142583:loadbalancer/app/juice-load-balancer/17965b5b583d3504`

```sh
aws elbv2 create-target-group \
--name juice-alb-targets \
--protocol HTTP --port 80 \
--vpc-id vpc-0b6844aabf0345867 \
| jq '.TargetGroups[0].TargetGroupArn'
```

`arn:aws:elasticloadbalancing:us-east-2:165400142583:targetgroup/juice-alb-targets/1f8a05adc1618ee9`

Use the instance ID for the `--targets Id=` flag...

```sh
aws elbv2 register-targets \
--target-group-arn arn:aws:elasticloadbalancing:us-east-2:165400142583:targetgroup/juice-alb-targets/1f8a05adc1618ee9 \
--targets Id=i-09e426233c05c7557
```

```sh
aws elbv2 create-listener \
--load-balancer-arn arn:aws:elasticloadbalancing:us-east-2:165400142583:loadbalancer/app/juice-load-balancer/17965b5b583d3504 \
--protocol HTTP \
--port 80 \
--default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-2:165400142583:targetgroup/juice-alb-targets/1f8a05adc1618ee9
```

For testing we'll need access to juice shop. HTTPS isn't configured, and many modern browsers will assume you want HTTPS, throwing an error. When requesting a page from Juice Shop prepend `http`.

```sh
aws elbv2 describe-load-balancers \
--load-balancer-arns arn:aws:elasticloadbalancing:us-east-2:165400142583:loadbalancer/app/juice-load-balancer/17965b5b583d3504 \
| jq '.LoadBalancers[0].DNSName'
```

`juice-load-balancer-2054615823.us-east-2.elb.amazonaws.com`

## Configuring and Testing a Managed Rule Set 

If you can access Juice Shop without issue, move on to the next steps in which we'll set up a web ACL. The web ACL is the part of the WAF that matches traffic to specific rules. [Managed rule sets](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-list.html) are curated by aws and should be used for basic coverage. The rule set `AWSManagedRulesSQLiRuleSet` matches SQL injection payloads. Web ACls accept a JSON object to define their rule set:

```bash
vi waf-rule.json
```

```json
[
    {
        "Name":"juice-rules",
        "Priority":0,
        "Statement":{
            "ManagedRuleGroupStatement": {
                "VendorName": "AWS",
                "Name": "AWSManagedRulesSQLiRuleSet"
            }
        },
        "OverrideAction": {
            "None": {}
        },
        "VisibilityConfig":{
            "SampledRequestsEnabled":false,
            "CloudWatchMetricsEnabled":false,
            "MetricName":"juice-rules"
        }
    }
]
```

```sh
aws wafv2 create-web-acl \
    --name Juice \
    --scope REGIONAL \
    --default-action Allow={} \
    --visibility-config SampledRequestsEnabled=false,CloudWatchMetricsEnabled=false,MetricName=Juice-ACL \
    --rules file://waf-rule.json \
    --region us-east-2 \
    | jq '.Summary.ARN'
```

`arn:aws:wafv2:us-east-2:165400142583:regional/webacl/Juice/819640d1-f2b4-46ab-afb2-67af0f173226`

After the ACL is created, associate it with the load balancer previously created to analyze traffic. 

```sh
aws wafv2 associate-web-acl \
    --web-acl-arn arn:aws:wafv2:us-east-2:165400142583:regional/webacl/Juice/819640d1-f2b4-46ab-afb2-67af0f173226 \
    --resource-arn arn:aws:elasticloadbalancing:us-east-2:165400142583:loadbalancer/app/juice-load-balancer/17965b5b583d3504 \
    --region us-east-2
```

We can test the WAF by sending juice shop SQL payloads. It doesn't matter which endpoint is used, and it can be done by hand. However Juice Shop has a known SQL injection vulnerability in its login page. We can also use a tool called sqlmap, which generates 100s of SQLi payloads, to exploit this vulnerability. 

```
git clone --depth 1 https://github.com/sqlmapproject/sqlmap.git sqlmap-dev
```

This is a sample login request, captured and provided for convenience. Replace all elb references with your own. 

```sh
vi  request
```

```
POST /rest/user/login HTTP/1.1
Accept: application/json, text/plain, */*
Content-Type: application/json
Origin: http://juice-load-balancer-2054615823.us-east-2.elb.amazonaws.com
Cookie: welcomebanner_status=dismiss; continueCode=E3OzQenePWoj4zk293aRX8KbBNYEAo9GL5qO1ZDwp6JyVxgQMmrlv7npKLVy; language=en
Content-Length: 34
Accept-Language: en-US,en;q=0.9
Host: juice-load-balancer-2054615823.us-east-2.elb.amazonaws.com
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/15.4 Safari/605.1.15
Referer: http://juice-load-balancer-2054615823.us-east-2.elb.amazonaws.com/
Accept-Encoding: gzip, deflate
Connection: keep-alive
{"email":"*","password":"*"}
```

When running SQLMap error code 401 should be ignored. Juice Shop sends this error code on failed login attempts, which we don't care about and will be generating a lot of. The WAF responds with error code 403, which will signify a request matched a rule in the web acl. We shouldn't however see a 200 status code signifying SQL injection bypassed authentication. 

```sh
python3 sqlmap.py -r ./request --ignore-code 401
```

That's it! The WAF is capable of much more like logging, storing flagged requests, and advanced custom rules. It's important to remember how vulnerable Juice Shop is. [Terminate the instance](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/terminate-instances.html) and consider deleting unused resources to avoid legit attacks and racking up unnecessary charges! 

---

**References** 

https://pwning.owasp-juice.shop/part1/running.html

https://sqlmap.org

https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-list.html

https://awscli.amazonaws.com/v2/documentation/api/latest/reference/wafv2

https://docs.aws.amazon.com/cli/latest/reference/elb

https://docs.aws.amazon.com/cli/latest/reference/ec2

https://aws.amazon.com/amazon-linux-ami/


