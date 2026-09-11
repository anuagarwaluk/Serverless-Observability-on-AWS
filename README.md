## Serverless Observability on AWS

A fully instrumented serverless CRUD stack, validated by hunting a seeded 5 second fault down to a single line of code.

Show Image Show Image Show Image Show Image Show Image Show Image Show Image


## Executive summary

When a production system slows down, the expensive part is rarely the fix. It is the hours a team spends working out what to fix. This project demonstrates the engineering discipline that collapses that search: full observability across metrics, logs and traces, wired into a serverless stack from day one.

To prove the instrumentation works, a 5 second fault was deliberately seeded into one code path. The telemetry found it blind: from "the API is slow" to the exact line of code in minutes, with no debugger and no guesswork.

The same instrumentation publishes business counters (records created, updated, deleted) onto the same dashboard as technical health. Engineering and leadership read one version of the truth. That is the difference between "we think it is the database" and "it is one line in the delete path, the fix ships today."

## Architecture

<img width="2400" height="3000" alt="aws-observability-architecture (2)" src="https://github.com/user-attachments/assets/067aab50-0bbd-4ca6-ba00-4c420ac3daf3" />

Request path

Users → Amazon S3 (static site) → Amazon API Gateway (REST) → AWS Lambda (Python) → Amazon DynamoDB

Telemetry plane

Every hop reports out:

Component	Reports to
API Gateway	X-Ray trace segments, CloudWatch metrics
Lambda	X-Ray subsegments (code level), structured logs, default and custom metrics
DynamoDB	X-Ray downstream segments, CloudWatch metrics
Everything	One CloudWatch dashboard, technical and business views side by side
The three pillars, as implemented

Monitoring tells you that something broke. Observability lets you ask why, without shipping new code to find out.

Pillar	Question it answers	AWS service	What this stack does
Metrics	WHAT is happening	CloudWatch Metrics	AWS defaults plus four custom business counters
Logs	WHY it happened	CloudWatch Logs	Structured JSON with request correlation IDs
Traces	WHERE it happened	AWS X-Ray	Service map, per request traces, code level subsegments

All three correlate on the request ID. That correlation is what turns three separate tools into one investigation.

Case study: finding a 5 second bug without opening the code

Every operation on the stack returned in milliseconds, except DELETE, which took over 5 seconds. Here is the hunt, exactly as the telemetry surfaced it.

Step 1. X-Ray service map: which service? API Gateway and DynamoDB showed millisecond averages. Lambda showed 5.51 seconds on the DELETE path. The problem lived in the function.
<img width="1843" height="615" alt="Screenshot 2026-07-30 at 11 57 55" src="https://github.com/user-attachments/assets/202d1363-b746-4793-9b1d-d040634c7a70" />

Step 2. Trace timeline: which code block? Opening a single DELETE trace broke the Lambda invocation into subsegments:


<img width="3240" height="1048" alt="Screenshot 2026-07-30 at 11 57 38" src="https://github.com/user-attachments/assets/75019a86-f814-4cdd-a817-5d439af7ba33" />
<img width="1781" height="756" alt="Screenshot 2026-07-30 at 11 46 53" src="https://github.com/user-attachments/assets/0a8851d0-b50b-431e-be7e-6f6351d06305" />


Segment	Duration
AWS::Lambda::Function	5.51 s
└ delete_validation_process	5.00 s
└ remaining subsegments	milliseconds

One subsegment owned the entire delay.

Step 3. The line itself. The subsegment name mapped straight to the handler code:

<img width="3214" height="1090" alt="Screenshot 2026-07-30 at 11 58 11" src="https://github.com/user-attachments/assets/9dc25bc0-47d2-4a11-97c1-5ed01eadaf8b" />


python
with xray_recorder.in_subsegment('delete_validation_process') as subsegment:
    subsegment.put_annotation('operation', 'delete_validation')
    subsegment.put_metadata('item_id', item_id)
    time.sleep(5)  # seeded fault: could the telemetry find this line blind?

