# S3 DataSync

> 차량 데이터가 저장된 S3 (source-s3-bucket) 에서 Another 환경에 구축된 S3 (destination-s3-bucket) 로의 DataSync 작업 필요



첨부 문서

- [How to use AWS DataSync to migrate data between Amazon S3 buckets](https://aws.amazon.com/ko/blogs/storage/how-to-use-aws-datasync-to-migrate-data-between-amazon-s3-buckets/)
- [Tutorial: Transferring Data from Amazon S3 to an Amazon S3 AWS Account](https://docs.aws.amazon.com/ko_kr/datasync/latest/userguide/tutorial_s3-s3-cross-account-transfer.html)



## Architecture

<img width="694" alt="스크린샷 2023-11-08 오후 9 49 32" src="https://github.com/younggwon1/service-monitoring/assets/108652621/73b371a2-93cd-4a06-ae7e-866f73214252">



- Account A : 111111111111
  - Amazone S3 Bucket : source-s3-bucket
- Account B : 222222222222
  - Amazone S3 Bucket : destination-s3-bucket



## Resource List

### **Source**

- Source S3

  - Name : source-s3-bucket

- S3 Bucket IAM Role

  - AWSDataSyncS3BucketAccess-source-s3-bucket

    - 신뢰 관계

      ~~~Yaml
      {
          "Version": "2012-10-17",
          "Statement": [
              {
                  "Effect": "Allow",
                  "Principal": {
                      "Service": "datasync.amazonaws.com"
                  },
                  "Action": "sts:AssumeRole",
                  "Condition": {
                      "StringEquals": {
                          "aws:SourceAccount": "111111111111"
                      },
                      "ArnLike": {
                          "aws:SourceArn": "arn:aws:datasync:ap-northeast-2:111111111111:*"
                      }
                  }
              }
          ]
      }
      ~~~

    - 권한

      ~~~yaml
      {
          "Version": "2012-10-17",
          "Statement": [
              {
                  "Action": [
                      "s3:GetBucketLocation",
                      "s3:ListBucket",
                      "s3:ListBucketMultipartUploads"
                  ],
                  "Effect": "Allow",
                  "Resource": "arn:aws:s3:::source-s3-bucket"
              },
              {
                  "Action": [
                      "s3:AbortMultipartUpload",
                      "s3:DeleteObject",
                      "s3:GetObject",
                      "s3:ListMultipartUploadParts",
                      "s3:PutObjectTagging",
                      "s3:GetObjectTagging",
                      "s3:PutObject"
                  ],
                  "Effect": "Allow",
                  "Resource": "arn:aws:s3:::source-s3-bucket/*"
              }
          ]
      }
      ~~~

  - source-s3-bucket_to_destination-s3-bucket

    - 신뢰 관계

      ~~~yaml
      {
          "Version": "2012-10-17",
          "Statement": [
              {
                  "Effect": "Allow",
                  "Principal": {
                      "Service": "datasync.amazonaws.com"
                  },
                  "Action": "sts:AssumeRole"
              }
          ]
      }
      ~~~

    - 권한

      ~~~yaml
      {
          "Version": "2012-10-17",
          "Statement": [
              {
                  "Action": [
                      "s3:GetBucketLocation",
                      "s3:ListBucket",
                      "s3:ListBucketMultipartUploads"
                  ],
                  "Effect": "Allow",
                  "Resource": "arn:aws:s3:::destination-s3-bucket"
              },
              {
                  "Action": [
                      "s3:AbortMultipartUpload",
                      "s3:DeleteObject",
                      "s3:GetObject",
                      "s3:ListMultipartUploadParts",
                      "s3:PutObjectTagging",
                      "s3:GetObjectTagging",
                      "s3:PutObject"
                  ],
                  "Effect": "Allow",
                  "Resource": "arn:aws:s3:::destination-s3-bucket/*"
              }
          ]
      }
      ~~~

- DataSync

  - Name : source-s3-bucket_to_destination-s3-bucket

    <img width="837" alt="스크린샷 2023-11-08 오후 9 59 25" src="https://github.com/younggwon1/service-monitoring/assets/108652621/4941e6b6-0d80-476a-93ef-be3133d0a39f">



### Destination

- Destination S3

  - Name : destination-s3-bucket

  - S3 Bucket Policy

    ~~~Yaml
    {
        "Version": "2008-10-17",
        "Statement": [
            {
                "Sid": "DataSyncCreateS3LocationAndTaskAccess",
                "Effect": "Allow",
                "Principal": {
                    "AWS": "arn:aws:iam::111111111111:role/source-s3-bucket_to_destination-s3-bucket"
                },
                "Action": [
                    "s3:GetBucketLocation",
                    "s3:ListBucket",
                    "s3:ListBucketMultipartUploads",
                    "s3:AbortMultipartUpload",
                    "s3:DeleteObject",
                    "s3:GetObject",
                    "s3:ListMultipartUploadParts",
                    "s3:PutObject",
                    "s3:GetObjectTagging",
                    "s3:PutObjectTagging"
                ],
                "Resource": [
                    "arn:aws:s3:::destination-s3-bucket",
                    "arn:aws:s3:::destination-s3-bucket/*"
                ]
            }
        ]
    }
    ~~~

    



### **Requirement**

1. D-1 데이터만 개발 S3 에 올라올 수 있도록 하면 좋을 것 같다.
   1. 개발 S3 Lifecycle → 하루 치 데이터만 남길 수 있도록..
   2. DataSync 과정에서 하루치 데이터만 넘길 수는 없을까??
      1. 전체를 넘기기에는 사이즈가 크다..



#### Reflection of requirements

DataSync 의 Include Patterns 에 이관할 S3 Object 에 대해 명시를 해줘야 해당 Object 만 이관된다.

- EX)

  Include patterns

  - /topics/project1/year=2023/month=11/day=06/*
  - /topics/project2/year=2023/month=11/day=06/*
  - /topics/project3/year=2023/month=11/day=06/*
  - /topics/project4/year=2023/month=11/day=06/*
  - /topics/project5/year=2023/month=11/day=06/*

하지만 업데이트를 수작업으로 해야하기 때문에 번거로움이 존재한다.



따라서 `Lambda Function` 을 활용하여 Include patterns 을 업데이트하는 로직을 구현한다.

- 구현 중 이슈 사항

  - timestamp 를 활용한 Include Patterns 설정

    1. 하루치의 데이터만 마이그레이션하기 위해 DataSync의 Include patterns 에서 `"/topics/project/year=${timestamp:yyyy}/month=${timestamp:MM}/day=${timestamp:dd}/"` *또는* `"/topics/project/year= !{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/"` 등을 시도했지만 동작하지 않는 것 같다.
    2. 답변 : timestamp 변수를 활용한 패턴은 가능하지 않다.
       1. <https://docs.aws.amazon.com/datasync/latest/userguide/filtering.html>

  - 코드 구현을 통한 Include Patterns 설정

    1. datasync 의 Include patterns 를 업데이트하여 하루 전날의 object 를 migration 해보기 위해 코드를 구현하였지만, Includes 의 length 가 Maximum 이 1이어서 아래와 같이 여러 개의 값을 설정할 수 없었다. 혹시 아래와 같은 기능을 동작하기 위해서는 어떻게 패턴을 작성해야하는지 궁금.

       ~~~yaml
       # EX) 11월 3일에 DataSync 가 동작할 경우의 Migration 할 대상 설정
       datasync_client = boto3.client('datasync')
       try:
           response = datasync_client.update_task(
               TaskArn=taskArn,
               Includes=[
                   {
                       'FilterType': 'SIMPLE_PATTERN', 
                       'Value': "/topics/project1/year=2023/month=11/day=02/*"
                   },
                   {
                       'FilterType': 'SIMPLE_PATTERN', 
                       'Value': "/topics/project2/year=2023/month=11/day=02/*"
                   },
                   {
                       'FilterType': 'SIMPLE_PATTERN', 
                       'Value': "/topics/project3/year=2023/month=11/day=02/*"
                   }
               ]
           )
           logger.info(response)
       except ClientError as e:
           logger.exception(e)
           raise
       ~~~

    2. 답변

       - Include patterns 의 length 가 Maximum 이 1이기에 여러줄로 만들 수 없다. 따라서 Value를 pipeline **(|)**으로 구분하여 한줄에 여러 개의 Include patterns를 설정해야한다. 예시 문서 [1],[2]를 참고.
         - [1] <https://docs.aws.amazon.com/datasync/latest/userguide/filtering.html#include-filters>
         - [2] <https://aws.amazon.com/ko/blogs/storage/excluding-and-including-specific-data-in-transfer-tasks-using-aws-datasync-filters/>

       ~~~yaml
       Includes=[
           {
               'FilterType': 'SIMPLE_PATTERN', 
               'Value': "/topics/project1/year=2023/month=11/day=02/*|/topics/project2/year=2023/month=11/day=02/*|/topics/project3/year=2023/month=11/day=02/*"
           }
       ]
       ~~~






~~~python
import json
import boto3
import logging
from datetime import datetime
from dateutil.relativedelta import relativedelta
from botocore.exceptions import ClientError

def lambda_handler(event, context):
    """
    MEMO: 데이터가 적재된 S3 -> Another 환경 S3 로 Migration 하는 함수
    Source S3 : source-s3-bucket
    Destination S3 : Destination-s3-bucket
    DataSync : source-s3-bucket_to_destination-s3-bucket
    """
    
    logger = logging.getLogger(__name__)
    
    # 1. Trigger 되는 날짜 하루 전 날짜를 추출
    before_one_day = datetime.today() - relativedelta(days=1)
    
    # 2. source-s3-bucket/topics/* Object 추출
    date_format = "year="+before_one_day.strftime("%Y")+"/month="+before_one_day.strftime("%m")+"/day="+before_one_day.strftime("%d")+"/*"
    
    s3_client = boto3.client("s3")
    try:
        response = s3_client.list_objects_v2(Bucket='Destination-s3-bucket', Prefix='topics/', Delimiter = '/')
    except ClientError as e:
        logger.exception(e)
        raise
    else:
        prefixes = ["/" + prefix['Prefix'].strip() + date_format for prefix in response['CommonPrefixes']]
        result_include_patterns = '|'.join(prefixes)
    
    # 3. 추출한 날짜를 기반으로 S3 객체 Pattern 에 맞게 DataSync Include Pattern Update
    taskArn = "arn:aws:datasync:ap-northeast-2:111111111111:task/task-0xxxxxxxxxxxxxx"
    
    datasync_client = boto3.client('datasync')
    try:
        response = datasync_client.update_task(
            TaskArn=taskArn,
            Includes=[
                {
                    'FilterType': 'SIMPLE_PATTERN', 
                    'Value': result_include_patterns
                }
            ]
        )
        logger.info(response)
    except ClientError as e:
        logger.exception(e)
        raise
~~~

