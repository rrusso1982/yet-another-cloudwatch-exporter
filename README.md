# YACE - yet another cloudwatch exporter [![Docker Image](https://quay.io/repository/invisionag/yet-another-cloudwatch-exporter/status?token=58e4108f-9e6f-44a4-a5fd-0beed543a271 "Docker Repository on Quay")](https://quay.io/repository/invisionag/yet-another-cloudwatch-exporter)

## Project Status

YACE is currently in quick iteration mode. Things will probably break in upcoming versions. However, it has been in production use at InVision AG for a couple of months already.

## Features

* Stop worrying about your AWS IDs - Auto discovery of resources via tags
* Structured JSON logging
* Filter monitored resources via regex
* Automatic adding of tag labels to metrics
* Automatic adding of dimension labels to metrics
* Allows to export 0 even if CloudWatch returns nil
* Allows exports metrics with CloudWatch timestamps (disabled by default)
* Static metrics support for all cloudwatch metrics without auto discovery
* Pull data from multiple AWS accounts using cross-account roles
* Supported services with auto discovery through tags:

  * alb - Application Load Balancer
  * appsync - AppSync
  * cf - Cloud Front
  * dynamodb - NoSQL Online Datenbank Service
  * ebs - Elastic Block Storage
  * ec - ElastiCache
  * ec2 - Elastic Compute Cloud
  * ecs-svc - Elastic Container Service (Service Metrics)
  * ecs-containerinsights - ECS/ContainerInsights (Fargate metrics)
  * efs - Elastic File System
  * elb - Elastic Load Balancer
  * emr - Elastic MapReduce
  * es - ElasticSearch
  * fsx - FSx File System
  * kinesis - Kinesis Data Stream
  * ngw - Nat Gateway
  * lambda - Lambda Functions
  * nlb - Network Load Balancer
  * rds - Relational Database Service
  * r53r - Route53 Resolver
  * s3 - Object Storage
  * sqs - Simple Queue Service
  * tgw - Transit Gateway
  * vpn - VPN connection
  * asg - Auto Scaling Group
  * kafka - Managed Apache Kafka
  * firehose - Managed Streaming Service
  * sns - Simple Notification Service
  * sfn - Step Functions

## Image

