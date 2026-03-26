# AWS EC2 Auto-Stop — Lambda + EventBridge

Automatically stops a running EC2 instance on a scheduled interval using AWS Lambda and EventBridge. No manual intervention required — fully serverless and event-driven.

---

## Architecture

```
EventBridge (rate: 5 minutes)
    └──> AWS Lambda (Python + boto3)
              └──> Amazon EC2 — StopInstances API
                        |
                   IAM Role (least privilege)
```

---

## AWS Services Used

| Service | Role |
|---|---|
| AWS Lambda | Runs the stop logic on a schedule |
| Amazon EventBridge | Triggers Lambda every 5 minutes |
| Amazon EC2 | Target instance to be stopped |
| AWS IAM | Grants Lambda permission to call EC2 APIs |
| Amazon CloudWatch Logs | Captures Lambda execution logs |

---

## Prerequisites

- An AWS account
- A running EC2 instance (note the Instance ID and region)
- AWS Console access

---

## Setup

### 1. Launch EC2 Instance

Go to **EC2 > Instances > Launch Instance** and note:
- Instance ID (e.g. `i-0xxxxxxxxxxxxxxxxx`)
- Region (e.g. `us-east-1`)

### 2. Create IAM Role

Go to **IAM > Roles > Create Role**:
- Trusted entity: `AWS Service → Lambda`
- Attach the inline policy below (least privilege)
- Add `AWSLambdaBasicExecutionRole` for CloudWatch Logs
- Role name: `lambda-to-access-ec2`

**IAM Policy (`iam_policy.json`):**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "VisualEditor0",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:StartInstances",
        "ec2:StopInstances"
      ],
      "Resource": "*"
    }
  ]
}
```

### 3. Create Lambda Function

Go to **Lambda > Create Function**:
- Choose: **Use a blueprint** → search `cana` → select `lambda-canary` (python3.12)
- Function name: `fun-01` (or your choice)
- Execution role: select `lambda-to-access-ec2`
- EventBridge trigger: `rate(5 minutes)`
- Click **Create Function**

### 4. Deploy the Function Code

Replace the default blueprint code with [`ec2_stop_lambda.py`](ec2_stop_lambda.py):

```python
import boto3
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    instance_id = 'YOUR_INSTANCE_ID'  # Replace with your EC2 Instance ID
    region = 'YOUR_REGION'            # e.g. us-east-1

    ec2 = boto3.client('ec2', region_name=region)

    try:
        response = ec2.stop_instances(InstanceIds=[instance_id])
        state = response['StoppingInstances'][0]['CurrentState']['Name']
        logger.info(f'Instance {instance_id} state: {state}')
        return {
            'statusCode': 200,
            'body': f'Successfully triggered stop for {instance_id}. State: {state}'
        }
    except Exception as e:
        logger.error(f'Error stopping instance: {str(e)}')
        raise e
```

> **Important:** Replace `YOUR_INSTANCE_ID` and `YOUR_REGION` with your actual values. Click **Deploy** after editing.

### 5. Configure EventBridge Trigger

If not already set during creation: **Lambda > Configuration > Triggers > Add Trigger**
- Source: EventBridge (CloudWatch Events)
- New rule → name: `rule-01`
- Rule type: Schedule expression
- Expression: `rate(5 minutes)`

---

## Testing

Before relying on the schedule, manually test from the Lambda console:

1. Open your Lambda function → **Test** tab
2. Create new test event — name: `TestStopEC2`
3. Use this payload:

```json
{
  "version": "0",
  "id": "test-event-001",
  "detail-type": "Scheduled Event",
  "source": "aws.events",
  "account": "123456789012",
  "time": "2024-01-01T00:00:00Z",
  "region": "us-east-1",
  "detail": {}
}
```

4. Click **Test** → confirm `Status: Succeeded`
5. Verify the EC2 instance moves to `stopping` → `stopped` in the EC2 Console

> Make sure the EC2 instance is in a **Running** state before testing.

**Expected result:**
```
Status: Succeeded
Response: { "statusCode": 200, "body": "Successfully triggered stop for i-0x.... State: stopping" }
```

---

## Schedule Expression Reference

| Expression | Meaning |
|---|---|
| `rate(5 minutes)` | Every 5 minutes (this project) |
| `rate(1 hour)` | Every hour |
| `cron(0 18 * * ? *)` | Every day at 6 PM UTC |
| `cron(0 8 ? * MON-FRI *)` | Weekdays at 8 AM UTC |

---

## Monitoring

Logs are available at: **CloudWatch > Log Groups > `/aws/lambda/<function-name>`**

Expected log output:
```
[INFO] Instance i-0x... state: stopping
[INFO] Successfully triggered stop for i-0x.... State: stopping
```

---

## Key Concepts

| Concept | Application |
|---|---|
| Serverless Compute | Lambda runs without managing servers |
| IAM Least Privilege | Only 3 EC2 permissions — no over-permission |
| Event-Driven Architecture | EventBridge triggers Lambda, Lambda acts on EC2 |
| Observability | CloudWatch Logs captures every invocation |

---

## Cleanup

To avoid unintended charges or repeated stops:
1. **Disable the EventBridge rule**: EventBridge > Rules > select `rule-01` > Disable
2. **Delete the Lambda function** if no longer needed
3. **Terminate the EC2 instance** if it was created only for this project
