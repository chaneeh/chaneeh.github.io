---
title:   "AWS cli 튜토리얼 - [ec2]"
excerpt: "AWS cli 튜토리얼 - [ec2]"
toc: true
toc_sticky: true

categories:
  - Blog
tags:
  - Blog
  - AWS cli
  - TOC
last_modified_at: 2023-06-06T12:06:00+09:00
---

# Background

aws cli들은 infra 구축의 자동화를 위해 자주 쓰이는 interface 입니다.

aws cli 및 option 사용 방법들을 정리해보았습니다.

# Tutorial
## [ec2] describe-instances

### **Synopsis**

```bash
aws ec2 describe-instances
[--filters <value>]
[--instance-ids <value>]
[--output <value>]
[--query <value>]
[--region <value>]
...
```

### Options

**`--filters`**

---

- `instance-id` - The ID of the instance
- `instance-state-name` - The state of the instance 
- `tag:<key>` - Use the tag key in the filter name and the tag value as the filter value. 
- (structure)
    - Name → (string)
        - The name of the filter. Filter names are case-sensitive.
    - Values -> (list)
        - The filter values.

**`--instance-ids`**

---

- The instance IDs.
- Default: Describes all your instances.

**`--query`** (global)

---

- A JMESPath query to use in filtering the response data.

### Examples

**Example 1: To describe an instance**

The following `describe-instances` example describes the specified instance.

```bash
aws ec2 describe-instances --instance-ids i-0123456789
```

Output:

```bash
{
    "Reservations": [
        {
            "Groups": [],
            "Instances": [
                {
                    ...
                }
            ],
            "OwnerId": {ownerid},
        }
    ]
}
```

**Example 2: To filter for instances with the specified type and key-name**

The following `describe-instances` example uses filters to scope the results to instances of the specified type.

```bash
aws ec2 describe-instances \
    --filters Name=instance-type,Values=t2.xlarge && Name=key-name,Values=my-pem-key
```

**Example 3: To filter for instances with the specified name tag** 

The following `describe-instances` example uses name tag filters to scope the results to instances of the specified name "K8s-mng-system-v2"

```bash
aws ec2 describe-instances \
    --filters Name=tag:Name,Values=my-prod-server
```

**Example 4: To filter for instances with the specified response columns**

The following `describe-instances` examples display the instance ID, Availability Zone, and the value of the `Name` tag for instances that have a tag with the name `tag-key`, in table format.

```bash
aws ec2 describe-instances \
    --query 'Reservations[*].Instances[*].{Instance:InstanceId,AZ:Placement.AvailabilityZone,Name:Tags[?Key==`Name`]|[0].Value}' \
    --output table
```

Output

```bash
-------------------------------------------------------------
|                     DescribeInstances                     |
+--------------+-----------------------+--------------------+
|      AZ      |       Instance        |        Name        |
+--------------+-----------------------+--------------------+
|  us-east-2b  |  i-057750d42936e468a  |  my-prod-server    |
|  us-east-2a  |  i-001efd250faaa6ffa  |  test-server-1     |
|  us-east-2a  |  i-027552a73f021f3bd  |  test-server-2     |
+--------------+-----------------------+--------------------+
```

**Example 5: To filter for instances with the specified name tag**

```bash
aws ec2 describe-instances \
    --query 'Reservations[*].Instances[*].{Instance:InstanceId, Name:Tags[?Value==`my-prod-server`]|[0].Value}' \
    --output table
```

Output

```bash
----------------------------------------------
|              DescribeInstances             |
+----------------------+---------------------+
|       Instance       |        Name         |
+----------------------+---------------------+
|  i-1234567890abcdefg |  None               |
|  i-11111111aaaaaaaaa |  None               |
|  i-abcdefg1234567890 |  my-prod-server     |
|  i-aaaaaaaa111111111 |  None               |
+----------------------+---------------------+
```

## [ec2] create-instances

### **Synopsis**

```bash
aws ec2 create-instances
[--filters <value>]
[--instance-ids <value>]
[--output <value>]
[--query <value>]
[--region <value>]
...
```


# References
- **AWS cli documents**
    - [https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/describe-instances.html](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/describe-instances.html)
- **JMSEPath query tutorial**
    - [https://jmespath.org/tutorial.html](https://jmespath.org/tutorial.html)