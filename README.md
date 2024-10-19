# Database read-replicas scaling up/down

- AWS RDS does not use Auto Scaling Groups like EC2 does, so you don’t need to define a Launch Configuration or an Auto Scaling Group to scale RDS instances.
- Instead, RDS has a specific mechanism to manage read replicas that operates independently of EC2-style auto scaling groups.
- For RDS read replicas, auto-scaling happens through the creation of replicas based on CloudWatch alarms

# RDS DB Instance 

```ruby
resource "aws_db_subnet_group" "db_subnet_group" {
  name       = "main"
  subnet_ids = [aws_subnet.subnet.id, aws_subnet.subnet2.id]

  tags = {
    Name = "DB subnet group"
  }
}

resource "aws_db_parameter_group" "postgres" {
  name   = "my-pg"
  family = "postgres16"

  parameter {
    name  = "log_connections"
    value = "1"
  }
}

resource "aws_db_instance" "postgres" {
  identifier             = "postgres-db"
  engine                 = "postgres"
  instance_class         = "db.t3.micro"
  allocated_storage      = 20
  db_name                = var.db_name
  db_subnet_group_name   = aws_db_subnet_group.db_subnet_group.name
  username               = var.db_username
  password               = var.db_password
  parameter_group_name   = aws_db_parameter_group.postgres.name
  skip_final_snapshot    = true
  publicly_accessible    = false
  vpc_security_group_ids = [aws_security_group.db_security_group.id]
}
```

# Solutions

## Amazon Aurora

There’s built-in support for Aurora Auto Scaling, which will automatically add or remove read replicas based on your database’s load without any manual intervention or need for extra setup.

## CloudWatch Alarms

- You can create CloudWatch alarms that monitor the CPU, connections, or other metrics on your primary database.
- When these metrics exceed a threshold, you can trigger **AWS RDS Autoscaling events** -> **AWS Lambda functions** to add or remove read replicas.

