---
title: "Orchestrating large-scale document processing with AWS Step Functions and Amazon Bedrock batch inference"
url: "https://aws.amazon.com/blogs/compute/orchestrating-large-scale-document-processing-with-aws-step-functions-and-amazon-bedrock-batch-inference/"
date: "Wed, 26 Nov 2025 21:41:51 +0000"
author: "Brian Zambrano"
feed_url: "https://aws.amazon.com/blogs/compute/category/application-services/aws-step-functions/feed/"
---
<p>Organizations often have large volumes of documents containing valuable information that remains locked away and unsearchable. This solution addresses the need for a <strong>scalable, automated text extraction and knowledge base pipeline</strong> that transforms static document collections into intelligent, searchable repositories for generative AI applications.</p> 
<p>Organizations can automate the extraction of both content and structured metadata to build comprehensive knowledge bases that power retrieval-augmented generation (RAG) solutions while significantly reducing manual processing costs and time-to-value. The architecture not only demonstrates the processing of 500 research papers automatically, but also scales to handle enterprise document volumes cost-effectively through the <a href="https://aws.amazon.com/bedrock/" rel="noopener noreferrer" target="_blank">Amazon Bedrock</a> batch inference pricing model.</p> 
<h2>Overview</h2> 
<p><a href="https://aws.amazon.com/blogs/machine-learning/automate-amazon-bedrock-batch-inference-building-a-scalable-and-efficient-pipeline/" rel="noopener noreferrer" target="_blank">Amazon Bedrock batch inference</a> is a feature of Amazon Bedrock that offers a 50% discount on inference requests. Although Amazon Bedrock schedules and runs the batch job (needing a minimum of 100 inference requests) as capacity becomes available, the inference won’t be real-time. For use cases where you can accommodate minutes to hours of latency, Amazon Bedrock batch inference is a good option.</p> 
<p>This post demonstrates how to build an automated, serverless pipeline using <a href="https://aws.amazon.com/step-functions/" rel="noopener noreferrer" target="_blank">AWS Step Functions</a>, <a href="https://aws.amazon.com/textract/" rel="noopener noreferrer" target="_blank">Amazon Textract</a>, Amazon Bedrock batch inference, and <a href="https://aws.amazon.com/bedrock/knowledge-bases/" rel="noopener noreferrer" target="_blank">Amazon Bedrock Knowledge Bases</a> to extract text, create metadata, and load it into a knowledge base at scale. The example solution processes 500 research papers in PDF format from <a href="https://www.amazon.science/" rel="noopener noreferrer" target="_blank">Amazon Science</a>, extracts text using Amazon Textract, generated structured metadata with Amazon Bedrock batch inference and the <a href="https://aws.amazon.com/ai/generative-ai/nova/" rel="noopener noreferrer" target="_blank">Amazon Nova Pro</a> model, and loads the final output, including Amazon Bedrock Knowledge Base filter, into an Amazon Bedrock Knowledge Base.</p> 
<h2>Architecture</h2> 
<p>This solution uses Step Functions with parallel Amazon Textract job processing through child workflows run by <a href="https://docs.aws.amazon.com/step-functions/latest/dg/state-map-distributed.html" rel="noopener noreferrer" target="_blank">Distributed Map</a>. You can use the concurrency controls offered by Distributed Map to process documents as quickly as possible within your Amazon Textract quotas. Increasing processing speed necessitates adjusting your Amazon Textract quota and updating the Distributed Map configuration. Amazon Bedrock batch inference handles concurrency, scaling, and throttling. This means that you can create the job without managing these complexities.</p> 
<p>In this example implementation, the solution processes research papers to extract metadata such as:</p> 
<ul> 
 <li>Code availability and repository locations</li> 
 <li>Dataset availability and access methods</li> 
 <li>Research methodology types</li> 
 <li>Reproducibility indicators</li> 
 <li>Other relevant research attributes</li> 
</ul> 
<p>The high-level parts of this solution include:</p> 
<ul> 
 <li>Extracting text from PDF documents with Amazon Textract in parallel, through Step Functions Distributed Map.</li> 
 <li>Analyzing extracted text using Amazon Bedrock batch inference to extract structured metadata.</li> 
 <li>Loading extract text and metadata into a searchable knowledge base using Amazon Bedrock Knowledge Bases with <a href="https://aws.amazon.com/opensearch-service/features/serverless/" rel="noopener noreferrer" target="_blank">Amazon OpenSearch Serverless</a>.</li> 
