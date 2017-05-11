# Kinesis101

### Create a Stream
```
aws kinesis create-stream --stream-name kex --shard-count 1
aws kinesis wait stream-exists --stream-nam kex
```
Note that Estimate the number of shards needs: Avg. Record Size, Maximum Records Written/s, Number of Consuming App

### Create IAM Policy and User
Stream ARN is:
```
arn:aws:kinesis:<region>:<account>:stream/<name>
```
DynamoDB ARN (for KCL):
```
arn:aws:dynamodb:<region>:<account>:table/<name>
```

In `Policy.json`:
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Stmt123",
      "Effect": "Allow",
      "Action": [
        "kinesis:DescribeStream",
        "kinesis:PutRecord",
        "kinesis:PutRecords",
        "kinesis:GetShardIterator",
        "kinesis:GetRecords"
      ],
      "Resource": [
        "arn:aws:kinesis:us-west-2:123:stream/StockTradeStream"
      ]
    },
    {
      "Sid": "Stmt456",
      "Effect": "Allow",
      "Action": [
        "dynamodb:*"
      ],
      "Resource": [
        "arn:aws:dynamodb:us-west-2:123:table/StockTradesProcessor"
      ]
    },
    {
      "Sid": "Stmt789",
      "Effect": "Allow",
      "Action": [
        "cloudwatch:PutMetricData"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
```