* `quay.io/invisionag/yet-another-cloudwatch-exporter:x.x.x` e.g. 0.5.0
* See [Releases](https://github.com/ivx/yet-another-cloudwatch-exporter/releases) for binaries

## Configuration

### Top level configuration

| Key          | Description                                              |
| ------------ | -------------------------------------------------------- |
| discovery    | Auto-discovery configuration                             |
| static       | List of static configurations                            |
| translations | A list of name -> value mapping for output compatibility |

### Translations configuration

| Key      | Description                             |
| -------- | --------------------------------------- |
| services | A map of service names -> output values |
| suffixes | A map of suffix translations -> output values |

Suffix translations are currently used for alb:TargetGroup,LoadBalancer and elb:AvailabilityZone

### Auto-discovery configuration

| Key                   | Description                                       |
| --------------------- | ------------------------------------------------- |
| exportedTagsOnMetrics | List of tags per service to export to all metrics |
| jobs                  | List of auto-discovery jobs                       |

exportedTagsOnMetrics example:

```yaml
exportedTagsOnMetrics:
  ec2:
    - Name
    - type
```

### Auto-discovery job

| Key                  | Description                                                                                |
| -------------------- | ------------------------------------------------------------------------------------------ |
| regions              | List of AWS regions                                                                        |
| type                 | Service name, e.g. "ec2", "s3", etc.                                                       |
| length (Default 120) | How far back to request data for in seconds                                                |
| delay                | If set it will request metrics up until `current_time - d
| type                 | Service name, e.g. "ec2", "s3", etc.                                                       |
| roleArn              | IAM role to assume (optional)                                                              |
| searchTags           | List of Key/Value pairs to use for tag filtering (all must match), Value can be a regex.   |
| metrics              | List of metric definitions                                                                 |
| additionalDimensions | List of dimensions to return beyond the default list per service                           |

searchTags example:

```yaml
searchTags:
  - Key: env
    Value: production
```

### Metric definition

| Key                    | Description |
| ---------------------- | -------------------------------------------------------------------------------- |
| name                   | CloudWatch metric name                                                           |
| statistics             | List of statistic types, e.g. "Minimum", "Maximum", etc.                         |
| period                 | Statistic period in seconds                                                      |
| length                 | How far back to request data for in seconds(for static jobs)                     |
| delay                  | If set it will request metrics up until `current_time - delay`(for static jobs)  |
| nilToZero              | Return 0 value if Cloudwatch returns no metrics at all                           |
| addCloudwatchTimestamp | Export the metric with the original CloudWatch timestamp                         |

* **Watch out using `addCloudwatchTimestamp` for sparse metrics, e.g from S3, since Prometheus won't scrape metrics containing timestamps older than 2-3 hours**

### Static configuration

| Key        | Description                                                |
| ---------- | ---------------------------------------------------------- |
| regions    | List of AWS regions                                        |
| roleArn    | IAM role to assume                                         |
| namespace  | CloudWatch namespace                                       |
| name       | Must be set with multiple block definitions per namespace  |
| customTags | Custom tags to be added as a list of Key/Value pairs       |
| dimensions | CloudWatch metric dimensions as a list of Name/Value pairs |
| metrics    | List of metric definitions                                 |

### Example of config File

```yaml
translations:
  services:
    ec: elasticache
discovery:
  exportedTagsOnMetrics:
    ec2:
      - Name
    ebs:
      - VolumeId
  jobs:
  - type: es
    regions:
      - eu-west-1
    searchTags:
      - Key: type
        Value: ^(easteregg|k8s)$
    metrics:
      - name: FreeStorageSpace
        statistics:
        - Sum
        period: 600
        length: 60
      - name: ClusterStatus.green
        statistics:
        - Minimum
        period: 600
        length: 60
      - name: ClusterStatus.yellow
        statistics:
        - Maximum
        period: 600
        length: 60
      - name: ClusterStatus.red
        statistics:
        - Maximum
        period: 600
        length: 60
  - type: elb
    regions:
      - eu-west-1
    length: 900
    delay: 120
    awsDimensions:
     - AvailabilityZone
    searchTags:
      - Key: KubernetesCluster
        Value: production-19
    metrics:
      - name: HealthyHostCount
        statistics:
        - Minimum
        period: 600
        length: 600 #(this will be ignored)
      - name: HTTPCode_Backend_4XX
        statistics:
        - Sum
        period: 60
        length: 900 #(this will be ignored)
        delay: 300 #(this will be ignored)
        nilToZero: true
  - type: alb
    regions:
      - eu-west-1
    searchTags:
      - Key: kubernetes.io/service-name
        Value: .*
    metrics:
      - name: UnHealthyHostCount
        statistics: [Maximum]
        period: 60
        length: 600
  - type: vpn
    regions:
      - eu-west-1
    searchTags:
      - Key: kubernetes.io/service-name
        Value: .*
    metrics:
      - name: TunnelState
        statistics:
        - p90
        period: 60
        length: 300
  - type: kinesis
    regions:
      - eu-west-1
    metrics:
      - name: PutRecords.Success
        statistics:
        - Sum
        period: 60
        length: 300
  - type: s3
    regions:
      - eu-west-1
    searchTags:
      - Key: type
        Value: public
    metrics:
      - name: NumberOfObjects
        statistics:
          - Average
        period: 86400
        length: 172800
        additionalDimensions:
          - name: StorageType
            value: AllStorageTypes
      - name: BucketSizeBytes
        statistics:
          - Average
        period: 86400
        length: 172800
        additionalDimensions:
          - name: StorageType
            value: StandardStorage
  - type: ebs
    regions:
      - eu-west-1
    searchTags:
      - Key: type
        Value: public
    metrics:
      - name: BurstBalance
        statistics:
        - Minimum
        period: 600
        length: 600
        addCloudwatchTimestamp: true
  - type: kafka
    regions:
      - eu-west-1
    searchTags:
      - Key: env
        Value: dev
    awsDimensions:
      - Broker ID
      - Topic
    metrics:
      - name: BytesOutPerSec
        statistics:
        - Average
        period: 600
        length: 600
static:
  - namespace: AWS/AutoScaling
    name: must_be_set
    regions:
      - eu-west-1
    dimensions:
     - name: AutoScalingGroupName
       value: Test
    customTags:
      - Key: CustomTag
        Value: CustomValue
    metrics:
      - name: GroupInServiceInstances
        statistics:
        - Minimum
        period: 60
        length: 300
```

## Metrics Examples

```text
### Metrics with exportedTagsOnMetrics
aws_ec2_cpuutilization_maximum{dimension_InstanceId="i-someid", name="arn:aws:ec2:eu-west-1:472724724:instance/i-someid", tag_Name="jenkins"} 57.2916666666667

### Info helper with tags
aws_elb_info{name="arn:aws:elasticloadbalancing:eu-west-1:472724724:loadbalancer/a815b16g3417211e7738a02fcc13bbf9",tag_KubernetesCluster="production-19",tag_Name="",tag_kubernetes_io_cluster_production_19="owned",tag_kubernetes_io_service_name="nginx-ingress/private-ext",region="eu-west-1"} 0
aws_ec2_info{name="arn:aws:ec2:eu-west-1:472724724:instance/i-someid",tag_Name="jenkins"} 0

### Track cloudwatch requests to calculate costs
yace_cloudwatch_requests_total 168
```

## Query Examples without exportedTagsOnMetrics

```text
# CPUUtilization + Name tag of the instance id - No more instance id needed for monitoring
aws_ec2_cpuutilization_average + on (name) group_left(tag_Name) aws_ec2_info

# Free Storage in Megabytes + tag Type of the elasticsearch cluster
(aws_es_free_storage_space_sum + on (name) group_left(tag_Type) aws_es_info) / 1024

# Add kubernetes / kops tags on 4xx elb metrics
(aws_elb_httpcode_backend_4_xx_sum + on (name) group_left(tag_KubernetesCluster,tag_kubernetes_io_service_name) aws_elb_info)

# Availability Metric for ELBs (Sucessfull requests / Total Requests) + k8s service name
# Use nilToZero on all metrics else it won't work
((aws_elb_request_count_sum - on (name) group_left() aws_elb_httpcode_backend_4_xx_sum) - on (name) group_left() aws_elb_httpcode_backend_5_xx_sum) + on (name) group_left(tag_kubernetes_io_service_name) aws_elb_info

# Forecast your elasticsearch disk size in 7 days and report metrics with tags type and version
predict_linear(aws_es_free_storage_space_minimum[2d], 86400 * 7) + on (name) group_left(tag_type, tag_version) aws_es_info

# Forecast your cloudwatch costs for next 32 days based on last 10 minutes
# 1.000.000 Requests free
# 0.01 Dollar for 1.000 GetMetricStatistics Api Requests (https://aws.amazon.com/cloudwatch/pricing/)
((increase(yace_cloudwatch_requests_total[10m]) * 6 * 24 * 32) - 100000) / 1000 * 0.01
```

## IAM

The following IAM permissions are required for YACE to work.

```json
"tag:GetResources",
"cloudwatch:GetMetricData",
"cloudwatch:GetMetricStatistics",
"cloudwatch:ListMetrics"
```

## Running locally

```shell
docker run -d --rm -v $PWD/credentials:/exporter/.aws/credentials -v $PWD/config.yml:/tmp/config.yml \
-p 5000:5000 --name yace quay.io/invisionag/yet-another-cloudwatch-exporter:vx.xx.x # release version as tag - Do not forget the version 'v'

```

## Kubernetes Installation

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: yace
data:
  config.yml: |-
    ---
    # Start of config file
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: yace
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: yace
    spec:
      containers:
      - name: yace
        image: quay.io/invisionag/yet-another-cloudwatch-exporter:vx.x.x # release version as tag - Do not forget the version 'v'
        imagePullPolicy: IfNotPresent
        args:
          - "--config.file=/tmp/config.yml"
        ports:
        - name: app
          containerPort: 5000
        volumeMounts:
        - name: config-volume
          mountPath: /tmp
      volumes:
      - name: config-volume
        configMap:
          name: yace
```
## Options
### Requests concurrency
The flags 'cloudwatch-concurrency' and 'tag-concurrency' define the number of concurrent request to cloudwatch metrics and tags. Their default value is 5.

Setting a higher value makes faster scraping times but can incur in throttling and the blocking of the API.

### Decoupled scraping
The flag 'decoupled-scraping' makes the exporter to scrape Cloudwatch metrics in background in fixed intervals, in stead of each time that the '/metrics' endpoint is fetched. This protects from the abuse of API requests that can cause extra billing in AWS account. This flag is activated by default.

If the flag 'decoupled-scraping' is activated, the flag 'scraping-interval' defines the seconds between scrapes. Its default value is 300.

## Troubleshooting / Debugging

### Help my metrics are intermittent

* Please, try out a bigger length e.g. for elb try out a length of 600 and a period of 600. Then test how low you can
go without losing data. ELB metrics on AWS are written every 5 minutes (300) in default.

### My metrics only show new values after 5 minutes

* Please, try to set a lower value for the 'scraping-interval' flag or set the 'decoupled-scraping' to false.

## Contribute

[Development Setup / Guide](/CONTRIBUTE.md)

# Thank you

* [Justin Santa Barbara](https://github.com/justinsb) - For telling me about AWS tags api which simplified a lot - Thanks!
* [Brian Brazil](https://github.com/brian-brazil) - Who gave a lot of feedback regarding UX and prometheus lib - Thanks!