</ul> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/11/26/computeblog-2442-1.png"><img alt="Complete architecture diagram" class="alignnone wp-image-25420 size-full" height="1303" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/11/26/computeblog-2442-1.png" style="margin: 10px 0px 10px 0px;" width="819" /></a></p> 
<p>Figure 1. Complete architecture diagram</p> 
<h2>Prerequisites</h2> 
<p>The following prerequisites are necessary to complete this solution:</p> 
<ul> 
 <li>Access to an <a href="https://portal.aws.amazon.com/gp/aws/developer/registration/index.html" rel="noopener noreferrer" target="_blank">AWS account</a> through the <a href="https://aws.amazon.com/console/" rel="noopener noreferrer" target="_blank">AWS Management Console</a> and the <a href="https://aws.amazon.com/cli" rel="noopener noreferrer" target="_blank">AWS Command Line Interface (AWS CLI)</a>. The <a href="https://aws.amazon.com/iam" rel="noopener noreferrer" target="_blank">AWS Identity and Access Management (IAM)</a> user that you use must have permissions to make the necessary AWS service calls and manage AWS resources mentioned in this post. While providing permissions to the IAM user, follow the <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html#grant-least-privilege" rel="noopener noreferrer" target="_blank">principle of least-privilege</a>.</li> 
 <li><a href="https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html" rel="noopener noreferrer" target="_blank">AWS CLI</a> installed and configured. If you are using long-term credentials such as access keys, then follow <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html" rel="noopener noreferrer" target="_blank">manage access keys for IAM users</a> and <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/securing_access-keys.html" rel="noopener noreferrer" target="_blank">secure access keys</a> for best practices.</li> 
 <li><a href="https://git-scm.com/book/en/v2/Getting-Started-Installing-Git" rel="noopener noreferrer" target="_blank">Git Installed</a>.</li> 
 <li>Python 3.13+ installed.</li> 
 <li>Node and npm installed.</li> 
 <li><a href="https://aws.amazon.com/cdk/" rel="noopener noreferrer" target="_blank">AWS Cloud Development Kit (AWS CDK)</a> installed.</li> 
</ul> 
<h2>Running the solution</h2> 
<p>The complete solution uses AWS CDK to implement two <a href="https://aws.amazon.com/cloudformation/" rel="noopener noreferrer" target="_blank">AWS CloudFormation</a> stacks:</p> 
<ol> 
 <li>BedrockKnowledgeBaseStack: Creates the knowledge base infrastructure</li> 
 <li>SFNBatchInferenceStack: Implements the main processing workflow</li> 
</ol> 
<p>First, clone the GitHub repository into your local development environment and install the requirements:</p> 
<p><code>git clone https://github.com/aws-samples/sample-step-functions-batch-inference.git .</code></p> 
<p><code>cd sample-step-functions-batch-inference</code></p> 
<p><code>npm install</code></p> 
<p>Next, deploy the solution using AWS CDK:</p> 
<p><code>cdk deploy --all</code></p> 
<p>After deploying the cdk stacks, upload your data sources (PDF files) into the AWS CDK-created <a href="https://aws.amazon.com/s3/" rel="noopener noreferrer" target="_blank">Amazon S3</a> input bucket. In this example, I uploaded 500 Amazon Science papers. The input bucket name is included in the AWS CDK outputs:</p> 
<p>Outputs:</p> 
<p><code>SFNBatchInference.BatchInputBucketName = sfnbatchinference-batchinputbucket11aaa222-nrjki8tewwww</code></p> 
<h3>Parallel text extraction</h3> 
<p>The process begins when you upload a manifest.json file to the input bucket. The manifest file lists the files for processing, which already exist in the input bucket. The filenames listed in manifest.json define what constitutes a single processing job run. To create another run, you would create a different manifest.json and upload it to the same S3 bucket.</p> 
<div class="hide-language"> 
 <pre><code class="lang-json">[
  {
    "filename": "flexecontrol-flexible-and-efficient-multimodal-control-for-text-to-image-generation.pdf"
  },
  {
    "filename": "adaptive-global-local-context-fusion-for-multi-turn-spoken-language-understanding.pdf"
  }
]
</code></pre> 
</div> 
<p>The AWS CDK definition for the input bucket includes <a href="https://aws.amazon.com/eventbridge/" rel="noopener noreferrer" target="_blank">Amazon EventBridge</a> notifications and creates a rule that triggers the Step Functions workflow whenever a manifest.json file is uploaded.</p> 
<div class="hide-language"> 
 <pre><code class="lang-ts">private createS3Buckets() {
    const batchBucket = new s3.Bucket(this, "BatchInputBucket", {
      removalPolicy: cdk.RemovalPolicy.DESTROY,
      autoDeleteObjects: true,
    })
    batchBucket.enableEventBridgeNotification()

    new cdk.CfnOutput(this, "BatchInputBucketName", {
      value: batchBucket.bucketName,
      description: "Name of input bucket to send PDF documents that Textract will read.",
    })

    const manifestFileCreatedRule = new eventBridge.Rule(this, "ManifestFileCreatedRule", {
      eventPattern: {
        source: ["aws.s3"],
        detailType: ["Object Created"],
        detail: {
          bucket: {
            name: [batchBucket.bucketName],
          },
          object: {
            key: ["manifest.json"],
          },
        },
      },
    })

    return { batchBucket, manifestFileCreatedRule }
  }
