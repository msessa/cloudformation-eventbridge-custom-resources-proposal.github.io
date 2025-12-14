# CloudFormation Custom Resources: EventBridge Integration Proposal

## Executive Summary

This proposal introduces EventBridge Bus ARN as a third ServiceToken type for CloudFormation Custom Resources, complementing the existing SNS and Lambda-backed implementations. This integration addresses current architectural limitations and enables new deployment patterns that improve scalability, maintainability, and cross-account resource sharing.

## Problem Statement

CloudFormation currently supports two Custom Resource ServiceToken types, each with significant limitations:

### SNS-Backed Custom Resources
- **No built-in filtering**: All subscribers receive all messages, requiring custom filtering logic
- **Limited target types**: Cannot directly invoke StepFunctions StateMachines or other EventBridge-supported targets
- **Message delivery constraints**: SNS delivery semantics may not align with custom resource requirements

### Lambda-Backed Custom Resources
- **Direct coupling**: Tight binding between custom resources and their providers
- **Asset management overhead**: Requires bundling, versioning, and deployment of Lambda function code
- **Runtime maintenance burden**: Lambda runtimes age requiring regular code updates
- **Cross-account complexity**: Limited Lambda resource policy support complicates organizational deployments

## Proposed Solution

Introduce **EventBridge Bus ARN** as a third ServiceToken type for CloudFormation Custom Resources.

### Syntax
```yaml
Resources:
  MyCustomResource:
    Type: AWS::CloudFormation::CustomResource
    Properties:
      ServiceToken: !GetAtt MyEventBridge.Arn
      # Custom properties...
```

### Core Benefits
- **Decoupled architecture**: Custom resources and providers operate independently
- **Enhanced target support**: Native integration with Lambda, StepFunctions, SQS, SNS, and all EventBridge targets
- **Built-in filtering**: EventBridge rules enable precise event routing based on resource properties
- **Cross-account sharing**: Full IAM resource policy support for organizational deployments
- **Operational excellence**: Event replay, DLQ support, and comprehensive logging capabilities
- **Asset-free deployment**: Enables custom resource patterns without code bundling requirements

## Architecture Patterns

### Use Case 1: Asset-less Custom Resource Provider

```mermaid
graph TB
    subgraph "AWS Account - Single Stack"
        subgraph "CloudFormation Stack"
            CR1[Custom Resource 1<br/>ServiceToken: Bus ARN]
            CR2[Custom Resource 2<br/>ServiceToken: Bus ARN]
            CR3[Custom Resource N<br/>ServiceToken: Bus ARN]
            
            EB[EventBridge Bus<br/>Dedicated Custom Resource Bus]
            
            RULE[EventBridge Rule<br/>Source: aws.cloudformation<br/>DetailType: Custom Resource Event]
            
            SF[StepFunctions StateMachine<br/>Custom Resource Provider<br/>- onCreate Logic<br/>- onUpdate Logic<br/>- onDelete Logic]
            
            CW[CloudWatch Logs<br/>Execution Logging]
        end
        
        subgraph "CloudFormation Service"
            CFN[CloudFormation Engine]
        end
    end
    
    %% Event Flow
    CFN -->|Sends Custom Resource Events| EB
    EB -->|Routes via Rule| RULE
    RULE -->|Triggers| SF
    SF -->|Logs Execution| CW
    SF -->|HTTP PUT to Pre-Signed ResponseURL| CFN
    
    %% Resource Relationships
    CR1 -.->|References| EB
    CR2 -.->|References| EB
    CR3 -.->|References| EB
    
    %% Styling
    classDef customResource fill:#ff9999,stroke:#333,stroke-width:2px
    classDef eventBridge fill:#99ccff,stroke:#333,stroke-width:2px
    classDef stepFunction fill:#99ff99,stroke:#333,stroke-width:2px
    classDef cloudFormation fill:#ffcc99,stroke:#333,stroke-width:2px
    
    class CR1,CR2,CR3 customResource
    class EB,RULE eventBridge
    class SF stepFunction
    class CFN cloudFormation
```

**Components within single stack:**
- StepFunctions StateMachine implementing custom resource lifecycle
- Dedicated EventBridge bus for custom resource events
- EventBridge rule routing events to StateMachine
- Custom resources using the bus as ServiceToken

**Advantages:**
- Zero asset management overhead
- Natural StackSets compatibility
- Simplified CI/CD pipelines
- Reduced deployment complexity

### Use Case 2: Centralized Custom Resource Providers

