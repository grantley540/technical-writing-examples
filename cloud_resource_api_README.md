# CRAPI
Crapi (Cloud Resource API) is a service that aggregates AWS and Azure resource data from ACME's many cloud accounts.  This resource data is collected, stored and available for consumption by clients via a REST API.  The data can be used for auditing/compliance, search tools, configuration management solutions and more.

## AWS
Crapi gets it's AWS data from two sources.  The first source is the AWS-config service.  This service provides an AWS-generated snapshot file in an account.  This snapshot contains many different types of AWS resources including things like EC2 instances, IAM roles, IAM users, etc.  This snapshot does not include all AWS resource types, though.  In order to supplement useful resource types missing from the AWS-config service snapshot file, crapi harvests additional resource types from accounts using service-based `describe*` api calls.

The AWS data crapi provides (from both sources) is the exact same data available in each AWS account individually via AWS `describe*` API calls.  The benefit crapi provides is that data from many accounts is aggregated and available from one central location.  This makes it easier to query data across accounts and reduces the strain on AWS APIs (possibly resulting in throttling) and reduces duplicated effort across teams.  

### Supported AWS Resources
#### AWS-config Source
Up-to-date information on what resources AWS-config (and subsequently, crapi) support can be found here:
https://docs.aws.amazon.com/config/latest/developerguide/resource-config-reference.html

#### Additional Harvested Source
Notice that harvested resource types include the prefix `harvest` in the name.  This is to avoid name clashing when AWS adds new types to the AWS-config service.  If a duplicate type is added to the config service, it will be removed from the harvester type to avoid collecting duplicate data.  
```
aws/harvest/ec2/transitgateway
aws/harvest/efs/volume
aws/harvest/es/domain
aws/harvest/directconnect/connection
aws/harvest/directconnect/virtualinterface
aws/harvest/guardduty/detector
aws/harvest/guardduty/masterinvite
aws/harvest/iam/accesskey
aws/harvest/iam/mfadevice
```
#### Resource Schemas
Example Harvested Resource Schema:
```
{
  "ResourceId": "AKIXXXX",
  "ResourceName": null,
  "ResourceType": "aws_harvest_iam_accesskey",
  "AwsRegion": "us-east-1",
  "AwsAccountId": "000000000",
  "AwsAccountAlias": "someaccountalias",
  "HarvestDate": "2052-09-30T20:20:38Z",
  "IndexDate": "2052-09-30T20:20:38Z",
  "TTL": "2052-09-30T21:05:38Z",
  "crapiAuthzScope": "000000000",
  "crapiSource": "harvester",
  "Resource": {
      "AccessKeyId": "AKIXXXXXXXXJ",
      "CreateDate": "2052-08-05T20:22:31Z",
      "Status": "Active",
      "UserName": "XXXXXXX"
  }
}
```
Example AWS-config Snapshot Resource Schema:
```
{
  "AccountId": null,
  "Arn": "arn:aws-us-east-1:iam::000000:user/Administrator",
  "AvailabilityZone": "Not Applicable",
  "AwsRegion": "global",
  "Configuration": null,
  "ConfigurationItemCaptureTime": "2052-07-18T19:15:54.039Z",
  "ConfigurationItemMD5Hash": null,
  "ConfigurationItemStatus": "ResourceDiscovered",
  "ConfigurationStateId": null,
  "RelatedEvents": [],
  "Relationships": [
      {
          "RelationshipName": null,
          "ResourceId": "000000",
          "ResourceName": "Administrators",
          "ResourceType": "AWS::IAM::Group"
      }
  ],
  "ResourceCreationTime": "2025-05-18T14:09:33Z",
  "ResourceId": "000000",
  "ResourceName": "Administrator",
  "ResourceType": "AWS::IAM::User",
  "SupplementaryConfiguration": null,
  "Tags": {},
  "Version": null,
  "configurationStateId": 0000000000009,
  "configuration": {
      "arn": "arn:aws-us-east-1:iam::00000000:user/Administrator",
      "attachedManagedPolicies": [],
      "createDate": "2025-05-18T14:09:33.000Z",
      "groupList": [
          "Administrators"
      ],
      "path": "/",
      "tags": [],
      "userId": "00000",
      "userName": "Administrator",
      "userPolicyList": []
  },
  "supplementaryConfiguration": {},
  "awsAccountId": "000000",
  "configurationItemVersion": "1.3",
  "awsAccountAlias": "000000",
  "IndexDate": "2052-10-01T14:29:36Z",
  "TTL": "2052-10-01T15:14:36Z",
  "crapiAuthzScope": "00000"
}
```