</code></pre> 
</div> 
<p>The first step in the Step Functions workflow is a Distributed Map run that performs the following actions for each PDF in the manifest file:</p> 
<ol> 
 <li>Starts an Amazon Textract job, providing an <a href="https://aws.amazon.com/sns/" rel="noopener noreferrer" target="_blank">Amazon Simple Notification Service (Amazon SNS)</a> topic for completion notification.</li> 
 <li>Writes the Step Functions task token to <a href="https://aws.amazon.com/dynamodb/" rel="noopener noreferrer" target="_blank">Amazon DynamoDB</a>, pausing the individual child workflow.</li> 
 <li>Processes the Amazon SNS message when the Amazon Textract job completes, triggering an <a href="https://aws.amazon.com/lambda/" rel="noopener noreferrer" target="_blank">AWS Lambda</a> function.</li> 
 <li>Uses a Lambda function to retrieve the task token from DynamoDB using the Amazon Textract JobId.</li> 
 <li>Fetches the raw results from Amazon Textract, organizes the text for readability, and writes results to an S3 bucket</li> 
</ol> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/11/26/computeblog-2442-2.png"><img alt="First step in the Step Functions workflow" class="alignnone wp-image-25425 size-full" height="896" src="//d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/11/26/computeblog-2442-2.png" style="margin: 10px 0px 10px 0px;" width="1429" /></a></p> 
<p>A key component of this architecture is the callback pattern that Amazon Textract supports using the NotificationChannel option, as shown in the preceding figure. The AWS CDK definition the Step Functions state that starts the Amazon Textract job is shown in the following.</p> 
<div class="hide-language"> 
 <div class="hide-language"> 
  <pre><code class="lang-ts">const startTextractStep = new tasks.CallAwsService(this, "StartTextractJob", {
  service: "textract",
  action: "startDocumentAnalysis",
  resultPath: "$.textractOutput",
  parameters: {
    DocumentLocation: {
      S3Object: {
        Bucket: sourceBucket.bucketName,
        Name: sfn.JsonPath.stringAt("$.filename"),
      },
    },
    FeatureTypes: ["LAYOUT"],
    NotificationChannel: {
      RoleArn: textractRoleArn,
      SnsTopicArn: snsTopicArn,
    },
  },
  iamResources: ["*"],
})
</code></pre> 
 </div> 
</div> 
<p>The Lambda function that handles task tokens extracts the Amazon Textract JobId from the Amazon SNS message, fetches the TaskToken from DynamoDB, and resumes the Step Functions Workflow by sending the TaskToken:</p> 
<div class="hide-language"> 
 <pre><code class="lang-ts">from aws_lambda_powertools.utilities.data_classes import SNSEvent, event_source

@event_source(data_class=SNSEvent)
def handle_textract_task_complete(event, context):
    # Multiple records can be delivered in a single event
    for record in event.records:
        sns_message = json.loads(record.sns.message)
        textract_job_id = sns_message["JobId"]

        # Get both task token and original file from DynamoDB
        ddb_item = _get_item_from_ddb(textract_job_id)

        # Send both the job ID and original file name in the response
        _send_task_success(
            ddb_item["TaskToken"],
            {
                "TextractJobId": textract_job_id,
                "OriginalFile": ddb_item["OriginalFile"],
            },
        )
        # Delete the task token from DynamoDB after use
        _delete_item_from_ddb(textract_job_id)

def _send_task_success(task_token: str, output: None | dict = None) -&gt; None:
    """Sends task success to Step Functions with the provided output"""
    sfn = boto3.client("stepfunctions")
    sfn.send_task_success(taskToken=task_token, output=json.dumps(output or {}))
