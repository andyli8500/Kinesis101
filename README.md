# Kinesis101

## Create a Stream
```
aws kinesis create-stream --stream-name kex --shard-count 1
aws kinesis wait stream-exists --stream-nam kex
```
