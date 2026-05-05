---
title: "Handle unpredictable processing times with operational consistency when integrating asynchronous AWS services with an AWS Step Functions state machine"
url: "https://aws.amazon.com/blogs/compute/handle-unpredictable-processing-times-with-operational-consistency-when-integrating-asynchronous-aws-services-with-an-aws-step-functions-state-machine/"
date: "Fri, 14 Nov 2025 20:59:55 +0000"
author: "Philip Whiteside"
feed_url: "https://aws.amazon.com/blogs/compute/category/application-services/aws-step-functions/feed/"
---
<p>Integrating asynchronous AWS services with an <a href="https://aws.amazon.com/step-functions/" rel="noopener noreferrer" target="_blank">AWS Step Functions</a> state machine, presents a challenge when building serverless applications on <a href="https://aws.amazon.com/" rel="noopener noreferrer" target="_blank">Amazon Web Services</a> (AWS). Services such as <a href="https://aws.amazon.com/translate/" rel="noopener noreferrer" target="_blank">Amazon Translate</a>, <a href="https://aws.amazon.com/macie/" rel="noopener noreferrer" target="_blank">Amazon Macie</a>, and <a href="https://aws.amazon.com/bedrock/bda/" rel="noopener noreferrer" target="_blank">Amazon Bedrock Data Automation</a> (BDA) excel at handling long-running operations that can take more than 10 minutes to complete because of their asynchronous nature. Asynchronous services return an immediate <a href="https://www.rfc-editor.org/rfc/rfc9110.html#status.2xx" rel="noopener noreferrer" target="_blank">200 OK response, indicating that the request has succeeded</a>, upon job submission (see the API response syntax of <a href="https://docs.aws.amazon.com/translate/latest/APIReference/API_StartTextTranslationJob.html#API_StartTextTranslationJob_ResponseSyntax" rel="noopener noreferrer" target="_blank">StartTextTranslationJob</a> in Amazon Translate, <a href="https://docs.aws.amazon.com/macie/latest/APIReference/jobs.html#jobs-response-body-createclassificationjobresponse-example" rel="noopener noreferrer" target="_blank">CreateClassificationJob</a> in Macie, and <a href="https://docs.aws.amazon.com/bedrock/latest/APIReference/API_data-automation-runtime_InvokeDataAutomationAsync.html" rel="noopener noreferrer" target="_blank">InvokeDataAutomationAsync</a> in BDA), rather than waiting for the actual task completion and results.</p> 
<p>In this post, we explore using AWS Step Function state machine with <a href="https://aws.amazon.com/blogs/architecture/introduction-to-messaging-for-modern-cloud-architecture/" rel="noopener noreferrer" target="_blank">asynchronous AWS services</a>, look at some scenarios where the processing time can be unpredictable, explain when traditional solutions such as <a href="https://aws.amazon.com/sqs/faqs/#topic-0" rel="noopener noreferrer" target="_blank">polling</a> (periodically check) fall short, and demonstrate how to implement a generalized <a href="https://docs.aws.amazon.com/step-functions/latest/dg/connect-to-resource.html#connect-wait-token" rel="noopener noreferrer" target="_blank">callback pattern</a> to handle asynchronous operations into a more manageable synchronous flow. We cover the related architecture, technical implementation, and best practices, and we provide a real-world examples that uses the <a href="https://aws.amazon.com/cdk/" rel="noopener noreferrer" target="_blank">AWS Cloud Development Kit</a> (AWS CDK). Services used in this generalized callback pattern include <a href="https://aws.amazon.com/dynamodb/" rel="noopener noreferrer" target="_blank">Amazon DynamoDB</a>, <a href="https://aws.amazon.com/eventbridge/" rel="noopener noreferrer" target="_blank">Amazon EventBridge</a> and AWS Step Functions.</p> 
<h2>Understanding the issue this solution addresses</h2> 
<p>Asynchronous operations are designed to handle long-running operations without blocking resources, a design followed by many AWS services. However, these services create challenges in Step Functions workflows by returning immediate 200 OK responses rather than confirming task completion. This breaks the Step Functions execution model, which expects each step to be complete before advancing. Developers often attempt to address this issue through polling loops to repeatedly check the status of operations, an approach that works for containerized applications and <a href="https://aws.amazon.com/ec2" rel="noopener noreferrer" target="_blank">Amazon Elastic Compute Cloud (Amazon EC2)</a>. For these services, compute resources are already provisioned, but compute resources become problematic in serverless architectures when <a href="https://aws.amazon.com/lambda/" rel="noopener noreferrer" target="_blank">AWS Lambda</a> functions have a 15-minute execution limit, making them unsuitable for long-running polls. </p> 
<p>Step Functions supports <a href="https://docs.aws.amazon.com/step-functions/latest/dg/integrate-optimized.html" rel="noopener noreferrer" target="_blank">Run a Job (.sync)</a> to call a service and have Step Functions wait for a job to complete, but this works only for selected optimized integrations. However, this functionality is limited to specific AWS services such as <a href="https://aws.amazon.com/glue/" rel="noopener noreferrer" target="_blank">AWS Glue</a>. Amazon Translate, Macie, and other services are not optimized integrations. If your operation is not <a href="https://docs.aws.amazon.com/step-functions/latest/dg/integrate-optimized.html" rel="noopener noreferrer" target="_blank">listed</a> as working with .sync, it can benefit from the generalized callback pattern covered in this post.</p> 
<p>For these non-optimized integrations, an option is to use polling (periodically check). However, polling can lead to additional latency in response because polling times are unlikely to align with job completion. This is shown in the following figure.</p> 
<div class="wp-caption aligncenter" id="attachment_24886" style="width: 1034px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/10/27/sfnresume-image-1.png"><img alt="Timeline diagram showing alternating 'Job' blocks and 'Delay' blocks, with 'Poll' markers indicated at regular intervals along the time axis. The diagram illustrates a sequential process of job execution and delay periods." class="wp-image-24886 size-large" height="486" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/10/27/sfnresume-image-1-1024x486.png" style="margin: 10px 0px 10px 0px;" width="1024" /></a>
 <p class="wp-caption-text" id="caption-attachment-24886">Figure 1: A job processing and delay timeline diagram</p>