![Screenshot 2024-10-19 at 21 20 05](https://github.com/user-attachments/assets/c44976fb-53b1-4c88-9abc-f703132917c5)

### AWS Lambda function - Steps

- To automate scaling with Lambda, you could use a Lambda function that triggers on CloudWatch Alarms **(such as CPU usage crossing 75%)** to add more replicas.
- Trigger a Lambda function that uses the `aws_db_instance` Terraform resource to add read replicas when the alarm triggers.

### Summary

- CloudWatch alarms detect high CPU usage.
- SNS topics trigger the Lambda function.
- Lambda manages the scaling of read replicas by creating and deleting them as needed based on traffic spikes or reductions.

1. Setup CloudWatch Alarm to check on CPU usage:

```ruby
resource "aws_cloudwatch_metric_alarm" "high_cpu_alarm" {
  alarm_name          = "HighCPUAlarm"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/RDS"
  period              = 300
  statistic           = "Average"
  threshold           = 75
  alarm_description   = "Alarm when RDS CPU exceeds 75%"
  actions_enabled     = true

  dimensions = {
    DBInstanceIdentifier = aws_db_instance.postgres.id
  }

  alarm_actions = [aws_sns_topic.rds_autoscale_topic.arn]
}
```

2. Create an SNS topic to trigger the Lambda function when the alarm goes off.

```ruby
resource "aws_sns_topic" "rds_autoscale_topic" {
  name = "rds-autoscale-topic"
}

resource "aws_sns_topic_subscription" "lambda_subscription" {
  topic_arn = aws_sns_topic.rds_autoscale_topic.arn
  protocol  = "lambda"
  endpoint  = aws_lambda_function.rds_autoscale_lambda.arn
}
```

3. Create the Lambda Function to Add/Remove Read Replicas

```python
import json
import boto3

rds_client = boto3.client('rds')

# Define the threshold for the number of replicas
MAX_REPLICAS = 3  # Maximum read replicas you want to create
MIN_REPLICAS = 1  # Minimum number of read replicas to keep

# Modify this function to scale replicas
def lambda_handler(event, context):
    # Get the current replicas
    instance_identifier = event['Records'][0]['Sns']['Message']
    current_replicas = get_current_replicas(instance_identifier)
    
    if is_scale_up_event(event):
        if current_replicas < MAX_REPLICAS:
            create_read_replica(instance_identifier)
    elif is_scale_down_event(event):
        if current_replicas > MIN_REPLICAS:
            delete_read_replica(instance_identifier)
    
    return {
        'statusCode': 200,
        'body': json.dumps('Scaling operation completed')
    }

def get_current_replicas(instance_identifier):
    response = rds_client.describe_db_instances(DBInstanceIdentifier=instance_identifier)
    return len(response['DBInstances'][0]['ReadReplicaDBInstanceIdentifiers'])

def create_read_replica(instance_identifier):
    # Create a read replica of the DB instance
    rds_client.create_db_instance_read_replica(
        DBInstanceIdentifier=f"{instance_identifier}-replica",
        SourceDBInstanceIdentifier=instance_identifier,
        DBInstanceClass='db.t3.micro',
        AvailabilityZone='us-west-2a'  # Adjust to your zone
    )
    print(f"Created read replica for {instance_identifier}")

def delete_read_replica(instance_identifier):
    # Get the replica instance identifier
    replicas = rds_client.describe_db_instances(DBInstanceIdentifier=instance_identifier)['DBInstances'][0]['ReadReplicaDBInstanceIdentifiers']
    if replicas:
        # Delete one of the replicas
        replica_id = replicas[0]
        rds_client.delete_db_instance(
            DBInstanceIdentifier=replica_id,
            SkipFinalSnapshot=True
        )
        print(f"Deleted read replica {replica_id}")

def is_scale_up_event(event):
    # Implement logic to determine if this is a scale-up event
    return 'scale_up' in event['Records'][0]['Sns']['Message']

def is_scale_down_event(event):
    # Implement logic to determine if this is a scale-down event
    return 'scale_down' in event['Records'][0]['Sns']['Message']
```

4. Create the Lambda Function using Terraform

```ruby
resource "aws_lambda_function" "rds_autoscale_lambda" {
  function_name = "rds_autoscale_lambda"
  role          = aws_iam_role.lambda_exec.arn
  handler       = "lambda_function.lambda_handler"
  runtime       = "python3.9"
  timeout       = 60
  memory_size   = 128

  filename = "lambda_function.zip" # Package your Python code into a zip file

  environment {
    variables = {
      AWS_DEFAULT_REGION = var.aws_region
    }
  }
}

resource "aws_iam_role" "lambda_exec" {
  name = "lambda_exec_role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Action = "sts:AssumeRole",
        Effect = "Allow",
        Principal = {
          Service = "lambda.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_basic_execution" {
  role       = aws_iam_role.lambda_exec.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_iam_role_policy" "lambda_policy" {
  role = aws_iam_role.lambda_exec.id

  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Action = [
          "rds:CreateDBInstanceReadReplica",
          "rds:DeleteDBInstance",
          "rds:DescribeDBInstances"
        ],
        Effect   = "Allow",
        Resource = "*"
      }
    ]
  })
}
```

# Improvements

## CloudWatch Alarms

1. Cloudwatch alarm for **RDS Memory** utilization

```ruby
resource "aws_cloudwatch_metric_alarm" "rds-postgres-memory-utilization-alarm" {
  alarm_name          = "${var.project}-rds-postgres-memory-utilization-alarm-${var.environment}"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = "1"
  metric_name         = "FreeableMemory"
  namespace           = "AWS/RDS"
  period              = "300"
  statistic           = "Average"
  threshold           = "6400"
  alarm_description   = "This metric monitors RDS Postgres memory utilization"
  alarm_actions       = [aws_sns_topic.databases-sns-topic.arn]
  ok_actions          = [aws_sns_topic.databases-sns-topic.arn]
  dimensions = {
    DBInstanceIdentifier = aws_db_instance.rds-postgres.id
  }
}
```

2. Cloudwatch alarm for **RDS Storage** utilization

```ruby
resource "aws_cloudwatch_metric_alarm" "rds-postgres-storage-utilization-alarm" {
  alarm_name          = "${var.project}-rds-postgres-storage-utilization-alarm-${var.environment}"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = "1"
  metric_name         = "FreeStorageSpace"
  namespace           = "AWS/RDS"
  period              = "3600"
  statistic           = "Average"
  threshold           = "16000"
  alarm_description   = "This metric monitors RDS Postgres storage utilization"
  alarm_actions       = [aws_sns_topic.databases-sns-topic.arn]
  ok_actions          = [aws_sns_topic.databases-sns-topic.arn]
  dimensions = {
    DBInstanceIdentifier = aws_db_instance.rds-postgres.id
  }
}
```
