---
title:   "AWS cli 튜토리얼 - 1"
excerpt: "AWS cli 튜토리얼 - 1"
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

예시를 통해 자주 쓰이는 aws cli 및 option 사용 방법들을 정리해보았습니다.

## Tutorial : [ec2] describe-instances

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
- `instance-state-name` - The state of the instance (`pending` | `running` | `shutting-down` | `terminated` | `stopping` | `stopped` )
- `tag:<key>` - The key/value combination of a tag assigned to the resource. Use the tag key in the filter name and the tag value as the filter value. For example, to find all resources that have a tag with the key `Owner` and the value `TeamA` , specify `tag:Owner` for the filter name and `TeamA` for the filter value.
- (structure)
    - A filter name and value pair that is used to return a more specific list of results from a describe operation. Filters can be used to match a set of resources by specific criteria, such as tags, attributes, or IDs.
    - If you specify multiple filters, the filters are joined with an `AND` , and the request returns only results that match all of the specified filters.
    - Name → (string)
        - The name of the filter. Filter names are case-sensitive.
    - Values -> (list)
        - The filter values. Filter values are case-sensitive. If you specify multiple values for a filter, the values are joined with an `OR` , and the request returns all results that match any of the specified values.
        

Shorthand Syntax:

```bash
Name=string,Values=string,string ...
```

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
aws ec2 describe-instances --instance-ids i-088fdb5b7601cfe82
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