</div> 
<p>The Step Functions generalized callback pattern can solve this latency issue by pausing execution for up to one year while waiting for task completion (this does not incur additional cost). When such an asynchronous operation finishes, a callback mechanism resumes the workflow where it left off. This generalized callback pattern transforms asynchronous operations into synchronous ones, and it maintains cost efficiency and operational agility.</p> 
<h2>Scenarios</h2> 
<p>To help us see where this generalized callback pattern could be applied, let’s look at a few scenarios. Each of these scenarios makes use of AWS Step Functions state machines to run the applications’ workflows.</p> 
<h3>Scenario 1: Document translation with personally identifiable information compliance</h3> 
<p>Organizations must manage personally identifiable information (PII) when translating documents because PII can be duplicated across language outputs. For example, when translating a document containing “Jane Doe,” that name appears in both the original and translated versions, creating multiple instances of sensitive data that need compliance measures. Amazon Translate batch translation has a <a href="https://docs.aws.amazon.com/general/latest/gr/translate-service.html#limits_amazon_translate" rel="noopener noreferrer" target="_blank">default </a>concurrency of 10, meaning that translations could take more than 10 minutes or be queued for longer periods. Additionally, the Amazon Translate batch translation operation is asynchronous, holding the translation request in a queue until completed. The generalized callback pattern in this post makes sure that Step Functions state machine workflows resume appropriately to apply consistent PII handling across all outputs. In this scenario the design makes use of tagging <a href="https://aws.amazon.com/s3/" rel="noopener" target="_blank">Amazon Simple Storage Service (Amazon S3)</a> files as containing PII or not, which in turn associates S3 lifecycle policies for specific retention periods to those S3 objects.</p> 
<div class="wp-caption aligncenter" id="attachment_24885" style="width: 355px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/10/27/sfnresume-image-2.png"><img alt="Workflow diagram showing five connected steps: 1) Start, 2) StartTextTranslationJob, 3) Wait for Translate result, 4) Tag all files, 5) ending with End state." class="wp-image-24885 size-full" height="414" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/10/27/sfnresume-image-2.png" style="margin: 10px 0px 10px 0px;" width="345" /></a>
 <p class="wp-caption-text" id="caption-attachment-24885">Figure 2: A text translation workflow diagram</p>