## Azure
Azure data is collected via the Azure resource graph.

### Supported Azure Resources:
Refer to https://docs.microsoft.com/en-us/azure/governance/resource-graph/reference/supported-tables-resources for an up-to-date list of resources included in Azure resource graph.


Example Resource Schema:
```
{
  "IndexDate": "2052-10-01T14:06:08Z",
  "crapiAuthzScope": "000000",
  "ResourceId": "/subscriptions/0000000/resourceGroups/XXX-virtual-machine-images/providers/Microsoft.Compute/images/someimage",
  "TTL": "2052-10-01T15:21:08Z",
  "id": "/subscriptions/00000/resourceGroups/XXX-virtual-machine-images/providers/Microsoft.Compute/images/someimage",
  "identity": "removed",
  "kind": "",
  "location": "northeurope",
  "managedBy": "",
  "name": "XXX-00000000000017",
  "plan": "removed",
  "properties": {
      "hyperVGeneration": "V1",
      "provisioningState": "Succeeded",
      "sourceVirtualMachine": {
          "id": "/subscriptions/00000/resourceGroups/XXX-VM-COMPONENTS/providers/Microsoft.Compute/virtualMachines/someimage"
      },
      "storageProfile": {
          "dataDisks": [
              {
                  "caching": "ReadWrite",
                  "diskSizeGB": 128,
                  "lun": 10,
                  "managedDisk": {
                      "id": "/subscriptions/0000000/resourceGroups/XXX-VM-COMPONENTS/providers/Microsoft.Compute/disks/someimage"
                  },
                  "storageAccountType": "Standard_LRS"
              },
              {
                  "caching": "ReadWrite",
                  "diskSizeGB": 128,
                  "lun": 11,
                  "managedDisk": {
                      "id": "/subscriptions/00000000/resourceGroups/XXX-VM-COMPONENTS/providers/Microsoft.Compute/disks/someimage"
                  },
                  "storageAccountType": "Standard_LRS"
              }
          ],
          "osDisk": {
              "caching": "ReadWrite",
              "diskSizeGB": 128,
              "managedDisk": {
                  "id": "/subscriptions/000000000/resourceGroups/XXX-VM-COMPONENTS/providers/Microsoft.Compute/disks/someimage"
              },
              "osState": "Generalized",
              "osType": "Windows",
              "storageAccountType": "Standard_LRS"
          },
          "zoneResilient": false
      }
  },
  "resourceGroup": "XXX-virtual-machine-images",
  "sku": [],
  "subscriptionId": "000000",
  "tags": [],
  "tenantId": "0000000",
  "type": "microsoft.compute/images",
  "zones": null
}
```

## Data Freshness
AWS data is loaded every 30 minutes.  Azure data is loaded every hour.

Resource data is only stored in the elasticsearch cluster for as long as the resource exists.  If a resource is deleted, it is automatically purged from elasticsearch after the next successful data load (30 minutes for AWS, 1 hour for Azure).  Resource history is not maintained.  

The `TTL` field in a resources data indicates the 'expiration' date and is used by crapi to purge old data.  The `IndexDate` field provides the timestamp (GMT) of when that resource was indexed into crapi.

## How to use
crapi includes a simple RESTful API to query its data.  This API (known as wrappy) is a simple wrapper around elasticsearch that provides a friendlier interface to the data.  

### crapi-Wrappy
Base URL: someurl.com
<br>
This endpoint is publicly accessible on the internet.  The api-key provides security.

#### Auth

An API key is required to use crapi-wrappy. Each API key is mapped to a list of accounts it can access. When you make a request, crapi-wrappy will only return resources belonging to the accounts that the API key is authorized for.

The API key needs to be included in the header of any requests you make as `x-api-key`

