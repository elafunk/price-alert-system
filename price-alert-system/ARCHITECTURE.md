## Introduction

Many online stores do not provide reliable price alerts. The goal of this project is to design a cloud-based system that periodically monitors product prices and notifies a user when a specified price threshold is reached.

## In Scope

- Track one product's URL (single store/single page format)
- Check the price on a schedule (eg every 5 hours)
- Store: target price, last observed price, last checked time
- Send one email notification when price is less than or equal to the target
- Prevent duplicate notifications once triggered

## Out of Scope

- Multiple users/authentication
- UI or mobile app
- Supporting many stores or complex scraping
- Real-time checking (every second/minute)
- Price history charts
- SMS/push notifications (email only for now)

## Functional Requirements

- The system must fetch the product page and extract a numeric price.
- The system must run automatically on a fixed schedule.
- The system must compare the extracted price to a stored target price.
- The system must notify the user when the price is at or below the target.
- The system must store the last observed price and last checked timestamp.
- The system must avoid sending duplicate alerts for the same product once it has triggered.

## Non-Functional Requirements

- Cost: should run on AWS at minimal cost (serverless preferred).
- Reliability: temporary failures (timeouts, 500s) should not crash the whole system.
- Maintainability: code should be modular so adding another product later is easy.
- Observability: system should log failures and key events (price fetched, notification sent).
- Respectful traffic: price checks should be infrequent (hours, not minutes) to avoid overloading sites.

## Architectural Overview

1. A scheduled event triggers the system periodically
2. A Lambda function fetches the product page and extracts the current price
3. The Lambda reads the product configuration (URL, target price, last state) from DynamoDB
4. The Lambda updates DynamoDB with the new price and timestamp
5. If the current price is less than or equal to the target and an alert has not been sent, the Lambda publishes a message to SNS
6. SNS delivers an email notification to the user
7. CloudWatch stores logs for debugging and visibility

## AWS Services

- EventBridge -> triggers the checks on a schedule without servers
- Lambda -> runs code only when needed (low ops + low cost)
- DynamoDB -> simple persistent state store (target price, last price, last checked)
- SNS -> easy email notification delivery
- CloudWatch -> logs and metrics for monitoring

## Edge cases & possible failures

- Product page unavailable or timeout -> Log the error, do not update last_price, try again at next schedule
- Price parsing breaks (HTML changed) -> Log parsing error and store an error field in DynamoDB for visibility
- Duplicate notifications -> Use a boolean flag (notif_sent) or store last_notified_price
- Rate limiting or blocks -> Keep schedule conservative

## Deployment Region

All AWS resources are deployed in us-east-1 to avoid cross-region complexity.

## Observability

Lambda executions are logged to CloudWatch, allowing inspection of events, errors, and execution details.

## Planned IAM Permissions

The Lambda execution role will eventually require:

- dynamodb:GetItem
- dynamodb:PutItem
- dynamodb:UpdateItem
- sns:Publish

## Data Stored Per Product

The system stores minimal state required to make decisions between executions:

- product identifier
- product URL
- target price
- last observed price
- timestamp of last check
- whether a notification has already been sent

## DynamoDB Item Schema

Each item represents one tracked product with the following attributes:

- product_id (String, primary key)
- url (String)
- target_price (Number)
- last_price (Number)
- last_checked (String, ISO timestamp)
- notification_sent (Boolean)