</div> 
<h3>Scenario 2: Using concurrent execution to pause the state machine until processes have completed</h3> 
<p>Continuing from scenario 1, Macie and Amazon Translate can run in parallel (each approximately 10 minutes) rather than sequentially (approximately 20 minutes) for a better user experience. Similarly to Amazon Translate batch translation operations being asynchronous, the Macie create classification operation is also asynchronous. Step Functions state machines enable concurrent execution of both service requests. The generalized callback pattern enables the state machine to pause each parallel workflow and resume only when the asynchronous services have completed their jobs. Without this pattern, both services would immediately return 200 OK responses, causing the workflow to continue prematurely before translations or classification results are available. If the classification results are not available later in the workflow, then the appropriate PII tags will not be applied and therefore the appropriate lifecycle retention policy will also not be applied, resulting in not adhering to PII handling practices.</p> 
<div class="wp-caption aligncenter" id="attachment_24887" style="width: 667px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/10/27/sfnresume-image33.png"><img alt="" class="wp-image-24887 size-full" height="574" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/10/27/sfnresume-image33.png" style="margin: 10px 0px 10px 0px;" width="657" /></a>
 <p class="wp-caption-text" id="caption-attachment-24887">Figure 3: A parallel classification and translation workflow diagram</p>
</div> 
<h3>Scenario 3: Intelligent document processing</h3> 
<p>Organizations that use Bedrock Data Automation for intelligent document processing must take into consideration Regional concurrency limits. BDA has Regional <a href="https://docs.aws.amazon.com/general/latest/gr/bedrock.html#limits_bedrock" rel="noopener noreferrer" target="_blank">concurrency limits</a> “Max number of concurrent jobs” of 25 jobs in the us-east-1 and us-west-2 Regions. Also, BDA has a concurrency limit of only five jobs in other supported Regions, so large document batches could be queued for extended periods resulting in long processing wait times for the user. This service functionality is handled asynchronously as the duration of the request could be many minutes. The generalized callback pattern makes sure that workflows resume appropriately as soon as a job finishes rather than waiting an arbitrary time to check if the job has been completed. For example, the generalized callback pattern for BDA can be used to enhance the solution outlined in the blog post, <a href="https://aws.amazon.com/blogs/machine-learning/scalable-intelligent-document-processing-using-amazon-bedrock-data-automation/" rel="noopener noreferrer" target="_blank">Scalable intelligent document processing using Amazon Bedrock Data Automation</a>.</p> 
<div class="wp-caption aligncenter" id="attachment_24884" style="width: 620px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/10/27/sfnresume-image-3.png"><img alt="" class="wp-image-24884 size-full" height="409" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/10/27/sfnresume-image-3.png" style="margin: 10px 0px 10px 0px;" width="610" /></a>
 <p class="wp-caption-text" id="caption-attachment-24884">Figure 4: A data automation workflow diagram</p>
</div> 
<h2>Solution architecture</h2> 
<p>The following architecture diagram shows the generalized callback pattern (the blue section on the right side) integrated with your existing application (the grey section on the left side).</p> 
<div class="wp-caption aligncenter" id="attachment_24883" style="width: 971px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/10/27/sfnresume-image-4.png"><img alt="" class="wp-image-24883 size-full" height="621" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/10/27/sfnresume-image-4.png" style="margin: 10px 0px 10px 0px;" width="961" /></a>
 <p class="wp-caption-text" id="caption-attachment-24883">Figure 5: The Step Functions generalized callback architecture</p>
