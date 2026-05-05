---
title: "Processing Amazon S3 objects at scale with AWS Step Functions Distributed Map S3 prefix"
url: "https://aws.amazon.com/blogs/compute/processing-amazon-s3-objects-at-scale-with-aws-step-functions-distributed-map-s3-prefix/"
date: "Fri, 24 Oct 2025 21:22:10 +0000"
author: "Biswanath Mukherjee"
feed_url: "https://aws.amazon.com/blogs/compute/category/application-services/aws-step-functions/feed/"
---
<p>If you’re building large scale enterprise applications, you’ve likely faced the complexities of processing large volumes of data files. Whether you’re analyzing your application logs, processing customer data files, or transforming machine learning datasets, you know the complexity involved in orchestrating workflows. You’ve probably written nested workflows and additional custom code to process objects from <a href="https://aws.amazon.com/s3/" rel="noopener noreferrer" target="_blank">Amazon Simple Storage Service (Amazon S3</a>) buckets.</p> 
<p>With <a href="https://docs.aws.amazon.com/step-functions/latest/dg/state-map-distributed.html" rel="noopener noreferrer" target="_blank">AWS Step Functions Distributed Map</a>, you can process large scale datasets by running concurrent iterations of workflow steps across data entries in parallel, achieving massive scale with simplified management.</p> 
<p>With the new prefix-based iteration feature and <code><a href="https://docs.aws.amazon.com/step-functions/latest/dg/input-output-itemreader.html#itemsource-example-s3-object-data-flatten" rel="noopener noreferrer" target="_blank">LOAD_AND_FLATTEN</a></code> transformation parameter for <a href="https://docs.aws.amazon.com/step-functions/latest/dg/state-map-distributed.html" rel="noopener noreferrer" target="_blank">Distributed Map</a>, your workflows can now iterate over S3 objects under a specified prefix using&nbsp;<code><a href="https://docs.aws.amazon.com/AmazonS3/latest/API/API_ListObjectsV2.html" rel="noopener noreferrer" target="_blank">S3ListObjectsV2</a></code> to process their contents in a single Map state, avoiding nested workflows and reducing operational complexity.</p> 
<table border="2px" cellpadding="10px" style="border-color: #FF9900;"> 
 <tbody> 
  <tr> 
   <td><strong>This post is part of a series of post about AWS Step Functions Distributed Map:</strong><p></p> <p></p> 
    <ul> 
     <li>Processing Amazon S3 objects at scale with AWS Step Functions Distributed Map S3 prefix</li> 
     <li><a href="https://aws.amazon.com/blogs/compute/optimizing-nested-json-array-processing-using-aws-step-functions-distributed-map/" rel="noopener" target="_blank">Optimizing nested JSON array processing using AWS Step Functions Distributed Map</a></li> 
     <li><a href="https://aws.amazon.com/blogs/compute/orchestrating-big-data-processing-with-aws-step-functions-distributed-map/" rel="noopener" target="_blank">Orchestrating big data processing with AWS Step Functions Distributed Map</a></li> 
    </ul> </td> 
  </tr> 
 </tbody> 
</table> 
<p>In this post, you’ll learn how to process Amazon S3 objects at scale with the new AWS Step Functions Distributed Map S3 prefix and transformation capabilities.</p> 
<h2>Use case: Application log processing and summarization</h2> 
<p>You’ll build a sample Step Functions state machine that demonstrates processing of all the log files from the given S3 prefix using a Distributed Map. You’ll analyze all the log files to build a summary <code>INFO</code>, <code>WARNING</code> and <code>ERROR</code> messages in the log file on hourly basis. The following diagram presents the AWS Step Functions state machine:</p> 
<div class="wp-caption aligncenter" id="attachment_24860" style="width: 587px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/10/23/Computeblog-2492-image-1.png"><img alt="Log files processing workflow" class="size-full wp-image-24860" height="891" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/10/23/Computeblog-2492-image-1.png" style="margin: 10px 0px 10px 0px;" width="577" /></a>
 <p class="wp-caption-text" id="caption-attachment-24860"><em>Log files processing workflow</em></p>
</div> 
<ol> 
 <li>The state machine iterates over all the log files from the specified S3 prefix using S3 <code>ListObjectsV2</code> and process them using AWS Step Functions Distributed Map.</li> 
 <li>For each log file entry, the state machine puts hourly <code>ErrorCount</code> metric into <a href="https://aws.amazon.com/cloudwatch/" rel="noopener noreferrer" target="_blank">Amazon CloudWatch</a>.</li> 
 <li>The state machine then stores hourly metrics count in a <a href="https://aws.amazon.com/dynamodb/" rel="noopener noreferrer" target="_blank">Amazon DynamoDB</a> table.</li> 
 <li>The state machine then invokes an <a href="https://aws.amazon.com/lambda/" rel="noopener noreferrer" target="_blank">AWS Lambda</a> function to perform metrics aggregation.</li> 
