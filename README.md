```
sequenceDiagram
    participant User
    participant Orchestrator API
    participant Orchestrator Trigger Lambda
    participant CR Creation Lambda
    participant Status Table
    participant IAM Manager APIs
    participant ADCL API
    participant ADCLs Step Function
    participant Request Table
    participant CodeBuild
    participant Trigger Lambda
    participant CR Orchestration Lambda

    User ->> Orchestrator API: Initiate Request
    Orchestrator API ->> Orchestrator Trigger Lambda: Pre-process Request
    Orchestrator Trigger Lambda ->> CR Creation Lambda: Create CR
    CR Creation Lambda ->> CR Orchestration Lambda: Check CR Status
    CR Orchestration Lambda ->> Status Table: Check Status
    Status Table ->> CR Orchestration Lambda: Status (Approved/Not Approved)
    CR Orchestration Lambda ->> Orchestrator API: Get Status by Request ID
    Orchestrator API ->> User: Status Response
    Orchestrator Trigger Lambda ->> Orchestrator Proxy Lambda: Proxy Request
    Orchestrator Proxy Lambda ->> CodeBuild: Manage Trigger (asynchronous)
    CodeBuild ->> Status Table: Get Execution Results
    Status Table ->> Orchestrator API: Get Execution Records by Request ID
    Orchestrator API ->> User: Execution Records Response
    Orchestrator Trigger Lambda ->> ADCL API: Pre-process Request
    ADCL API ->> Status Table: Update Execution Status
    Status Table ->> Orchestrator API: Execution Status Updated
    Orchestrator API ->> User: Execution Status Response
    ADCLs Step Function ->> Direct Connect: Connect to IBM APIs
    Direct Connect ->> IBM APIs: Execute Query/Update/Create
    ADCLs Step Function ->> Request Table: Update Execution Status
    Request Table ->> Orchestrator API: Get Execution Records
    Orchestrator API ->> User: Execution Records Response
    Orchestrator Proxy Lambda ->> Status Table: Write Execution Results



flowchart TD
    A(User-facing Endpoint Part)
    A1[Orchestrator API endpoint - Orchestrator Trigger lambda for request pre-processing]
    A2[Orchestrator Trigger lambda will check the CR is created in production or non-production env, if non-production env = asynchronous Orchestrator Step Function]
    A3[If the CR Role is created in production env -> CR Creation lambda for create the CR]
    A4[Orchestrator API endpoint - Get status from DynamoDB table by request ID]
    A5[After IAM Role created successfully ADCLs API endpoint - Trigger lambda for request pre-processing (eg. validate the input param) to ADCLs Step Function - update execution status into the Request table]
    A6[Orchestrator step function - write execution results and request history into DynamoDB table]
    
    B(CodeBuild Part)
    B1[CodeBuild Step 2.1 - Manager API endpoint - Manage Trigger lambda & asynchronous CodeBuild job]
    B2[CodeBuild Step 2.1 - Manager API endpoint - Manage Status lambda - report every 60 seconds until build status is not IN_PROGRESS]
    B3[Orchestrator step function - write execution results and request history into DynamoDB table]
    
    C(ADCLs StepFunction Part)
    C1[ADCLs StepFunction - Direct Connect -> IBM APIs - the automation will connect to the IBM APIs for actions such as query, update, and creation]
    C2[ADCLs StepFunction - update execution status into the Request table]

    subgraph User-facing Endpoint Part
        A1 --> A2
        A2 --> A3
        A3 --> A4
        A4 --> A5
        A5 --> A6
    end
    
    subgraph CodeBuild Part
        B1 --> B2
        B2 --> B3
    end
    
    subgraph ADCLs StepFunction Part
        C1 --> C2
    end

    A --> B
    A --> C


```