</div> 
<h3>Key components of this post’s solution architecture</h3> 
<p>This generalized callback pattern architecture consists of four essential components working together. Each component plays a specific role while maintaining cost efficiency and operational reliability. The following components form the foundation of this pattern:</p> 
<ul> 
 <li><strong>Step Functions task</strong>: Implements the “Wait for Callback” task state generating unique task tokens for workflow resumption.</li> 
 <li><strong>EventBridge rule</strong>: Monitors asynchronous service completion events and is customizable for different service patterns. AWS services make use of an <a href="https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-event-bus.html" rel="noopener noreferrer" target="_blank">event bus</a> to route service event notifications to other services, such as job completions.</li> 
 <li><strong>DynamoDB</strong>: Provides persistent storage correlating job IDs with task tokens for quick lookup.</li> 
 <li><strong>Step Functions state machine</strong>: Manages the resume process and makes sure of proper cleanup of stored tokens.</li> 
</ul> 
<h3>Solution process</h3> 
<p>This generalized callback pattern operates through a coordinated sequence of four key steps. Each step builds upon the previous one. The following process demonstrates how the pattern manages workflow execution. The diagram above shows more detailed steps following these key steps.</p> 
<ol> 
 <li><strong>Start the asynchronous operation</strong> for which you want to wait for completion. The asynchronous service responds with success (200 OK) and the state machine continues. Initiating an Amazon Translate batch translation operation is one example of such an asynchronous operation.</li> 
 <li><strong>Trigger the generalized callback pattern</strong> with the “Wait for Callback” capability. Pair the task token with the jobId in DynamoDB using the unique jobId as the primary key. Example: <pre><code class="lang-json">{
    id    = translationJobId,
    token = stepFunctionTaskToken
}</code></pre> </li> 
 <li><strong>Monitor for completion</strong>: When the asynchronous service completes the requested job, such as translation of documents, an event is created in EventBridge that contains the jobId and status. Example: <pre><code class="lang-json">{
    jobId  = translationJobId,
    status = complete
}
</code></pre> </li> 
 <li><strong>Resume workflow</strong>: The EventBridge rule triggers the workflow to resume, which looks up the task token using the jobId, resumes the paused Step Functions execution, and cleans up the database entry.</li> 