To view the list of authorized accounts associated with your api key, you can make a request to the resource type `\account\authorizations`

#### Querying the API
The data in crapi is organized into elasticsearch indices based on the AWS or Azure resource type.  For example, ec2 instances are in one index, IAM roles are in another index and so on.  For AWS-config resources, the 'resource type value' is what drive these elasticsearch index names as well as the crapi-wrappy Restful API resource paths.  For Azure resources, the resource `type` field in the resource graph data is what drives the index names as well as the crapi-wrappy resource paths.

For the latest AWS-config-provided resource types, refer to the AWS-config resource types documentation to understand what path names to use in your API requests for your desired resource types:  
https://docs.aws.amazon.com/config/latest/developerguide/resource-config-reference.html  
This page has tables broken down by aws service and within these tables all of the resource types recorded for that service are listed.
The values in the "Resource Type Value" column in the tables are the values to refer to.  Just replace the double-colons ("::") in these values with a forward-slash ("/") to construct your API request path.

For example, if you want to query for IAM roles, find the IAM service table, locate IAM roles in the "Resource Type Value" column, take the corresponding value and convert the double-colons to forward-slashes. `AWS::IAM::Role` becomes `/aws/iam/role` and thus your API request would look like "https://<API_URL_HERE>/aws/iam/role"

The `Supported AWS Resources > AWS-config Source` section above is a static list of resource names which have already been converted to the forward-slash-delimited names.  For the latest list of resource types included in aws-config/crapi, however, it's recommended to use the above instructions and AWS link.

To query for any of the additional AWS harvested resource types, refer to the resource type names above in the `Supported AWS Resources > Additional Harvested Resources` section to determine the request path to use.  For example, to query for guard duty detectors, your API request would look like "https://<API_URL_HERE>/aws/harvest/guardduty/detector"

For Azure resource data, refer to the list provided above in the "Supported Azure Types" section.  For example, to query for route tables, your API request would look like "https://<API_URL_HERE>/microsoft/network/routetables"

##### Example Queries

```bash
curl -H "x-api-key: <API-KEY>" -H "Content-Type: application/json" "https://<API_URL_HERE>/aws/ec2/instance"
```

```bash
curl -H "x-api-key: <API-KEY>" -H "Content-Type: application/json" "https://<API_URL_HERE>/aws/ec2/instance?configuration.instanceType=t2.small&awsAccountId=000000000000"
```

You can add queries for any valid field name in the resource data.  Examine some of the data to get an idea of the fields available.  Nested field names are valid.  Complex field values such as lists are probably not going to work.

Note: No matter the query, only results from accounts to which your API key is authorized will be returned

##### Pagination
Pagination utilizes the "search after" pagination functionality provided by Elasticsearch:
https://www.elastic.co/guide/en/elasticsearch/reference/master/search-request-search-after.html

Simply query wrappy as described above. Take the resource id of the last resource returned in the hits array `$.hits.hits[<last item>]._id` and pass this id as a `search_after=< id here>` query param in subsequent requests.  You keep doing this until the length of the `$.hits.hits` array is zero.

###### Example:
First request:
```bash
curl -H "x-api-key: <API-KEY>" -H "Content-Type: application/json" "https://<API_URL_HERE>/aws/ec2/instance"
```

Take the resource id (`_id` field) from the last resource in the `$.hits.hits[]` array.  Let's pretend it's `arn:aws-us-east-1:ec2:us-east-1:000000000000:instance/i-XXXXX`

Second request:
```bash
curl -H "x-api-key: <API-KEY>" -H "Content-Type: application/json" "https://<API_URL_HERE>/aws/ec2/instance?search_after=arn:aws:ec2:us-east-1:XXXXXXX:instance/i-XXXXXXX"
```
Keep repeating this process until length of `$.hits.hits[]` equals zero.

##### Size
If you want to return more/less than 100 results in a single response/page, update the `size` parameter (limit 5000).  If you see HTTP 400 errors with the following response body:
```json
{
    "ErrorMsg": "response too large. decrease 'size'"
}
```

decrease the `size` parameter until the error resolves.  Note: different resource types contain varying amounts of data so the maximum `size` parameter allowed will differ per resource type.

