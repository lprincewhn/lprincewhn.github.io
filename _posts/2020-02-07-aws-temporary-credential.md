---
layout: post
title: AWS临时授权-01
key: 20200207
tags: aws security role
modify_date: 2020-02-07
---

AWS的授权本质是通过一组加密的属性完成的，这组属性称为安全凭证(security credential)。credential有两类：长期的和临时的。最常见的长期credential是为一个普通的IAM User创建的AccessKeyID和SecretAccessKey。除非手动失效，否则客户端可以永久使用这个credential访问AWS资源。而短期credential比长期的多一个属性SessionToken，这个属性定义了credential的有效期，有效期过后该credential自动失效。因此，通过临时credential可以实现临时授权，创建“临时账号”。

<!--more-->

## 0. 准备

### 0.1 需求场景

作为安全管理员，我希望最小化普通用户访问存储敏感数据的S3桶的权限，需要访问时临时授权，在指定时间后权限自动失效，以避免权限被滥用。

### 0.2 实验环境

实验架构图如下：

![2020-02-07-flow-1.jpg](http://lprincewhn.github.io/assets/images/2020-02-07-flow-1.jpg)

AWS STS提供了多种方法生成临时credential，本文使用其中的assumeRole方式，这种方式中临时credential的权限由IAM Role定义。实验所涉及的AWS资源如下：

- IAM
    - User：credadmin，管理员，负责生成访问S3资源的临时credential，并且将其存入Secret Manager中，因此需要Secret Manager的写入权限。
    - Role：EC2Normal，附加到EC2实例上的角色，默认情况，EC2实例中运行的程序使用该角色访问其他AWS资源，该角色不包含敏感S3桶的访问权限。
    - Role：SensitiveS3Access，用于访问敏感S3桶的角色。
- Secret Manager
    - Secret：SensitiveS3，用于存储临时credential，其中包括AccessKeyId，SecretAccessKey，SessionToken。
- EC2和S3
    - EC2：用于验证，运行访问敏感S3桶的应用程序。
    - Bucket：sensitive.xxxxxx，用于验证，存放敏感数据的S3桶。

## 1. 实验步骤

### 1.1 创建IAM User

credamin的访问类型为“Programmatic access”，这种类型的用户会自动生成通过API访问AWS资源所需的User credential，即Access Key ID和Secret Access Key，用户创建成功后请记下这两个值。

![2020-02-07-credadmin-1.jpg](http://lprincewhn.github.io/assets/images/2020-02-07-credadmin-1.jpg)

![2020-02-07-credadmin-2.jpg](http://lprincewhn.github.io/assets/images/2020-02-07-credadmin-2.jpg)

### 1.2 创建Secret

按照下图创建一个Secret用于存放临时credential，需要访问敏感数据的S3桶的时候需要从中读取。

![2020-02-07-Secret-1.jpg](http://lprincewhn.github.io/assets/images/2020-02-07-Secret-1.jpg)

### 1.3 创建角色

1. EC2Normal：这是附加到EC2实例上的角色，因此Trusted Entity选择EC2，权限至少需要包含Secret：SensitiveS3的读取权限。

![2020-02-07-EC2Normal-1.jpg](http://lprincewhn.github.io/assets/images/2020-02-07-EC2Normal-1.jpg)

2. SensitiveS3Access：

该角色用于访问敏感S3桶sensitive.xxxxxx，其权限可以在Role Permission中配置，也可以在S3桶的Bucket Policy中配置。本次实验采用后者，此时无需配置任何Policy。

![2020-02-07-SensitiveS3Access-1.jpg](http://lprincewhn.github.io/assets/images/2020-02-07-SensitiveS3Access-1.jpg)

创建时指定Trusted Entity为一个AWS账号，表示只有该AWS账号的Root用户才能使用该角色。手动修改Trusted Entity，指定为前面创建的IAM User：credadmin。
``` json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<aws accound id>:user/credadmin"
      },
      "Action": "sts:AssumeRole",
      "Condition": {}
    }
  ]
}
```

角色创建成功后，其Token的默认有效期为1个小时，可手动修改为其他时间，最大不超过12小时。
![2020-02-07-SensitiveS3Access-3.jpg](http://lprincewhn.github.io/assets/images/2020-02-07-SensitiveS3Access-3.jpg)


### 1.4 创建存放敏感数据的S3桶

S3桶创建完毕后，上传一个文件，如“case.png“，用于测试，修改其Bucket policy，赋予角色SensitiveS3Access完全访问权限。
``` json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::<aws account id>:role/SensitiveS3Access"
            },
            "Action": "s3:*",
            "Resource": "arn:aws:s3:::sensitive.xxxxxx/*"
        }
    ]
}
```

### 1.5 生成临时credential

编写程序，调用API生成角色SensitiveS3Access的临时credential，包括Access Key ID，Secret Access Key和临时credential独有的Session Token。

``` python
import boto3
import json
import pprint
import urllib
import requests

# 使用credamin的User credential访问STS
admin_client = boto3.client(
    'sts',
    aws_access_key_id='<access key id of credadmin>',
    aws_secret_access_key='<secret access key of credadmin>'
)
# 调用STS API：assumeRole获取角色SensitiveS3Access的临时credential
response = admin_client.assume_role(
    RoleArn='arn:aws:iam::<aws account id>:role/SensitiveS3Access',
    RoleSessionName='credadmin-authorize-ec2',
    DurationSeconds=7200
)
temp_credentials = {
    'sessionId': response['Credentials']['AccessKeyId'],
    'sessionKey': response['Credentials']['SecretAccessKey'],
    'sessionToken': response['Credentials']['SessionToken']
}

# 使用临时credential生成控制台访问URL
request_parameters = "?Action=getSigninToken"
request_parameters += "&SessionDuration=7200"
request_parameters += "&Session=" + urllib.parse.quote_plus(json.dumps(temp_credentials))
request_url = "https://signin.aws.amazon.com/federation" + request_parameters
r = requests.get(request_url)
signin_token = json.loads(r.text)
request_parameters = "?Action=login" 
request_parameters += "&Issuer=credadmin" 
request_parameters += "&Destination=" + urllib.parse.quote_plus("https://s3.console.aws.amazon.com/s3/buckets/sensitive.xxxxxx/")
request_parameters += "&SigninToken=" + signin_token["SigninToken"]
request_url = "https://signin.aws.amazon.com/federation" + request_parameters
print(request_url)

# 将临时credential存到Secret Manager
sm_client = boto3.client('secretsmanager')
response = sm_client.put_secret_value(
    SecretId='SensitiveS3',
    SecretString=json.dumps(temp_credentials)
)
```

执行上述程序后，将输出一个用于控制台访问的URL，并可以在Secret Manager中看到生成的临时credential。

### 1.6 访问控制台URL

在浏览器中输入上一步中生成的URL，可直接登陆到AWS控制台，留意左上方的登陆信息，可以看到此时登陆的身份为“SensitiveS3Access/credadmin-authorize-ec2”。

![2020-02-07-console-1.jpg](http://lprincewhn.github.io/assets/images/2020-02-07-console-1.jpg)

SensitiveS3Access是当前使用的角色，credadmin-authorize-ec2是调用assumeRole时指定的RoleSessionName参数，可作为该临时credential的用户名，作审计使用。

![2020-02-07-cloudtrail-1.jpg](http://lprincewhn.github.io/assets/images/2020-02-07-cloudtrail-1.jpg)

### 1.7 启动EC2，附加默认角色EC2Normal

![2020-02-07-EC2-1.jpg](http://lprincewhn.github.io/assets/images/2020-02-07-EC2-1.jpg)

### 1.8 在EC2上运行访问S3桶的程序

``` python
import json
import boto3
import pprint
# 直接访问S3，使用的角色是EC2Normal，将抛出权限异常。
s3_client = boto3.client(
    's3'
)
try:
    response = s3_client.get_object_acl(
        Bucket='sensitive.xxxxxx',
        Key='case.png'
    )
except:
    pprint.pprint("由于没有权限导致异常。")
# 从Secret Manager读取访问S3需要的临时credential
sm_client = boto3.client('secretsmanager')
response = sm_client.get_secret_value(
    SecretId='SensitiveS3',
)
credentials = json.loads(response['SecretString'])
pprint.pprint(credentials)
# 用临时credential访问S3，使用的角色是SensitiveS3Access，成功。
s3_client = boto3.client(
    's3',
    aws_access_key_id=credentials['sessionId'],
    aws_secret_access_key=credentials['sessionKey'],
    aws_session_token=credentials['sessionToken'],
)
response = s3_client.get_object_acl(
    Bucket='sensitive.xxxxxx',
    Key='case.png'
)
pprint.pprint(response)
```

由于TOKEN有效期被配置成了两小时，两小时内运行以上程序将可以获取到对象的ACL。如果两小时内没有重新生成临时credentia了，两小时后重新运行以上程序，则会报“Token过期”错误。
```
botocore.exceptions.ClientError: An error occurred (ExpiredToken) when calling the GetObjectAcl operation: The provided token has expired.
```

## 2. 几点思考
1. 如何翻译AssumeRole：申请另一个角色的权限。
2. 可否不使用SecretManager？可以，但SecretManger是一个安全，合理的credential的存放位置。
3. Secret的访问权限不够细致，EC2Normal应该只能读取他需要的Secret，无需整个SecretManager的读写权限。
4. 如何提高credadmin的长期credential的安全性？不使用User，而是创建一个相同权限的Role，将这个Role附加到特定的EC2实例上，在这个实例上部署生成临时credential的程序。