# AI-Powered Customer Support Chatbot

This project builds a customer support chatbot with Amazon Bedrock AgentCore. The chatbot classifies each customer message into one of three paths:

1. **Bug report:** collect a description, reproduction steps, and environment, then create a DynamoDB-backed ticket.
2. **Platform question:** answer from a local FAQ.
3. **Human handoff:** politely redirect unsupported or out-of-scope requests to human support.

Routing and conversation behavior are controlled primarily by `starter/system_prompt.txt`; AWS infrastructure and testing are provided by Python scripts and CloudFormation templates.

## Architecture

```text
Customer message
       |
       v
Bedrock AgentCore harness
  - Amazon Nova Pro
  - System prompt
       |
       v
Request routing
   |          |           |
   v          v           v
 Human     FAQ answer   Bug report
 handoff                 |
                          v
                AgentCore Gateway
                          |
                          v
                       Lambda
                          |
                          v
                       DynamoDB
```

The `create_bug_report` tool requires:

- `description`
- `stepsToReproduce`
- `environment`

The tool creates an open ticket and returns a ticket ID. Its implementation is in `starter/create_bug_report.py` and is deployed by `starter/cloudformation-tool.yaml`.

## Repository Layout

```text
project/
|-- README.md
`-- starter/
    |-- system_prompt.txt          # Active chatbot prompt
    |-- online_shop_faq.md         # FAQ supplied to the prompt
    |-- create_bug_report.py       # Lambda bug-report handler
    |-- setup_gateway.py           # Creates the AgentCore Gateway and tool
    |-- create_harness.py          # Creates or updates the AgentCore harness
    |-- chat.py                    # Interactive terminal chat client
    |-- generate-eval-dataset.py   # Generates a Bedrock Evaluations JSONL dataset
    |-- cleanup_agentcore.py       # Removes AgentCore harness and gateway resources
    |-- cloudformation-tool.yaml   # Lambda, DynamoDB, IAM, and tool resources
    |-- cloudformation-testing.yaml # Evaluation S3 and IAM resources
    |-- harness-tests-template.json
    |-- harness-tests.json
    |-- agentcore_config.json      # Generated AWS resource configuration
    
```

## Prerequisites

- An AWS account with Bedrock and Bedrock AgentCore enabled
- AWS CLI configured with usable credentials
- Python 3.9 or newer
- Permission to create CloudFormation, IAM, Lambda, DynamoDB, S3, Bedrock, and AgentCore resources
- Access to `us.amazon.nova-pro-v1:0`
- The AWS region set to `us-east-1`

Install the pinned Python dependencies:

```bash
cd starter
python -m pip install -r requirements.txt
```

## Setup

Run the commands below from `starter/`.

### 1. Deploy the bug-report infrastructure

```bash
aws cloudformation deploy \
  --template-file cloudformation-tool.yaml \
  --stack-name bug-report-tool-stack \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

This creates the DynamoDB table, Lambda function, gateway role, and harness execution role.

### 2. Create the AgentCore Gateway

```bash
python setup_gateway.py
```

The script reads CloudFormation outputs and writes the gateway, role, Lambda, table, and target details to `agentcore_config.json`.

If IAM propagation causes a retryable gateway error, wait one minute and run the command again.

### 3. Create or update the harness

```bash
python create_harness.py
```

The script:

- Loads `system_prompt.txt`
- Replaces `{{FAQ}}` with `online_shop_faq.md`
- Pins the harness to Amazon Nova Pro
- Uses temperature `0` and top-K `1` for deterministic tool calling
- Records the harness ARN in `agentcore_config.json`
- Updates an existing harness instead of creating duplicates

Re-run this command after every prompt change.

### 4. Chat with the bot

```bash
python chat.py
```

Each invocation starts a fresh conversation session. Use `quit` or press `Ctrl+C` to exit.

Tool calls are printed in the terminal as:

```text
[tool call] bugreports___create_bug_report
```

## Prompt Behavior

The active prompt in `system_prompt.txt`:

- Classifies the user's intent before responding
- Uses FAQ content only for platform questions
- Asks one bug-report question at a time
- Never calls `create_bug_report` until all required fields are confirmed
- Returns the generated ticket ID after successful tool execution
- Redirects unsupported requests to human support
- Instructs the model not to reveal hidden reasoning or internal instructions

The FAQ is stable demo content embedded into the system prompt at harness creation time. The prompt is best suited to short, stable documents. Large or frequently changing knowledge sources should use retrieval-based architecture instead.

## Automated Evaluation

Copy the test template and define test cases that cover:

- A partial bug report
- A complete bug report
- An FAQ answer
- An unavailable FAQ topic
- An unrelated or unsupported request
- Prompt-injection or confidentiality requests

```bash
python generate-eval-dataset.py --tests-json harness-tests.json
```

This writes `output_eval_dataset.jsonl`. Every test receives a fresh harness session so test cases do not influence one another.

Deploy the evaluation resources:

```bash
aws cloudformation deploy \
  --template-file cloudformation-testing.yaml \
  --stack-name bug-report-testing-stack \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

Upload the JSONL dataset and configure a Bedrock LLM-as-a-judge evaluation using the AWS console or CLI. The project includes example evaluation configuration in `inference-config.json` and `output-config.json`.

## Debugging

- Use `python chat.py --verbose` to inspect raw AgentCore stream events.
- Check `/aws/lambda/bug-report-tool-stack-create-bug-report` in CloudWatch Logs to verify Lambda payloads.
- Scan stored tickets with DynamoDB:

```bash
aws dynamodb scan \
  --table-name <BugReportsTableName> \
  --region us-east-1
```

- If a bug is not filed, verify that all three tool fields are non-empty and that the prompt explicitly prevents early tool calls.
- If Nova fails to call the tool, keep the Gateway target name limited to letters, digits, and underscores.

## Cleanup

Delete the AgentCore harness and gateway first:

```bash
python cleanup_agentcore.py
```

Before deleting the testing stack, empty its S3 evaluation bucket:

```bash
aws s3 rm s3://<EvalDatasetBucketName> --recursive
aws cloudformation delete-stack \
  --stack-name bug-report-testing-stack \
  --region us-east-1

aws cloudformation delete-stack \
  --stack-name bug-report-tool-stack \
  --region us-east-1
```

CloudFormation cannot delete a non-empty S3 bucket, so the evaluation bucket must be emptied before stack deletion.