</code></pre> 
</div> 
<p>The Distributed Map runs up to 10 child workflows concurrently, controlled by the maxConcurrency setting. Although Step Functions supports running up to 10,000 child workflow executions, the practical concurrency for this solution is constrained by Amazon Textract quotas. The startDocumentAnalysis API has a default quota of 10 requests per second (RPS), which means you must consider this limit when scaling your document processing workloads and potentially request quota increases for higher throughput requirements.</p> 
<div class="hide-language"> 
 <pre><code class="lang-ts">const distributedMap = new sfn.DistributedMap(this, "DistributedMap", {
  mapExecutionType: sfn.StateMachineType.STANDARD,
  maxConcurrency: 10,
  itemReader: new sfn.S3JsonItemReader({
    bucket: sourceBucket,
    key: "manifest.json",
  }),
  resultPath: "$.files",
}
</code></pre> 
</div> 
<h3>Running Amazon Bedrock batch inference</h3> 
<p>When all of the Amazon Textract jobs finish, the Distributed Map state creates an Amazon Bedrock batch inference input file, launches the Amazon Bedrock inference job, and waits for it to complete.</p> 
<ol start="6"> 
 <li>A Lambda function collects text results from Amazon S3 and creates an Amazon Bedrock batch inference input file with custom prompts.</li> 
 <li>The workflow starts the Amazon Bedrock batch inference job by calling createModelInvocationJob and sending the batch inference input file as input.</li> 
 <li>The workflow pauses and stores the task token in DynamoDB.</li> 
 <li>An EventBridge rule matches completed Amazon Bedrock batch inference events, and upon job completion and triggers a Lambda function. The Lambda function retrieves the task token and resumes the workflow, as shown in the following figure.</li> 
</ol> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/11/26/computeblog-2442-3.png"><img alt="Lambda function retrieves the task token and resumes the workflow" class="alignnone wp-image-25424 size-full" height="1308" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/11/26/computeblog-2442-3.png" style="margin: 10px 0px 10px 0px;" width="1429" /></a></p> 
<p>A batch inference input is a single jsonl file with multiple entries such as the following example. The prompt in each inference request instructs the large language model (LLM) to analyze the paper and extract metadata. Read the full <a href="https://github.com/aws-samples/sample-step-functions-batch-inference/blob/956b5fc645c7de5f43d650d21ef9df011db67170/src/bedrock-batcher/handler.py#L41-L81" rel="noopener noreferrer" target="_blank">prompt template in the GitHub repository</a>.</p> 
<div class="hide-language"> 
 <pre><code class="lang-json">{
  "recordId": "c1b8a3b2086141f963",
  "modelInput": {
    "messages": [
      {
        "role": "user",
        "content": [
          {
            "text": "Analyze the following research paper transcript and extract metadata about code and dataset availability. Extract the following metadata from this research paper transcript:\n\n1. **has_code**: Does the paper mention or link to source code? (true/false) ...... Return only valid JSON matching the schema above. Do not include any text outside of the JSON structure."
          }
        ]
      }
    ],
    "inferenceConfig": { "maxTokens": 4096 }
  }
}
</code></pre> 
</div> 
<h3>Populating the Amazon Bedrock Knowledge Base</h3> 
<p>After the batch inference completes, the workflow does the following:</p> 
<ol start="10"> 
 <li>Extracts inference results and creates metadata files based on the Amazon Bedrock inference results (example metadata shown in the following figure).</li> 
 <li>Starts an Amazon Bedrock Knowledge Base ingestion job.</li> 
 <li>Monitors the ingestion job status using Step Functions Wait and Choice states.</li> 
 <li>Sends a completion notification through Amazon SNS.</li> 
</ol> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/11/26/computeblog-2442-4.png"><img alt="Populating the Amazon Bedrock Knowledge Base" class="alignnone wp-image-25423 size-full" height="1689" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/11/26/computeblog-2442-4.png" style="margin: 10px 0px 10px 0px;" width="1332" /></a></p> 
<p>The following shows the example metadata format:</p> 
<div class="hide-language"> 
 <pre><code class="lang-json">{
  "metadataAttributes": {
    "has_code": true,
    "has_dataset": false,
    "code_availability": "publicly_available",
    "dataset_availability": "not_available",
    "research_type": "methodology",
    "is_reproducible": true,
    "code_repository_url": "https://github.com/amazon-science/PIXELS"
  }
}
</code></pre> 
</div> 
<h2>Testing the knowledge base</h2> 
<p>After the workflow completes successfully, you can test the knowledge base to verify that the documents and metadata have been properly ingested and are searchable. There are two practical methods for testing an Amazon Bedrock Knowledge Base:</p> 
<ol> 
 <li>Using the Console</li> 
 <li>Using the AWS SDK to run a query</li> 
</ol> 
<h2>Testing through the Console</h2> 
<p>The Console provides an intuitive interface for testing your knowledge base queries with metadata filters:</p> 
<ol> 
 <li>Navigate to the <a href="https://console.aws.amazon.com/bedrock/" rel="noopener noreferrer" target="_blank">Amazon Bedrock console</a>.</li> 
 <li>In the left navigation pane, choose <strong>Knowledge Bases</strong> under the <strong>Build </strong>section.</li> 
 <li>Choose the knowledge base created by the AWS CDK deployment (the name will be output by the AWS CDK stack).</li> 
 <li>Choose the <strong>Test</strong> button in the upper right corner.</li> 
 <li>In the test interface, choose your preferred foundation model (FM) (such as Amazon Nova Pro).</li> 
 <li>Expand the <strong>Configurations</strong> column, then navigate to the <strong>Filters </strong>section.</li> 
 <li>Configure filters based on the extracted metadata, as shown in the following figure.</li> 
</ol> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/11/25/computeblog-2442-5.png"><img alt="Configure filters based on the extracted metadata" class="alignnone wp-image-25422 size-full" height="247" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/11/25/computeblog-2442-5.png" style="margin: 10px 0px 10px 0px;" width="383" /></a></p> 
<p>Enter a natural language query related to your documents, for example: “Recent research on retrieval augmented generation?”</p> 
<p>The console displays the generated response along with source attributions showing which documents were retrieved and used to formulate the answer, filtered by your specified metadata attributes, as shown in the following figure.</p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/11/25/compute-2442-6.png"><img alt="A chat example" class="alignnone wp-image-25421 size-full" height="1074" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/11/25/compute-2442-6.png" style="margin: 10px 0px 10px 0px;" width="1095" /></a></p> 
<h2>Testing via API</h2> 
<p>For programmatic testing and integration into applications, use the AWS SDK with metadata filtering. The following is a Python example using boto3:</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">model_arn = "arn:aws:bedrock:us-east-1::foundation-model/amazon.nova-pro-v1:0"

# Query for papers with publicly available code
response = bedrock_agent_runtime.retrieve_and_generate(
    input={'text': "What recent research has been done on RAG?"},
    retrieveAndGenerateConfiguration={
        'type': 'KNOWLEDGE_BASE',
        'knowledgeBaseConfiguration': {
            'knowledgeBaseId': knowledge_base_id,
            'modelArn': model_arn,
            'retrievalConfiguration': {
                'vectorSearchConfiguration': {
                    'numberOfResults': 5,
                    'filter': {"equals": {"key": "has_code", "value": True}},
                }
            },
        },
    },
)

# Display results
print(f"Response: {response['output']['text']}\n")
print("Source Documents:")

for citation in response.get('citations', []):
    for reference in citation.get('retrievedReferences', []):
        metadata = reference.get('metadata', {})
        print(f" Document: {reference['location']['s3Location']['uri']}\n")
</code></pre> 
</div> 
<p>The following is the test script output:</p> 
<div class="hide-language"> 
 <pre><code class="lang-bash">Response: Recent research on Retrieval-Augmented Generation (RAG) has focused on enhancing the system's ability to dynamically retrieve and utilize relevant information from a Vector Database (VDB) to improve decision-making and performance. Key innovations include:

1. **Dynamic Retrieval and Utilization**: The system is designed to query the VDB for contextually relevant past experiences, which significantly improves decision quality and accelerates performance by leveraging a growing repository of relevant experiences.

2. **Teacher-Student Instructional Tuning**: A novel mechanism where a Teacher agent refines a Student agent's core policy through direct interaction. The Teacher generates a modified SYSTEM prompt based on the Student's actions, creating a meta-learning loop that enhances the Student's reasoning policy over time.
</code></pre> 
</div> 
<h2>Conclusion</h2> 
<p>This solution demonstrates how to combine multiple AWS AI and serverless services to build a scalable document processing pipeline. Organizations can use AWS Step Functions for orchestration, Amazon Textract for document processing, Amazon Bedrock batch inference for intelligent content analysis, and Amazon Bedrock Knowledge Bases for searchable storage. In turn, they can automate the extraction of insights from large document collections while optimizing costs.</p> 
<p>Following this solution, you can build a solid foundation for production-scale document processing pipelines that maintain the flexibility to adapt to your specific requirements while making sure of reliability, scalability, and operational excellence. Follow this link to learn more about <a href="https://serverlessland.com/" rel="noopener noreferrer" target="_blank">serverless architectures</a>.</p>
