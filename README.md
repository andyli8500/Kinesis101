# Kinesis101

## Streams
### Create a Stream
```bash
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
```bash
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

## Kinesis Analytics
```
aws kinesisanalytics create-application --application-name kex-analytics --region us-east-1
```
Configure the source stream in the console

1. Tumbling window - count the buy/sell within every 10 seconds
```sql
CREATE OR REPLACE STREAM "DESTINATION_SQL_STREAM_tw" (tradetype VARCHAR(4), trade_count INTEGER);
-- Create a pump which continuously selects from a source stream (SOURCE_SQL_STREAM_001)
-- performs an aggregate count that is grouped by columns ticker over a 10-second tumbling window
-- and inserts into output stream (DESTINATION_SQL_STREAM)
CREATE OR REPLACE  PUMP "STREAM_PUMP_tw" AS INSERT INTO "DESTINATION_SQL_STREAM_tw"
-- Aggregate function COUNT|AVG|MAX|MIN|SUM|STDDEV_POP|STDDEV_SAMP|VAR_POP|VAR_SAMP)
SELECT STREAM tradetype, COUNT(*) AS trade_count
FROM "SOURCE_SQL_STREAM_001"
-- Uses a 10-second tumbling time window
GROUP BY tradetype, FLOOR(("SOURCE_SQL_STREAM_001".ROWTIME - TIMESTAMP '1970-01-01 00:00:00') SECOND / 10 TO SECOND);
```

2. a. Sliding window - count the buy/sell in every 10 seconds (overlapped)
```sql
CREATE OR REPLACE STREAM "DESTINATION_SQL_STREAM_sw" (tradetype VARCHAR(4), trade_count INTEGER);
-- Create a pump which continuously selects from a source stream (SOURCE_SQL_STREAM_001)
-- performs an aggregate count that is grouped by columns ticker over a 10-second sliding window
CREATE OR REPLACE PUMP "STREAM_PUMP_sw" AS INSERT INTO "DESTINATION_SQL_STREAM_sw"
-- COUNT|AVG|MAX|MIN|SUM|STDDEV_POP|STDDEV_SAMP|VAR_POP|VAR_SAMP)
SELECT STREAM tradetype, COUNT(*) OVER FIVE_SECOND_SLIDING_WINDOW AS trade_count
FROM "SOURCE_SQL_STREAM_001"
-- Results partitioned by ticker_symbol and a 10-second sliding time window 
WINDOW FIVE_SECOND_SLIDING_WINDOW AS (
  PARTITION BY tradetype
  RANGE INTERVAL '5' SECOND PRECEDING);
```
2. b. 2 Sliding window - count the buy/sell in last 10 rows and 5 rows (overlapped)
```sql
CREATE OR REPLACE STREAM "DESTINATION_SQL_STREAM_sw2" (tradetype VARCHAR(4), trade_count_5 INTEGER, avg_price_5 DOUBLE, trade_count_10 INTEGER, avg_price_10 DOUBLE);
-- Create a pump which continuously selects from a source stream (SOURCE_SQL_STREAM_001)
-- performs an aggregate count that is grouped by columns ticker over a 10-second sliding window
CREATE OR REPLACE PUMP "STREAM_PUMP_sw2" AS INSERT INTO "DESTINATION_SQL_STREAM_sw2"
-- COUNT|AVG|MAX|MIN|SUM|STDDEV_POP|STDDEV_SAMP|VAR_POP|VAR_SAMP)
SELECT STREAM tradetype, COUNT(*) OVER FIVE_ROWS_SLIDING_WINDOW AS trade_count_5, AVG(price) OVER FIVE_ROWS_SLIDING_WINDOW AS avg_price_5,
              COUNT(*) OVER TEN_ROWS_SLIDING_WINDOW AS trade_count_10, AVG(price) OVER TEN_ROWS_SLIDING_WINDOW AS avg_price_10
FROM "SOURCE_SQL_STREAM_001"
-- Results partitioned by ticker_symbol and a 10-second sliding time window 
WINDOW 
FIVE_ROWS_SLIDING_WINDOW AS (
  PARTITION BY tradetype
  ROWS 5 PRECEDING),
TEN_ROWS_SLIDING_WINDOW AS (
  PARTITION BY tradetype
  ROWS 10 PRECEDING);
```

