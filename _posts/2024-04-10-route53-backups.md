---
layout: post
title: Route53 Backups
date: 2024-04-10 12:00:00 +0300
categories: [AWS, Route53, DNS]
---

# Route 53 Zone Backups 
  *serverless backup of route 53 data into zone files using Lambda*

#### Services Used
- Boto3 (AWS Python SDK)
- Route53 (DNS services which needs to be backed up)
- Lambda (run the process serverlessly)
- SNS (For Notification)

---

### Problem and Scope

DNS is one of the most critical services yet often the less paid attention too and seemingly complex.

Cloud Proivders are notorious for not helping you create a backed up zone file to keep you vendor locked.

However they do provide a quick functionality to upload zone files to get you registered. So that is why we need a seprate process. 


### Understanding the Solution

What we need to do is to somehow query the changing DNS record sets, arrange them into appropriate format for a zone file and store it somewhere.

The solution proposed here uses :

- **Boto3**, which is the Amazon SDK for all of its services, which we to query the data that comes in a json format using;

- **Lamda** (Serverless Compute) to fetch parse that json data in a python environment, convert it into a formatted zone file and put it on to ;

- **S3** , to store the files on to the S3 Bucket. and Auxillarly use, 

- **SNS** to help us with notification of the backups.


### Code

We use the *get_hosted_zone_count* method to get the total number of hosted zones on route53 and use the *list_hosted_zones* to get a list of all the **Hosted Zone id's** which we need to fetch all the resource records. We put all the zone id's into a list and interate through the list calling the next function which will process fetch indivual records.

```python
import boto3
import os.path

def Route53_Backup():
    client = boto3.client('route53')
    Number_of_Hosted_Zones = (client.get_hosted_zone_count())["HostedZoneCount"]
    response = client.list_hosted_zones()
    
    Hosted_Zones = response["HostedZones"]
    Hosted_List = []
    
    for i in range(len(Hosted_Zones)):                                                      
        if (i > Number_of_Hosted_Zones):
            exit
        Hosted_List.append(((Hosted_Zones[i])["Id"]).lstrip("/hostedzone/"))
        
    for zone in Hosted_List:
        Create_Zone_File(zone)
```

Here, for each zone  We query the *list_resource_record_sets* method which takes *zone id* as the argument and fetches the data in a Json format.

Creating a zone file  (this step could be skipped )

A zone file is in text format , so we query the json object for the fields we need and arrange those into a string to write to a text file locally.

Due to the complexity of records that are possible we restrict to standard DNS records that all providers have no problem importing for creating the zone file. The rest of the complex records we dump in a seprate json file that could be uploaded seprately.

These objects are then passed on to be uploaded to a preconfigured S3 Bucket

```python    
def Create_Zone_File(zone):
    print("Processing zone:" + zone + "...")
    client = boto3.client('route53')
    paginator = client.get_paginator('list_resource_record_sets')
    response_iterator = paginator.paginate(HostedZoneId=zone)
    
    for page in response_iterator:
        
        Resource_Record_Sets = page['ResourceRecordSets']
        Temp_File_Name = "/tmp/"+ "route53-zone-file-"+ zone +".txt"
        errors_file = "/tmp/errors" + zone + ".txt"
        
        for single_record in Resource_Record_Sets:
            if (((single_record.get("Name")) != None) and ((single_record.get("TTL")) != None and ((single_record.get("Type")) != None) and ((single_record.get("ResourceRecords")) != None))):
                with open(Temp_File_Name, "a") as my_file:
                    my_file.write("\n" + single_record["Name"] + "\t" + str(single_record["TTL"]) + "\t" + single_record["Type"]+ "\t" + (single_record["ResourceRecords"][0]["Value"]))
            else :
                with open(errors_file, "a") as my_file:
                    my_file.write(str(single_record))
        
        if(os.path.isfile(Temp_File_Name)):
            Upload_To_S3(Temp_File_Name)
        
        if (os.path.isfile(errors_file)):
            Upload_To_S3(errors_file)
```
This is the easy part, we use *s3.upload_file* method providing the file, bucket name and the destination key.

```python
def Upload_To_S3 (Temp_File_Name) :
    
    s3 = boto3.client('s3')
    s3.upload_file(Temp_File_Name, "your-bucket-name", Temp_File_Name.lstrip("/tmp/"))
    return("uploaded to s3")

```

```python        
def lambda_handler(event, context):
    
    Route53_Backup()
    return {
        'statusCode': 200,
        'body': json.dumps('Route53 Backup Successfull!!!')
    }

```

### Deployment

Once we create the lambda function , we still need to do a couple of things to make it work. We need a trigger to actually run the script and some sort of a notification for when it runs.



**Triggering** Here simply I use a eventbridge rate rule to run the function periodically howeveer lambda provides multiple ways you could trigger this.

**Notification** Here Simply I use a Amazon SNS topic that sends an email, however lambda again is pretty versatile in providing to your needs.

**Monitoring** The Script is too simple and does not need complex monitoring or logging, python log stattements are automatically pushed to cloudwatch logs for lambda functions making everyone's life easier.


### **Access Control** 

Lambda autocreates a role for you, adding appropriate permissions to the function is needed for s3, route53 and SNS.

These policies are an example and should be altered to best meet your configuration and security needs.

```json
{
    "Version": "2012-10-17",

    "Statement": [
        {
            "Sid": "r53-access",
            "Effect": "Allow",
            "Action": [
                "route53:GetHostedZone",
                "route53:ListHostedZones",
                "route53:ListResourceRecordSets",
                "route53:GetHostedZoneCount",
                "route53:GetHostedZoneLimit"
            ],
            "Resource": "*"
        }
    ]
    
    "Statement": [
        {
            "Sid": "s3-access",
            "Effect": "Allow",
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::<bucket-name>/*"
        }
    ]

    "Statement": [
        {
            "Sid": "sns-access"
            "Effect": "Allow",
            "Action": "sns:Publish",
            "Resource": "arn:aws:sns:<region>:<aws-account-id>:<sns-topic-name>"
        }
    ]
}
``````


