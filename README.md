# aws-EC2UserData & Blue-Green Deployment using Route53

# Lab enviroment will be consist of 3 CFN templates: 
1. Create common VPC teamplate to deploy in mulitples region 
2. Create Security groups
3. Create Public instances
4. Tesing using curl
5. AWS CLI- change-reslurce-record-sets

# Parameters 
```bash
ParameterKey='VPCCIDR',ParameterValue='192.168.0.0/16'
ParameterKey='PublicSubnet1CIDR',ParameterValue='192.168.1.0/24'
ParameterKey='PublicSubnet2CIDR',ParameterValue='192.168.2.0/24'
ParameterKey='PublicSubnet3CIDR',ParameterValue='192.168.3.0/24'
ParameterKey='RegionCode',ParameterValue='sgp'
ParameterKey='AZ1Code',ParameterValue='sgpaz1'
ParameterKey='AZ2Code',ParameterValue='sgpaz2'
ParameterKey='AZ3Code',ParameterValue='sgpaz3'
```

# IP Addresss assigments 
```
sgpvpc - 192.168.0.0/16
sydvpc - 172.16.0.0/16
tkovpc - 10.10.0.0/16
```

# Steps: 

## 1. Create common VPC teamplate and deploy in mulitples region (sg, syd, tokyo)
- [vpc.yaml](./Templates/vpc.yaml)

### i) `sgpvpc` for Mastervpc" 

```bash 
aws cloudformation create-stack --stack-name sgpvpc --template-body file://vpc.yaml --parameters ParameterKey='VPCCIDR',ParameterValue='192.168.0.0/16' ParameterKey='PublicSubnet1CIDR',ParameterValue='192.168.1.0/24' ParameterKey='PublicSubnet2CIDR',ParameterValue='192.168.2.0/24' ParameterKey='PublicSubnet3CIDR',ParameterValue='192.168.3.0/24' ParameterKey='RegionCode',ParameterValue='sgp' ParameterKey='AZ1Code',ParameterValue='sgpaz1' ParameterKey='AZ2Code',ParameterValue='sgpaz2' ParameterKey='AZ3Code',ParameterValue='sgpaz3'
```
### ii) `sgpvpc` for "Securitygroup" 

```bash
aws cloudformation create-stack --stack-name sgvpc-securitygroup --template-body file://vpc-securitygroup.yaml --parameters ParameterKey='vpcStackName',ParameterValue='sgpvpc' 
```
## 2. Create public-instance 
- [public-instance-v1.yaml](./Templates/public-instance-v1.yaml)


### i) `sgpvpc` for "public-instance-v1"

```bash
aws cloudformation create-stack --stack-name instancev1 --template-body file://public-instance-v1.yaml --parameters ParameterKey='vpcStackName',ParameterValue='sgpvpc' ParameterKey='vpcSecurityGroupStackName',ParameterValue='sgvpc-securitygroup' ParameterKey='appVersion',ParameterValue='v1'
```

### ii) `sgpvpc` for "public-instance-v2"

```bash
aws cloudformation create-stack --stack-name instancev2 --template-body file://public-instance-v2.yaml --parameters ParameterKey='vpcStackName',ParameterValue='sgpvpc' ParameterKey='vpcSecurityGroupStackName',ParameterValue='sgvpc-securitygroup' ParameterKey='appVersion',ParameterValue='v2'
```
## 3. Setup Route53 CNAME Records


## 4.Setup Traffic Policies



<br>

## 5. Create AWS CLI- Change-resource-record-sets

### i) 
- **CREATE** : Creates a resource record set that has the specified values.
- **DELETE** : Deletes an existing resource record set that has the specified values.
- **UPSERT** :
If a resource record set does not already exist, AWS creates it. If a
resource set does exist, Route 53 updates it with the values in the
request.

### ii)generate-cli-skeleton

```bash
[sai@sais-MBP 9.ec2-userdata]$ aws route53 change-resource-record-sets --generate-cli-skeleton
{
    "HostedZoneId": "",
    "ChangeBatch": {
        "Comment": "",
        "Changes": [
            {
                "Action": "CREATE",
                "ResourceRecordSet": {
                    "Name": "",
                    "Type": "A",
                    "SetIdentifier": "",
                    "Weight": 0,
                    "Region": "us-east-2",
                    "GeoLocation": {
                        "ContinentCode": "",
                        "CountryCode": "",
                        "SubdivisionCode": ""
                    },
                    "Failover": "SECONDARY",
                    "MultiValueAnswer": true,
                    "TTL": 0,
                    "ResourceRecords": [
                        {
                            "Value": ""
                        }
                    ],
                    "AliasTarget": {
                        "HostedZoneId": "",
                        "DNSName": "",
                        "EvaluateTargetHealth": true
                    },
                    "HealthCheckId": "",
                    "TrafficPolicyInstanceId": ""
                }
            }
        ]
    }
}
```

### ii) Create a simple resource record set in Route53 using AWS CLI

### iii) app-v1.json
```json
{
      "Comment": "UPSERT CNAME record to gritworks.io blue v1 app",
      "Changes": [
          {
              "Action": "UPSERT",
              "ResourceRecordSet": {
                  "Name": "www.gritworks.io",
                  "Type": "CNAME",
                  "TTL": 0,
                  "ResourceRecords": [
                      {
                          "Value": "ec2-18-142-26-173.ap-southeast-1.compute.amazonaws.com"
                      }
                  ]
              }
          }
      ]
  }
```

### iii) app-v2.json
```json
{
  "Comment": "UPSERT CNAME record to gritworks.io green v2 app",
  "Changes": [
      {
          "Action": "UPSERT",
          "ResourceRecordSet": {
              "Name": "www.gritworks.io",
              "Type": "CNAME",
              "TTL": 0,
              "ResourceRecords": [
                  {
                      "Value": "ec2-52-76-205-58.ap-southeast-1.compute.amazonaws.com"
                  }
              ]
          }
      }
  ]
}
```
### iv)change to v1 by running the following command
-[AWS Route53 Pages](https://aws.amazon.com/premiumsupport/knowledge-center/simple-resource-record-route53-cli/)

```bash
$ aws route53 change-resource-record-sets --hosted-zone-id Z0671885NGJVMC1JAYL2 --change-batch file://app-v1.json
{
    "ChangeInfo": {
        "Id": "/change/C09110163NBM7ABPCJNOY",
        "Status": "PENDING",
        "SubmittedAt": "2021-07-25T03:14:15.164000+00:00",
        "Comment": "UPSERT CNAME record to gritworks.io blue v1 app"
    }
}
```
### v) verify the change status:
```bash
$ aws route53 get-change --id /change/C09110163NBM7ABPCJNOY
{
    "ChangeInfo": {
        "Id": "/change/C09110163NBM7ABPCJNOY",
        "Status": "INSYNC",
        "SubmittedAt": "2021-07-25T03:14:15.164000+00:00",
        "Comment": "UPSERT CNAME record to gritworks.io blue v1 app"
    }
}
```

<br>


You may need to update image-id that availble on your different region.
![header image](outputs-images/instance-imageid-err-01.png)
