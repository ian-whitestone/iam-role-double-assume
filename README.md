# iam-role-double-assume

## Background

I have an application deployed on an enterprise managed platform. On this platform, they create an EC2 instance for each application. The EC2's instance has an IAM role that has limited permissions and cannot be modified. Users deploying on this system need to "bring their own access". The typical solution for this is to have an IAM user, with a set of credentials stored on the EC2 instance, that can assume an IAM role that has access to do whatever you need.

With this setup, assuming the IAM role is fairly simple. The following needs to be added to your AWS config files:

`~/.aws/config`
```
[profile my_iam_role]
role_arn = <iam_role_arn<
source_profile = my_user

[my_user]
role_arn = <iam_user_arn>
```


`~/.aws/credentials`
```
[my_user]
aws_access_key_id = XXX
aws_secret_access_key = XXX
```

And then you can do things like:

```
import boto3

session = boto3.Session(profile_name="my_iam_role")
client = session.client('s3')

# do stuff with client
```

## The Problem

There is an enterprise data lake, that currently grants access to IAM roles by creating a new IAM role, with access to the required datasets, that they can assume. 

With this setup, I will have an IAM user assuming an IAM role that can assume another IAM role. Not sure if this can be done, hence testing it out in this repo!

*TODO:*
- include picture

## Setting up a Test Environment


### IAM User Policy

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "sts:AssumeRole"
            ],
            "Resource": "*"
        }
    ]
}
```


### IAM Role 1

Attached Policy:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "arn:aws:iam::111222333444:role/IAM_ROLE_2"
        }
    ]
}
```

Trust Policy:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111222333444:user/whitestone"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```


### IAM Role 2

Managed Policy: AmazonS3FullAccess

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "s3:*",
            "Resource": "*"
        }
    ]
}
```

Trust Policy:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111222333444:role/IAM_ROLE_1",
        "Service": "s3.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```


### Bucket Policy


```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::111222333444:role/IAM_ROLE_2"
            },
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::iam-assume-role-test",
                "arn:aws:s3:::iam-assume-role-test/*"
            ]
        }
    ]
}
```


### AWS Config Files


`~/.aws/config`

```
[default]

[profile iam_role_2]
role_arn = arn:aws:iam::111222333444:role/IAM_ROLE_2
source_profile = iam_role_1

[profile iam_role_1]
role_arn = arn:aws:iam::111222333444:role/IAM_ROLE_1
source_profile = iam_user

[iam_user]
role_arn = arn:aws:iam::111222333444:user/whitestone
```


`~/.aws/credentials`

```
[iam_user]
aws_access_key_id = XXX
aws_secret_access_key = XXX
```

## Testing Everything Out

```
>>> touch test.txt
>>> aws --profile iam_role_2 s3 cp test.txt s3://iam-assume-role-test

>>> aws --profile iam_role_1 s3 cp test.txt s3://iam-assume-role-test

```

*TODO:*
- add command output
- add aws cli version