```mermaid
graph TB
    subgraph "CloudFormation Service"
        CFN[CloudFormation Engine]
    end

    subgraph "Platform Account (Custom Resource Providers)"
        subgraph "Shared EventBridge Infrastructure"
            SHARED_BUS[Shared EventBridge Bus<br/>Cross-Account Custom Resource Bus<br/>Organization-wide Access]
            
            RULE1[EventBridge Rule 1<br/>ResourceType: Custom::DatabaseSetup]
            RULE2[EventBridge Rule 2<br/>ResourceType: Custom::NetworkConfig]
            RULE3[EventBridge Rule 3<br/>ResourceType: Custom::SecurityGroup]
        end
        
        subgraph "Custom Resource Providers"
            LAMBDA1[Lambda Function<br/>Database Setup Provider]
            SF1[StepFunctions StateMachine<br/>Network Configuration Provider]
            LAMBDA2[Lambda Function<br/>Security Group Provider]
        end
        
        LOGS[CloudWatch Logs<br/>Centralized Logging]
    end
    
    subgraph "Consumer Account A"
        subgraph "Application Stack A"
            CRA1[Custom Resource<br/>Type: Custom::DatabaseSetup<br/>ServiceToken: Shared Bus ARN]
            CRA2[Custom Resource<br/>Type: Custom::NetworkConfig<br/>ServiceToken: Shared Bus ARN]
        end
        
        CFN[CloudFormation Engine]
    end
    
    subgraph "Consumer Account B"
        subgraph "Application Stack B"
            CRB1[Custom Resource<br/>Type: Custom::SecurityGroup<br/>ServiceToken: Shared Bus ARN]
            CRB2[Custom Resource<br/>Type: Custom::DatabaseSetup<br/>ServiceToken: Shared Bus ARN]
        end
        
        CFN[CloudFormation Engine]
    end
    
    subgraph "Consumer Account N"
        subgraph "Application Stack N"
            CRN1[Custom Resource<br/>Type: Custom::NetworkConfig<br/>ServiceToken: Shared Bus ARN]
        end
        
        CFN[CloudFormation Engine]
    end
    
    %% Cross-Account Event Flow
    CFN -->|Custom Resource Events| SHARED_BUS
    CFN -->|Custom Resource Events| SHARED_BUS
    CFN -->|Custom Resource Events| SHARED_BUS
    
    %% Rule-based Routing
    SHARED_BUS -->|Routes DatabaseSetup| RULE1
    SHARED_BUS -->|Routes NetworkConfig| RULE2
    SHARED_BUS -->|Routes SecurityGroup| RULE3
    
    %% Provider Invocation
    RULE1 -->|Triggers| LAMBDA1
    RULE2 -->|Triggers| SF1
    RULE3 -->|Triggers| LAMBDA2
    
    %% Response Flow
    LAMBDA1 -->|HTTP PUT to Pre-Signed ResponseURL| CFN
    LAMBDA1 -->|HTTP PUT to Pre-Signed ResponseURL| CFN
    SF1 -->|HTTP PUT to Pre-Signed ResponseURL| CFN
    SF1 -->|HTTP PUT to Pre-Signed ResponseURL| CFN
    LAMBDA2 -->|HTTP PUT to Pre-Signed ResponseURL| CFN
    
    %% Logging
    LAMBDA1 -->|Execution Logs| LOGS
    SF1 -->|Execution Logs| LOGS
    LAMBDA2 -->|Execution Logs| LOGS

    
    class CRA1,CRA2,CRB1,CRB2,CRN1 customResource
    class SHARED_BUS,RULE1,RULE2,RULE3 eventBridge
    class LAMBDA1,SF1,LAMBDA2 provider
    class CFNA,CFNB,CFNN cloudFormation
```

**Platform account components:**
- Shared EventBridge bus with organization-wide access
- Multiple custom resource providers (Lambda, StepFunctions)
- EventBridge rules routing by ResourceType to appropriate providers

**Consumer account components:**
- Custom resources targeting the shared bus ServiceToken
- No provider implementation required

**Advantages:**
- Centralized provider governance
- Consistent resource behavior across accounts
- Reduced duplication of provider logic
- Platform team ownership model

## Strategic Alignment

This proposal aligns with AWS strategic initiatives:
- **Serverless-first architecture**: Reduces operational overhead through managed services
- **Event-driven patterns**: Promotes modern, decoupled application design
- **Multi-account governance**: Enhances enterprise deployment capabilities
- **Developer productivity**: Simplifies custom resource development and maintenance

## Backwards Compatibility

EventBridge Bus ARN support represents an additive enhancement to CloudFormation. Existing SNS and Lambda-backed custom resources continue operating unchanged, ensuring zero impact on current deployments.

*Prepared for internal AWS product team review*