Example:
```bash
curl -H "x-api-key: <API-KEY>" -H "Content-Type: application/json" "https://<API_URL_HERE>/aws/ec2/instance?size=1000"
```

##### Data Structure
Raw elasticsearch query responses are returned from crapi-wrappy.
A sample response (with resource data ommitted for brevity) is shown below.

Request URL: `https://<URL_HERE>/aws/ec2/instance`

```javascript
{
    "took": 5,
    "_scroll_id": "",
    "hits": {
        "total": XXXXX,
        "max_score": 1,
        "hits": [
          <RESOURCES HERE>
        ]
    }
}
```

  `$.hits.total` is the total number of instances found (for authorized accounts based on the api-key used)<br>
  `$.hits.hits` is an array containing each of the individual resources (limited to the number returned in one page)

  A complete sample response for the same request URL is:<br>
```javascript
{
  "took": 5,
  "_scroll_id": "",
  "hits": {
      "total": XXXXX,
      "max_score": 1,
      "hits": [
          {
              "_score": 1,
              "_index": "aws_ec2_instance",
              "_type": "_doc",
              "_id": "i-00000000000000",
              "_uid": "",
              "_routing": "",
              "_parent": "",
              "_version": null,
              "sort": null,
              "highlight": null,
              "_source": {
                  "AccountId": null,
                  "Arn": "arn:aws:ec2:us-east-1:XXXXXXX:instance/i-000000000000",
                  "AvailabilityZone": "us-east-1b",
                  "AwsRegion": "us-east-1",
                  "Configuration": null,
                  "ConfigurationItemCaptureTime": "2025-06-16T02:13:14.693Z",
                  "ConfigurationItemMD5Hash": null,
                  "ConfigurationItemStatus": "OK",
                  "ConfigurationStateId": null,
                  "RelatedEvents": [],
                  "Relationships": [
                      {
                          "RelationshipName": null,
                          "ResourceId": "eni-00000000",
                          "ResourceName": null,
                          "ResourceType": "AWS::EC2::NetworkInterface"
                      },
                      {
                          "RelationshipName": null,
                          "ResourceId": "sg-00000000",
                          "ResourceName": null,
                          "ResourceType": "AWS::EC2::SecurityGroup"
                      },
                      {
                          "RelationshipName": null,
                          "ResourceId": "subnet-000000000",
                          "ResourceName": null,
                          "ResourceType": "AWS::EC2::Subnet"
                      },
                      {
                          "RelationshipName": null,
                          "ResourceId": "vol-00000000000",
                          "ResourceName": null,
                          "ResourceType": "AWS::EC2::Volume"
                      },
                      {
                          "RelationshipName": null,
                          "ResourceId": "vol-000000000000111",
                          "ResourceName": null,
                          "ResourceType": "AWS::EC2::Volume"
                      },
                      {
                          "RelationshipName": null,
                          "ResourceId": "vpc-00000000",
                          "ResourceName": null,
                          "ResourceType": "AWS::EC2::VPC"
                      }
                  ],
                  "ResourceCreationTime": "2028-06-15T19:39:09Z",
                  "ResourceId": "i-00000000000000",
                  "ResourceName": null,
                  "ResourceType": "AWS::EC2::Instance",
                  "SupplementaryConfiguration": null,
                  "Tags": {
                      "Name": "some name",
                      "app": "000000",
                      "chef_meta": "chef",
                      "created_timestamp": "2026-06-15 19:39:09+00:00",
                      "env": "prod",
                      "terraform": "1",
                      "xxx": "0000000"
                  },
                  "Version": null,
                  "configurationStateId": 0000000000003,
                  "configuration": {
                      "amiLaunchIndex": 0,
                      "architecture": "x86_64",
                      "blockDeviceMappings": [
                          {
                              "deviceName": "/dev/sda1",
                              "ebs": {
                                  "attachTime": "2029-06-15T19:39:10.000Z",
                                  "deleteOnTermination": true,
                                  "status": "attached",
                                  "volumeId": "vol-000000000000"
                              }
                          },
                          {
                              "deviceName": "/dev/sdg",
                              "ebs": {
                                  "attachTime": "2029-06-15T19:39:10.000Z",
                                  "deleteOnTermination": true,
                                  "status": "attached",
                                  "volumeId": "vol-00000000000011"
                              }
                          }
                      ],
                      "clientToken": "",
                      "ebsOptimized": false,
                      "elasticGpuAssociations": [],
                      "enaSupport": true,
                      "hypervisor": "xen",
                      "iamInstanceProfile": {
                          "arn": "arn:aws:iam::XXXXXXX:instance-profile/blahblah",
                          "id": "what"
                      },
                      "imageId": "ami-0000000",
                      "instanceId": "i-0000000000000",
                      "instanceType": "m4.large",
                      "keyName": "somekey",
                      "launchTime": "2029-06-15T19:39:09.000Z",
                      "monitoring": {
                          "state": "disabled"
                      },
                      "networkInterfaces": [
                          {
                              "attachment": {
                                  "attachTime": "2026-06-15T19:39:09.000Z",
                                  "attachmentId": "eni-attach-00000000",
                                  "deleteOnTermination": true,
                                  "deviceIndex": 0,
                                  "status": "attached"
                              },
                              "description": "",
                              "groups": [
                                  {
                                      "groupId": "sg-000000",
                                      "groupName": "ALL_OPEN"
                                  }
                              ],
                              "ipv6Addresses": [],
                              "macAddress": "xxxxxxxxxxx",
                              "networkInterfaceId": "eni-0000000",
                              "ownerId": "1234556789",
                              "privateDnsName": "blah.com",
                              "privateIpAddress": "10.0.0.0.1",
                              "privateIpAddresses": [
                                  {
                                      "primary": true,
                                      "privateDnsName": "10.0.0.0.1.ec2.internal",
                                      "privateIpAddress": "10.0.0.0.1"
                                  }
                              ],
                              "sourceDestCheck": true,
                              "status": "in-use",
                              "subnetId": "subnet-00000000",
                              "vpcId": "vpc-00000000"
                          }
                      ],
                      "placement": {
                          "availabilityZone": "us-east-1b",
                          "groupName": "",
                          "tenancy": "default"
                      },
                      "platform": "windows",
                      "privateDnsName": "blah.ec2.internal",
                      "privateIpAddress": "10.0.0.0.1",
                      "productCodes": [],
                      "publicDnsName": "",
                      "rootDeviceName": "/dev/sda1",
                      "rootDeviceType": "ebs",
                      "securityGroups": [
                          {
                              "groupId": "sg-0000000",
                              "groupName": "ALL_OPEN"
                          }
                      ],
                      "sourceDestCheck": true,
                      "state": {
                          "code": 16,
                          "name": "running"
                      },
                      "stateTransitionReason": "",
                      "subnetId": "subnet-0000",
                      "tags": [
                          {
                              "key": "created_timestamp",
                              "value": "2029-06-15 19:39:09+00:00"
                          },
                          {
                              "key": "uptime",
                              "value": "0000000"
                          }
                      ],
                      "virtualizationType": "hvm",
                      "vpcId": "vpc-00000"
                  },
                  "supplementaryConfiguration": {},
                  "awsAccountId": "123456789",
                  "configurationItemVersion": "1.3",
                  "awsAccountAlias": "XXXXX",
                  "IndexDate": "2031-06-19T01:02:59Z",
                  "TTL": "2052-06-19T01:32:59Z"
              },
              "fields": null,
              "_explanation": null,
              "matched_queries": null,
              "inner_hits": null,
              "_nested": null
}
```

`$.hits.hits[0]` root fields (`$.hits.hits[0]._index`, `$.hits.hits[0]._id`, etc) are elasticsearch fields showing things like index name, unique resource ID, etc.<br>
`$.hits.hits[0]._source` is the complete resource data as recorded by AWS-config or the harvested resource type.<br>
`$.hits.hits[0]._source.configuration` is the complete resource data as you'd see if you did a describe-* AWS api call.

*NOTE*: The additional fields found in `_source` provided by AWS-config are supplemental information that you wouldn't otherwise get from an AWS `describe-*` API call.  