</ol> 
<p>Not every service creates events for every action, so validate that your service operation generates the expected events. For example, Macie does not create events when no findings are discovered. In these cases, implement more event generation mechanisms through <a href="https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/Subscriptions.html" rel="noopener noreferrer" target="_blank">Amazon CloudWatch Logs</a> subscriptions that trigger Lambda functions to create custom events.</p> 
<h2>Technical implementation of the solution</h2> 
<p>For rapid deployment of this post’s solution, AWS CDK users can use <a href="https://github.com/aws-samples/sample-cdk-generalized-callback-pattern-for-stepfunctions-state-machines/" rel="noopener noreferrer" target="_blank">this sample CDK pattern with all key components</a>. Alternatively, you can implement the individual components yourself by using the following steps, with each component customizable to your requirements.</p> 
<p>Some of the JSON-based snippets below are <a href="https://docs.aws.amazon.com/step-functions/latest/dg/concepts-amazon-states-language.html" rel="noopener noreferrer" target="_blank">Amazon States Language</a> (ASL) snippets, which is the language that defines an AWS Step Functions state machine. State machines can be built in the AWS Console using the <a href="https://docs.aws.amazon.com/step-functions/latest/dg/getting-started.html" rel="noopener noreferrer" target="_blank">drag and drop visual builder</a>, or with ASL. The visual builder generates this ASL and you can <a href="https://docs.aws.amazon.com/step-functions/latest/dg/getting-started.html" rel="noopener noreferrer" target="_blank">toggle to view/edit the workflow code (ASL)</a>.</p> 
<h3>Use a Step Functions task that supports “WaitForCallback” to store task token in DynamoDB</h3> 
<p>Use a Step Functions task that supports ”WaitForCallback” to store the task token in DynamoDB alongside the job ID from the asynchronous service.</p> 
<p>AWS services generate a unique ID for that service which refers to that job/request/action. DynamoDB holds the mappings between job IDs and task tokens, supporting multiple state machines paused in parallel with concurrent execution. To prevent clashes when different asynchronous services generate overlapping IDs (for example, if Service A and Service B both generate ID “12345”), use separate DynamoDB tables for each service to maintain ID uniqueness. The <a href="https://github.com/aws-samples/sample-cdk-generalized-callback-pattern-for-stepfunctions-state-machines" rel="noopener noreferrer" target="_blank">sample AWS CDK pattern</a> demonstrates this approach by providing dedicated DynamoDB tables and Step Functions state machines for each service integration. This ID-token structure allows for quick lookups for workflow resumption and cleanup.</p> 
<p>The following ASL accomplishes this by using a DynamoDB PutItem task:</p> 
<pre><code class="lang-json">"DynamoDB PutItem": {
    "Type": "Task",
    "Resource": "arn:aws:states:::dynamodb:putItem",
    "Parameters": {
        "TableName": "resumeTokenSessionTable",
        "Item": {
            "id":    { "S.$": "$.JobId" },
            "token": { "S.$": "$$.Task.Token" },
            "ttl":   { "S.$": "$.ttl" }
        },
        "ConditionExpression": "attribute_not_exists(id)"
    },
    "Next": "XXXX"
}</code></pre> 
<p>In this example, the Item object stores three values: the job ID ($.JobId), the task token ($$.Task.Token), and a TTL value ($.ttl). The ttl field configures <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html" rel="noopener noreferrer" target="_blank">Time to Live</a> for automatic cleanup based on your service’s expected completion time. Since this stores only three small string values, data usage per entry is minimal. The primary consideration is the number of concurrent operations, as each active asynchronous job requires one DynamoDB entry until completion or TTL expiration.</p> 
<p>The DynamoDB table uses “id” as the primary key and includes a “token” attribute. These fields are essential for the “WaitForCallback” pattern: the “id” (job ID) allows your asynchronous service to look up the correct entry, while the “token” (Step Functions task token) is what your service sends back to Step Functions to resume the paused workflow. The following JSON shows an example of these values:</p> 
<pre><code class="lang-json">{
    "id":    { "S": "xxxxxxxx-yyyy-zzzz-aaaa-bbbbbbbbbbbb" },
    "token": { "S": "11111111-2222-3333-4444-555555555555" },
    "ttl":   { "S": "1480550400" }
}</code></pre> 
<p>When your asynchronous service completes its work, it retrieves the task token using the job ID, then calls Step Functions with that token to resume execution from where it paused.</p> 
<p>The task token acts as a unique identifier for resuming execution at the exact pause point. To prevent overriding an existing record when a duplicate id is used, you can specify a “<a href="https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_PutItem.html#API_PutItem_RequestSyntax" rel="noopener noreferrer" target="_blank">ConditionExpression”</a>. This ASL shows just the ConditionExpression.</p> 
<pre><code class="lang-json">“ConditionExpression”: “attribute_not_exists(id)”</code></pre> 
<h3>Create an EventBridge rule to monitor event patterns from your asynchronous service</h3> 
<p>EventBridge integration forms the heart of the event-driven resumption mechanism. You can create EventBridge rules to monitor specific event patterns from asynchronous AWS services. Most AWS services automatically publish completion events to default EventBridge at no cost, and you can use the EventBridge rule wizard to identify correct event patterns. For services that do not publish events—such as Macie that creates no events when no findings are discovered—implement shims by using <a href="https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html" rel="noopener noreferrer" target="_blank">Amazon CloudWatch Logs</a> to trigger Lambda functions that generate custom events. This JSON shows the EventBridge Rule pattern definition.</p> 
<pre><code class="lang-json">"EventPattern": {
    "source": [
        "aws.translate"
    ],
    "detail-type": [
        "Translate TextTranslationJob State Change"
    ],
    "detail": {
        "jobStatus": [
            "COMPLETED"
        ],
    }
}</code></pre> 
<h3>Resume the workflow</h3> 
<p>At this point, you know the operation has completed, so you can safely resume the workflow. Using the job ID, call the <a href="https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_GetItem.html" rel="noopener noreferrer" target="_blank">DynamoDB GetItem operation</a> to receive the task token. This ASL shows the task definition to get the task token for a given job ID retrieved from the event notification.</p> 
<pre><code class="lang-json">"getResumeToken": {
    "Next": "sendTaskSuccess",
    "Type": "Task",
    "ResultPath": "$.getResumeToken",
    "Resource": "arn:aws:states:::dynamodb:getItem",
    "Parameters": {
        "Key": {
            "id": { "S.$": "$.id" }
        },
        "TableName": "resumeTokenSessionTable"
    }
}</code></pre> 
<p>Use the task token to resume the workflow and then delete the DynamoDB entry for cleanup. This ASL shows the task definition to use the task token to resume the state machine at the point where it was paused at.</p> 
<pre><code class="lang-json">"sendTaskSuccess": {
    "Next": "deleteResumeToken",
    "Type": "Task",
    "ResultPath": "$.sendTaskSuccess",
    "Resource": "arn:aws:states:::aws-sdk:sfn:sendTaskSuccess",
    "Parameters": {
        "TaskToken.$": "$.getResumeToken.Item.token.S",
        "Output": {
            "status": "resume"
        }
    }
}</code></pre> 
<p>This ASL shows the task definition to clean up the DynamoDB to remove the used task token.</p> 
<pre><code class="lang-json">"deleteResumeToken": {
    "End": true,
    "Type": "Task",
    "Resource": "arn:aws:states:::dynamodb:deleteItem",
    "Parameters": {
        "Key": {
            "id": { "S.$": "$.id" }
        },
        "TableName": "resumeTokenSessionTable"
    }
}</code></pre> 
<p>This completes the technical implementation of our solution. With all components in place—the WaitForCallback task, EventBridge rules, workflow resumption logic, and DynamoDB storage—you now have a fully functional generalized callback pattern implementation that eliminates polling and efficiently manages asynchronous operations. </p> 
<p>Now that we’ve established how to implement the generalized callback pattern technically, let’s explore the best practices and important considerations that will help you optimize and secure your implementation.</p> 
<h2>Best practices and considerations</h2> 
<p>When implementing the generalized callback pattern in AWS Step Functions, it’s essential to understand and apply best practices that optimize costs, enhance security, and ensure efficient operation. This section outlines key considerations and recommendations for implementing the pattern effectively, focusing on cost optimization strategies and security measures that help maintain a robust and secure serverless workflow. By following these guidelines, you can maximise the benefits of the generalized callback pattern while minimising potential risks and unnecessary expenses.</p> 
<h3>Optimize costs by using this post’s generalized callback pattern</h3> 
<p>Managing costs for long-running asynchronous operations can present challenges. Traditional polling accumulates unnecessary expenses through repeated state transitions and execution time, but this post’s generalized callback pattern is an event-driven approach that significantly reduces operational costs.</p> 
<h4>Eliminate polling costs and minimize execution time</h4> 
<p>The generalized callback pattern reduces costs by eliminating polling transitions and pausing execution during wait periods. For standard workflows billed at $0.000025 per state transition, using just two transitions instead of continuous polling achieves approximately an 87% cost reduction. A 15-minute translation job polling every minute would need 15 transitions as opposed to two with the generalized callback pattern. For express workflows billed at $0.000001 per request and $0.00001667 per GB-second, the pattern delivers significant savings through reduced request count and minimal execution time. Traditional polling keeps workflows active during the entire operation, accumulating execution time charges. By contrast, the generalized callback pattern eliminates execution time charges during the wait period. In the translation job example mentioned previously in this paragraph, this could reduce the execution time from more than 15 minutes to just the seconds needed to start jobs and complete processes.</p> 
<h4>Increase resource efficiency</h4> 
<p>The callback pattern increases resource efficiency by removing constant polling, resulting in substantial reduction in CloudWatch logging and associated monitoring costs. This creates a more cost-effective solution with a reduced AWS resource footprint.</p> 
<h4>Further cost-optimize the callback pattern</h4> 
<p>Enhance cost efficiency through<a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-cost-optimization.html" rel="noopener noreferrer" target="_blank"> DynamoDB optimizations</a>. Choose on-demand mode for unpredictable workloads or provisioned mode with auto scaling for consistent patterns, configure auto scaling settings based on usage, and<a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html" rel="noopener noreferrer" target="_blank"> implement TTL</a> to automatically remove expired items without consuming write capacity.</p> 
<h3>Security considerations for the callback pattern</h3> 
<p>The callback pattern involves storing task tokens, processing events, and managing workflow resumption across multiple AWS services. Implementing proper access controls is essential to protect the integrity of your workflows and prevent unauthorized access or manipulation of the pattern’s components. </p> 
<p>This section outlines the security considerations for the callback pattern, focusing on access controls for data storage and event processing.</p> 
<h4>Data storage security</h4> 
<p>Enable <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/EncryptionAtRest.html" rel="noopener noreferrer" target="_blank">DynamoDB encryption at rest</a> by using AWS owned or user managed <a href="https://aws.amazon.com/kms/" rel="noopener noreferrer" target="_blank">AWS Key Management Service (AWS KMS)</a> keys. Implement identity-based policies by defining the Step Functions <a href="https://aws.amazon.com/iam/" rel="noopener noreferrer" target="_blank">AWS Identity and Access Management (IAM)</a> role actions (such as PutItem, GetItem, and DeleteItem) and resource-based policies that specify which IAM principals can access the table. Together, these help ensure that only authorized state machines access token storage and operations are limited to minimum permissions. Also, <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html" rel="noopener noreferrer" target="_blank">configure TTL</a> to automatically remove expired tokens so that these tokens do not accidentally get reused, which can result in errors with resuming the relevant AWS Step Function workflows.</p> 
<h4>Event processing security</h4> 
<p>Scope EventBridge rules precisely to match only specific necessary events. For Amazon Translate job completion, rules should explicitly match only translation job completion events, thus preventing unauthorized triggers. IAM roles should follow <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html#grant-least-privilege" rel="noopener noreferrer" target="_blank">least-privilege principles</a> so that only specific actions can cause workflows to resume.</p> 
<h2>Conclusion</h2> 
<p>The callback pattern presented in this post provides a solution for managing long-running asynchronous operations in serverless architectures. You can use the Step Functions “Wait for Callback” task state with EventBridge and DynamoDB to transform asynchronous services into synchronous workflows without the overhead of polling. This pattern reduces costs, improves efficiency through event-driven architecture, and maintains security through proper access controls. You can use the provided <a href="https://github.com/aws-samples/sample-cdk-generalized-callback-pattern-for-stepfunctions-state-machines/" rel="noopener noreferrer" target="_blank"> CDK implementation</a> to implement this pattern and adapt it to your specific needs while following recommended security and cost optimization practices.&nbsp;</p> 
<hr /> 
<h3>About the authors</h3> 
<p style="clear: both;"><img alt="" class=" wp-image-4649 alignleft" height="129" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/10/28/maria-1.jpeg" width="100" /><strong>Maria John</strong> is a Senior Solutions Architect at Amazon Web Services, helping customers build solutions on AWS.</p> 
<p style="clear: both;"><img alt="" class="size-full wp-image-4648 alignleft" height="130" src="https://d2908q01vomqb2.cloudfront.net/f1f836cb4ea6efb2a0b1b99f41ad8b103eff4b59/2024/04/16/profile-100x219-1.png" width="100" /><strong>Philip Whiteside</strong>&nbsp;is a Senior Solutions Architect at Amazon Web Services. Philip is passionate about overcoming barriers by utilizing technology.</p>
