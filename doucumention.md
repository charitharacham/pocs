# AWS Deletion Monitor - Proof of Concept (PoC) Documentation

## Overview
This Proof of Concept (PoC) demonstrates a real-time, event-driven AWS monitoring system designed to detect the deletion of critical resources and automatically notify administrators via Slack and Email, while maintaining a robust audit trail.

## Architecture Diagram

The system leverages a serverless architecture to ensure high availability, scalability, and zero maintenance overhead.

```mermaid
graph TD
    User([User / IAM Role]) -. "Deletes Resource\n(e.g., TerminateInstances)" .-> AWS_API[AWS API]
    
    subgraph AWS Cloud Environment
        AWS_API --> CloudTrail[AWS CloudTrail]
        CloudTrail -- "Logs API Call" --> EventBridge[Amazon EventBridge]
        
        EventBridge -- "Matches Rule & Triggers" --> Lambda{AWS Lambda\nDeletion Monitor}
        
        Lambda -- "Fetches Webhook" --> SecretsManager[(AWS Secrets Manager)]
        Lambda -- "Writes Audit Log" --> DynamoDB[(Amazon DynamoDB)]
        Lambda -- "Publishes Alert" --> SNS[Amazon SNS]
    end
    
    subgraph External Systems
        SNS -- "Sends Email" --> Email[Admin Email]
        Lambda -- "HTTP POST" --> Slack[Slack Workspace]
    end

    classDef aws fill:#FF9900,stroke:#232F3E,stroke-width:2px,color:white;
    classDef aws_compute fill:#D86613,stroke:#232F3E,stroke-width:2px,color:white;
    classDef aws_db fill:#3355DA,stroke:#232F3E,stroke-width:2px,color:white;
    classDef aws_sec fill:#DD344C,stroke:#232F3E,stroke-width:2px,color:white;
    classDef aws_int fill:#CC2264,stroke:#232F3E,stroke-width:2px,color:white;
    classDef aws_mgmt fill:#3F8624,stroke:#232F3E,stroke-width:2px,color:white;
    classDef external fill:#4A154B,stroke:#fff,stroke-width:2px,color:white;
    
    class CloudTrail aws_mgmt;
    class EventBridge aws_int;
    class Lambda aws_compute;
    class SecretsManager aws_sec;
    class DynamoDB aws_db;
    class SNS aws_int;
    class Slack external;
```

## Sequence Diagram

The following sequence diagram outlines the exact order of operations once a destructive action occurs.

```mermaid
sequenceDiagram
    participant U as User / IAM Role
    participant CT as AWS CloudTrail
    participant EB as Amazon EventBridge
    participant SM as AWS Secrets Manager
    participant L as AWS Lambda
    participant DDB as Amazon DynamoDB
    participant SNS as Amazon SNS
    participant S as Slack
    participant E as Email
    
    U->>CT: Destructive API Call (e.g., DeleteBucket)
    CT->>EB: Log API Event
    EB->>EB: Match Custom Rule
    EB->>L: Trigger Lambda with Event Payload
    
    activate L
    L->>SM: GetSecretValue(Slack Webhook)
    SM-->>L: Return Secure Webhook URL
    
    par Logging
        L->>DDB: PutItem (Event Details: Who, What, When)
    and Notification (SNS)
        L->>SNS: Publish (Email Message)
        SNS-->>E: Deliver Email Alert
    and Notification (Slack)
        L->>S: POST (Formatted Block Message)
    end
    
    L-->>EB: Return Success
    deactivate L
```

## Components and Responsibilities

1. **AWS CloudTrail**: Captures all management events (API calls) made within the AWS account.
2. **Amazon EventBridge**: Acts as the event router. It filters CloudTrail events for specific destructive actions (e.g., `DeleteBucket`, `TerminateInstances`, `DeleteDBInstance`) and routes them to the Lambda function.
3. **AWS Lambda**: The core processing unit. It parses the incoming event payload, enriches it, and handles the orchestration of notifications and logging.
4. **AWS Secrets Manager**: Securely stores the Slack incoming webhook URL, preventing sensitive credentials from being hardcoded in code or environment variables.
5. **Amazon DynamoDB**: Serves as the audit log repository. Stores structured records of each deletion event for historical tracking and compliance purposes.
6. **Amazon SNS**: Handles email delivery. Lambda publishes to an SNS topic, which then fans out the message to all subscribed email addresses.
7. **Slack Webhook**: An external integration point that receives a rich, formatted HTTP POST request from Lambda to alert the team in real-time.

## Security & Compliance Considerations
- **Least Privilege IAM**: The Lambda execution role only has permissions to `secretsmanager:GetSecretValue` for a specific secret, `dynamodb:PutItem` for a specific table, and `sns:Publish` for a specific topic.
- **Credential Management**: Webhooks are not exposed in plaintext anywhere in the configuration or code.
- **Auditability**: Every matched event is durably stored in DynamoDB before alerts are sent out.

