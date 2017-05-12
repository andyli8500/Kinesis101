# Kinesis101

## Streams
### Create a Stream
```
aws kinesis create-stream --stream-name kex --shard-count 1
aws kinesis wait stream-exists --stream-nam kex
```
Note that Estimate the number of shards needs: Avg. Record Size, Maximum Records Written/s, Number of Consuming App

### (Optional) Create IAM Policy and User
Stream ARN is:
```
arn:aws:kinesis:<region>:<account>:stream/<name>
```
DynamoDB ARN (for KCL):
```
arn:aws:dynamodb:<region>:<account>:table/<name>
```
Create `Policy.json`:
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
Create policy:
```
aws iam create-policy --policy-name kexPolicy --policy-document file://policy.json
{
    ...
    "Arn": "arn:aws:iam::<account>:policy/kexPolicy"
    ...
}
```

Create User:
```
aws iam create-user --user-name kexUser
```

Create Credential for the User in the console

Attach policy:
```
aws iam attach-user-policy --user-name kexUser --policy-arn arn:aws:iam::<account>:policy/kexPolicy
```

### Put trading data in Kinesis Streams
The sample trading records looks:
```
{"tickerSymbol": "AMZN", 
 "tradeType": "BUY", 
 "price": 395.87,
 "quantity": 16, 
 "id": 3567129045}
```

put_record.sh
```
#!/bin/bash

TickerSymbol=( "AMZN-1" "AMZN-2" "AMZN-3" "AMZN-4"
               "AMZN-5" "AMZN-6" "AMZN-7" "AMZN-8"
               "AMZN-9" "AMZN-10" "AMZN-11" "AMZN-12" )

TradeType=( "BUY" "SELL" )

for i in {0..1000000}
do
    id=$i
    quantity=$(shuf -i 1-1000 -n 1)
    single_price=$(shuf -i 1-1000 -n 1)
    price=$(expr $single_price \* $quantity)

    t_1=${#TickerSymbol[@]}
    idx=$(($RANDOM % $t_1))
    symbol=${TickerSymbol[$idx]}

    t_2=${#TradeType[@]}
    idx=$(($RANDOM % $t_2))
    ttype=${TradeType[$idx]}

    data="{
            tickerSymbol: $symbol, 
            tradeType: $ttype,
            price: $price,
            quantity: $quantity,
            id: $id
}"
    echo "$data"
    # put record
    aws kinesis put-record --stream-name kex --data "$data" --partition-key $id
done
```

## Firehose
change the `put-record` above to firehose
```
...
aws firehose put-record --delivery-stream-name kex-fh --record Data="'$data'" --region us-east-1
...
```