It could. No print statements, no redeploys, no guessing. The fault was planted to validate the instrumentation, but its production cousins are everywhere: a retry loop, a cold cache, a synchronous call to a slow third party. Same symptom, same hunt.

A note on cold starts. The first invocation in each burst shows a visibly higher duration in the trace list. X-Ray makes cold starts observable rather than anecdotal, which is exactly the evidence you need before deciding whether provisioned concurrency is worth paying for.

Instrumentation patterns

Three patterns carry this whole setup. Each is a few lines of code.

1. Code level tracing with X-Ray subsegments
python
from aws_xray_sdk.core import xray_recorder

with xray_recorder.in_subsegment('delete_validation_process') as subsegment:
    subsegment.put_annotation('operation', 'delete_validation')
    subsegment.put_metadata('item_id', item_id)
    # business logic here

Annotations are indexed and searchable across traces. Metadata carries the business context. Together they make a trace answer questions, not just draw timelines.

2. Structured logging with request correlation
python
logger.info("Processing request", extra={
    'request_id': request_id,
    'method': event.get('httpMethod'),
    'path': event.get('path')
})

logger.info() beats print() for three reasons: you can alert on log levels, the output is structured JSON that CloudWatch Logs Insights can query, and the request ID lets you jump from any trace to the exact log lines for that request.

3. Custom business metrics
python
cloudwatch.put_metric_data(
    Namespace='Observability/DataApp',
    MetricData=[{
        'MetricName': 'DataCreated',
        'Value': 1,
        'Unit': 'Count'
    }]
)

Published on every successful operation: DataCreated, DataRetrieved, DataUpdated, DataDeleted. Swap the names for OrdersPlaced or PaymentsFailed and this is the pattern behind every business facing dashboard.

AWS default metrics vs custom metrics
	AWS default metrics	Custom metrics
Where they come from	Automatic for every Lambda, zero code	put_metric_data in application code
Examples	Invocations, Duration, Errors, Throttles, ConcurrentExecutions	DataCreated, DataRetrieved, DataUpdated, DataDeleted
Question answered	Is the infrastructure healthy?	Is the business healthy?
Cost	Free	Paid per metric, so track what matters

The dashboard in this project shows both, side by side. Infrastructure health without business context is half the picture.

The debugging runbook this stack enables
Slow API reported
X-Ray service map:which service?
Trace list:which request?
Subsegments:which code block?
Logs via request ID:why did it happen?
Metrics:one off or pattern?
Start at the X-Ray service map to isolate the slow service.
Drill into individual traces for that operation.
Read the subsegment timeline to find the guilty code block.
Pivot to CloudWatch Logs using the request ID for the why.
Check metrics to decide whether this is a spike or a trend.

This sequence works because every signal shares the same correlation ID. Build that in from the start and incidents become queries.

Field notes: what separates production grade from a demo
The Lambda "Active tracing" checkbox is not enough. It gives you totals and cold start visibility. Code level breakdowns require the X-Ray SDK inside the handler.
X-Ray is not Lambda only. The same SDK and daemon instrument EC2, ECS, EKS and on premises workloads. It is a tracing service, not a serverless feature.
Structure your logs from day one. Retrofitting correlation IDs across a fleet of services mid incident is nobody's idea of fun.
Be selective with custom metrics. They cost money per metric. Track the numbers the business would ask about in a review, not everything you can count.
Tech stack

Services: Amazon S3 (static website hosting), Amazon API Gateway (REST), AWS Lambda (Python), Amazon DynamoDB, Amazon CloudWatch (Metrics, Logs, Dashboards), AWS X-Ray

Libraries and patterns: aws-xray-sdk (subsegments, annotations, metadata), Python logging with structured JSON output, boto3 CloudWatch put_metric_data


<img width="732" height="56" alt="image" src="https://github.com/user-attachments/assets/6d6d2775-4fcf-45af-a872-aa3b19b7db72" />