</ol> 
<p>The following is an example of the parameters in an&nbsp;<code><a href="https://docs.aws.amazon.com/step-functions/latest/dg/input-output-itemreader.html" rel="noopener noreferrer" target="_blank">ItemReader</a></code> configured to iterate over the content of S3 objects using S3 <code>ListObjectsV2</code>.</p> 
<div class="hide-language"> 
 <pre><code class="lang-json">{
  "QueryLanguage": "JSONata",
  "States": {
    ...
    "Map": {
        ...
        "ItemReader": {
            "Resource": "arn:aws:states:::s3:listObjectsV2",
            "ReaderConfig": {
                // InputType is required if Transformation is LOAD_AND_FLATTEN. Use one of the given values
                "InputType": "CSV | JSON | JSONL | PARQUET",
                // Transformation is OPTIONAL and defaults to NONE if not present
                "Transformation": "NONE | LOAD_AND_FLATTEN" 
            },
            "Arguments": {
                "Bucket": "amzn-s3-demo-bucket1",
                "Prefix": "{% $states.input.PrefixKey %}"
            }
        },
        ...
    }
}</code></pre> 
</div> 
<p>With the&nbsp;<code>LOAD_AND_FLATTEN</code>&nbsp;option, your state machine will do the following:</p> 
<ol> 
 <li>Read the&nbsp;actual content&nbsp;of each object listed by the Amazon S3&nbsp;<code>ListObjectsV2</code>&nbsp;call.</li> 
 <li>Parse the content based on <code>InputType</code> (CSV, JSON, JSONL, Parquet).</li> 
 <li>Create items from the file contents (rows/records) rather than metadata.</li> 
</ol> 
<p>We recommend including a trailing slash on your prefix. For example, if you select data with a prefix of <code>folder1</code>, your state machine will process both <code>folder1/myData.csv</code> and <code>folder10/myData.csv</code>. Using <code>folder1/</code> will strictly process only one folder. All of the objects listed by <code>prefix</code> need to be in the same data format. For example, if you are selecting <code>InputType</code> as <code>JSONL</code>, your S3 prefix should contain only <code>JSONL</code> files and not a mix of other types.</p> 
<p>The <a href="https://docs.aws.amazon.com/step-functions/latest/dg/input-output-contextobject.html#contextobject-map" rel="noopener noreferrer" target="_blank">context object</a> is an internal JSON structure that is available during an execution. The context object contains information about your state machine and execution. Your workflows can reference the context object in a <code>JSONata</code> expression with <code>$states.context</code>.</p> 
<p>Within a&nbsp;<a href="https://docs.aws.amazon.com/step-functions/latest/dg/state-map.html" rel="noopener noreferrer" target="_blank">Map state</a>, the context object includes the following data:</p> 
<div class="hide-language"> 
 <pre><code class="lang-json">"Map": {
   "Item": {
      "Index" : Number,
      "Key"   : "String", // Only valid for JSON objects
      "Value" : "String",
      "Source": "String"
   }
}</code></pre> 
</div> 
<p>For each Map iteration, the <code>Index</code> contains the index number for the array item that is being currently processed.</p> 
<p>A <code>Key</code> is only available when iterating over JSON objects. <code>Value</code> contains the array item being processed. For example, for the following input JSON object, Names&nbsp;will be assigned to <code>Key</code>&nbsp;and&nbsp;<code>{"Bob", "Cat"}</code>&nbsp;will be assigned to <code>Value</code>.</p> 
<div class="hide-language"> 
 <pre><code class="lang-json">{
	"Names": {"Bob", "Cat"}
} </code></pre> 
</div> 
<p><code>Source</code> contains one of the following:</p> 
<ul> 
 <li>For state input:&nbsp;<code>STATE_DATA</code></li> 
 <li>For&nbsp;Amazon S3 <code>LIST_OBJECTS_V2</code>&nbsp;with&nbsp;<code>Transformation=NONE</code>, the value will show the S3 URI for the bucket. For example:&nbsp;<code>S3://amzn-s3-demo-bucket1</code></li> 
 <li>For all the other input types, the value will be the Amazon S3 URI. For example:&nbsp;<code>S3://amzn-s3-demo-bucket1/object-key</code></li> 
</ul> 
<p>Using <code>LOAD_AND_FLATTEN</code> and the <code>Source</code> field, you can connect child executions to their sources.</p> 
<h2>Prerequisites</h2> 
<ul> 
 <li>Access to an <a href="https://portal.aws.amazon.com/gp/aws/developer/registration/index.html" rel="noopener noreferrer" target="_blank">AWS account</a> through the AWS Management Console and the <a href="https://aws.amazon.com/cli" rel="noopener noreferrer" target="_blank">AWS Command Line Interface (AWS CLI)</a>. The <a href="https://aws.amazon.com/iam" rel="noopener noreferrer" target="_blank">AWS Identity and Access Management (IAM)</a> user that you use must have permissions to make the necessary AWS service calls and manage AWS resources mentioned in this post. While providing permissions to the IAM user, follow the <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html#grant-least-privilege" rel="noopener noreferrer" target="_blank">principle of least-privilege</a>.</li> 
 <li><a href="https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html" rel="noopener noreferrer" target="_blank">AWS CLI</a> installed and configured. If you are using long-term credentials like access keys, follow <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html" rel="noopener noreferrer" target="_blank">manage access keys for IAM users</a> and <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/securing_access-keys.html" rel="noopener noreferrer" target="_blank">secure access keys</a> for best practices.</li> 
 <li><a href="https://git-scm.com/book/en/v2/Getting-Started-Installing-Git" rel="noopener noreferrer" target="_blank">Git Installed</a>.</li> 
 <li><a href="https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html" rel="noopener noreferrer" target="_blank">AWS Serverless Application Model (AWS SAM)</a> installed.</li> 
 <li>Python 3.13 or later installed.</li> 
</ul> 
<h2>Set up and run the workflow</h2> 
<p>Run the following steps to deploy and test the Step Functions state machine.</p> 
<ol> 
 <li>Clone the GitHub repository in a new folder and navigate to the project folder. 
  <div class="hide-language"> 
   <pre><code class="lang-code">git clone https://github.com/aws-samples/sample-stepfunctions-s3-prefix-processor.git
cd sample-stepfunctions-s3-prefix-processor</code></pre> 
  </div> </li> 
 <li>Run the following commands to deploy the application. 
  <div class="hide-language"> 
   <pre><code class="lang-code">sam deploy --guided</code></pre> 
  </div> </li> 
 <li>Enter the following details: 
  <ul> 
   <li><code>Stack name</code>: Stack name for CloudFormation (for example, stepfunctions-s3-prefix-processor)</li> 
   <li><code>AWS Region</code>: A supported AWS Region (for example, us-east-1)</li> 
   <li>Accept all other default values.</li> 
  </ul> <p>The outputs from the AWS SAM deploy will be used in the subsequent steps.</p></li> 
 <li>Run the following command to generate sample log files. 
  <div class="hide-language"> 
   <pre><code class="lang-code">python3 scripts/generate_logs.py</code></pre> 
  </div> </li> 
 <li>Run the following to upload the log files to the S3 bucket with the <code>/logs/daily</code> prefix. Replace <code>amzn-s3-demo-bucket1</code> with the value from the <code>sam deploy</code> output. 
  <div class="hide-language"> 
   <pre><code class="lang-code">aws s3 sync logs/ s3://amzn-s3-demo-bucket1/logs/ --exclude '*' --include '*.log'</code></pre> 
  </div> </li> 
 <li>Run the following command to execute the Step Functions workflow. Replace the <code>StateMachineArn</code> with the value from the <code>sam deploy</code> output. 
  <div class="hide-language"> 
   <pre><code class="lang-code">aws stepfunctions start-execution \
  --state-machine-arn &lt;StateMachineArn&gt; \
  --input '{}'</code></pre> 
  </div> <p>The Step Function state machine iterates over all the log files with the S3 prefix <code>/logs/daily</code> and processes them in parallel. The workflow updates the metrics in CloudWatch, stores hourly metrics count in DynamoDB, then invokes an AWS Lambda function to aggregate the metrics.</p></li> 
</ol> 
<h2>Monitor and verify results</h2> 
<p>Run the following steps to monitor and verify the test results.</p> 
<ol> 
 <li>Run the following command to get the details of the execution. Replace <code>executionArn</code> with your state machine ARN. 
  <div class="hide-language"> 
   <pre><code class="lang-code">aws stepfunctions describe-execution --execution-arn &lt;executionArn&gt;</code></pre> 
  </div> </li> 
 <li>When the status shows <code>SUCCEEDED</code>, run the following commands to check the processed output from the <code>LogAnalyticsSummaryTableName</code> DynamoDB table. Replace the value <code>LogAnalyticsSummaryTableName</code> with the value from the <code>sam deploy</code> output. 
  <div class="hide-language"> 
   <pre><code class="lang-code">aws dynamodb scan --table-name &lt;LogAnalyticsSummaryTableName&gt;</code></pre> 
  </div> </li> 
 <li>Check that hourly <code>ERROR</code>, <code>WARN</code>, and <code>INFO</code> logs statistics are saved in the DynamoDB table. The following is a sample output: 
  <div class="hide-language"> 
   <pre><code class="lang-code">{
    "Items": [
        {
            "ProcessingTime": {
                "S": "2025-10-07T23:45:10.790Z"
            },
            "WarningCount": {
                "N": "2"
            },
            "HourOfDay": {
                "S": "13"
            },
            "TotalRecords": {
                "N": "5"
            },
            "ErrorCount": {
                "N": "3"
            },
            "InfoCount": {
                "N": "0"
            },
            "HourKey": {
                "S": "2025-10-08 13"
            }
        },
        {
            "ProcessingTime": {
                "S": "2025-10-07T23:45:07.456Z"
            },
            "WarningCount": {
                "N": "3"
            },
            "HourOfDay": {
                "S": "09"
            },
            "TotalRecords": {
                "N": "6"
            },
            "ErrorCount": {
                "N": "2"
            },
            "InfoCount": {
                "N": "1"
            },
            "HourKey": {
                "S": "2025-10-08 09"
            }
        },
        …
],
    "Count": 24,
    "ScannedCount": 24,
    "ConsumedCapacity": null
}</code></pre> 
  </div> </li> 
 <li>Run the following command to check the output of the Step Functions state machine execution output. 
  <div class="hide-language"> 
   <pre><code class="lang-code">aws stepfunctions describe-execution --execution-arn &lt;executionArn&gt; --query 'output' --output text</code></pre> 
  </div> <p>The following is a sample output:</p> 
  <div class="hide-language"> 
   <pre><code class="lang-code">{
  "Summary": {
    "date": "2025-10-08",
    "totalErrors": 50,
    "totalWarnings": 41,
    "totalRecords": 133,
    "hourlyBreakdown": {
      "00": {
        "errors": 1,
        "warnings": 3,
        "records": 5
      },
      "01": {
        "errors": 1,
        "warnings": 1,
        "records": 5
      },
      "02": {
        "errors": 2,
        "warnings": 3,
        "records": 5
      },
      "03": {
        "errors": 3,
        "warnings": 2,
        "records": 7
      },
…
…
    "generatedAt": "2025-10-08T05:19:05.603889"
  }
}</code></pre> 
  </div> <p>The output of the Step Functions state machine shows the daily summary insights of the log files created by the Lambda function.</p></li> 
</ol> 
<h2>Clean up</h2> 
<p>To avoid costs, remove all resources created for this post once you’re done. Run the following command after replacing amzn-s3-demo-bucket1 with your own bucket name to delete the resources you deployed for this post’s solution:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">aws s3 rm s3://amzn-s3-demo-bucket1 --recursive
sam delete
rm -rf logs/</code></pre> 
</div> 
<h2>Conclusion</h2> 
<p>In this post, you learned how AWS Step Functions Distributed Map can use prefix-based iteration with <code>LOAD_AND_FLATTEN</code> transformation to read and process multiple data objects from Amazon S3 buckets directly. You no longer need one step to process object metadata and another to load the data objects. Loading and flatting in one step is particularly valuable for data processing pipelines, batch operations, and event-driven architectures where objects are continually added to S3 locations. By eliminating the need to maintain object manifests, you can build more resilient, dynamic data processing workflows with less code and fewer moving parts.</p> 
<p>New input sources for Distributed Map are available in all commercial AWS Regions where AWS Step Functions is available. For a complete list of AWS Regions where Step Functions is available, see the <a href="https://aws.amazon.com/about-aws/global-infrastructure/regional-product-services/">AWS Region Table</a>. To get started, you can use the Distributed Map mode today in the <a href="https://console.aws.amazon.com/states/home/" rel="noopener noreferrer" target="_blank">AWS Step Functions console</a>. To learn more, visit the Step Functions <a href="https://docs.aws.amazon.com/step-functions/latest/dg/input-output-fields-dist-map.html" rel="noopener noreferrer" target="_blank">developer guide.</a></p> 
<p>For more serverless learning resources, visit&nbsp;<a href="https://serverlessland.com/" rel="noopener noreferrer" target="_blank">Serverless Land</a>.</p>
