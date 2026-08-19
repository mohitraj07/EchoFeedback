# EchoFeedback

A web-based feedback and rating system with live results and instant email alerts, built on AWS (DynamoDB, Lambda, SNS, API Gateway, Amplify).

## How it works

1. User submits their name, a message, and a star rating on the webpage
2. The page sends the data to a Lambda function through a REST API (API Gateway)
3. Lambda saves the feedback to DynamoDB and publishes an alert to SNS at the same time
4. An email notification arrives immediately
5. The page re-fetches and displays all feedback live, with star ratings shown

## Services used

- **DynamoDB** — stores each feedback entry
- **Lambda** — backend logic connecting everything, using an `operation`-based routing pattern
- **SNS** — sends an email alert the moment new feedback is submitted
- **API Gateway (REST API)** — exposes the Lambda function as a POST endpoint
- **AWS Amplify** — hosts the frontend webpage
- **IAM** — scoped permissions (DynamoDB PutItem/Scan + SNS Publish, restricted to this project's specific resources only)

## Screenshots

### DynamoDB table
![DynamoDB table](Screenshot/01-DynamoDB-Table.png)

### SNS topic with confirmed subscription
![SNS subscription](Screenshot/02-SNS-Subscription.png)

### Lambda code deployed
![Lambda code deployed](Screenshot/03-LambdaCode-Deployed.png)

### Lambda test executed successfully
![Lambda test successful](Screenshot/04-LambdaTest-Successfull.png)

### Email alert received
![Email alert](Screenshot/05-Email-Alert.png)

### Scoped IAM policy
![IAM policy](Screenshot/06-IAM-Policy.png)

### REST API deployed (API Gateway)
![API Gateway invoke URL](Screenshot/07-API-Gateway-Invoke-URL.png)

### Frontend code
![index.html in VS Code](Screenshot/08-Index-HTML-VScode.png)

### AWS Amplify dashboard
![Amplify dashboard](Screenshot/09-AWS-Amplify-Dashboard.png)

### Live demo on Amplify
![Live demo](Screenshot/10-AWS-Amplify-Live-Demo.png)

## Lambda function code

```python
import json
import boto3
import time
import uuid
from decimal import Decimal

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('feedback')

sns = boto3.client('sns')
SNS_TOPIC_ARN = "arn:aws:sns:ap-south-1:828078130402:feedback-alerts"


class DecimalEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, Decimal):
            return int(obj) if obj % 1 == 0 else float(obj)
        return super(DecimalEncoder, self).default(obj)

sns = boto3.client('sns')
SNS_TOPIC_ARN = "arn:aws:sns:ap-south-1:xxxxxxxxxxxx:feedback-alerts"  # replace with your actual ARN


def lambda_handler(event, context):
    # Handles BOTH direct test events (operation at top level )
    # AND requests coming through API Gateway (where
    # the real data is a JSON string inside event['body']).
    if 'operation' in event:
        data = event
    else:
        data = json.loads(event.get('body', '{}'))

    operation = data.get('operation')

    if operation == 'submitFeedback':
        return submitFeedback(data)
    else:
        return getFeedback()


def submitFeedback(data):
    name = data.get('name', 'Anonymous')
    message = data.get('message', '')
    rating = data.get('rating', 0)

    if not message.strip():
        return build_response(400, {'error': 'Message cannot be empty'})

    feedback_id = str(uuid.uuid4())
    gmt_time = time.gmtime()
    now = time.strftime('%a, %d %b %Y %H:%M:%S', gmt_time)

    table.put_item(
        Item={
            'feedbackId': feedback_id,
            'name': name,
            'message': message,
            'rating': rating,
            'createdAt': now
        }
    )

    # Send an email alert via SNS
    sns.publish(
        TopicArn=SNS_TOPIC_ARN,
        Subject="New Feedback Received",
        Message=f"New feedback from {name}\nRating: {rating}/5\nMessage: {message}\nAt: {now}"
    )

    return build_response(200, {'message': f'Feedback from {name} submitted successfully'})


def getFeedback():
    response = table.scan()
    items = response['Items']
    items.sort(key=lambda x: x.get('createdAt', ''), reverse=True)

    return build_response(200, {'feedback': items})


def build_response(status_code, body_dict):
    return {
        'statusCode': status_code,
        'headers': {
            'Content-Type': 'application/json',
            'Access-Control-Allow-Origin': '*'
        },
        'body': json.dumps(body_dict, cls=DecimalEncoder)
    }
```

## Frontend

See [`index.html`](index.html) in this repo for the full page code.