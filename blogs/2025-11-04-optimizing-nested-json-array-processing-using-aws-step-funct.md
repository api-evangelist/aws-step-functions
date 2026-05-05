---
title: "Optimizing nested JSON array processing using AWS Step Functions Distributed Map"
url: "https://aws.amazon.com/blogs/compute/optimizing-nested-json-array-processing-using-aws-step-functions-distributed-map/"
date: "Tue, 04 Nov 2025 23:41:28 +0000"
author: "Biswanath Mukherjee"
feed_url: "https://aws.amazon.com/blogs/compute/category/application-services/aws-step-functions/feed/"
---
<p>When you’re working with large datasets, you’ve likely encountered the challenge of processing complex JSON structures in your automated workflows. You need to preprocess arrays within nested JSON objects before you can run parallel processing on them. Extracting data used to require custom code and extra processing steps, delaying you from building your core application logic.</p> 
<p>With <a href="https://docs.aws.amazon.com/step-functions/latest/dg/state-map-distributed.html" rel="noopener noreferrer" target="_blank">AWS Step Functions Distributed Map</a>, you can process large datasets with concurrent iterations of workflow steps across data entries. Using the enhanced ItemsPointer feature of Distributed Maps, you can extract array data directly from JSON objects stored in <a href="https://aws.amazon.com/s3/" rel="noopener noreferrer" target="_blank">Amazon S3</a>. Alternatively, for JSON object as state input, you can use Items (JSONata) or ItemsPath (JSONPath). With this enhancement you can point directly to arrays nested within JSON structures, eliminating the need for custom preprocessing of your data. With ItemsPointer, Items, and ItemsPath you can select the nested array data and simplify your workflows.</p> 
<p>In this post, we explore how to optimize processing array data embedded within complex JSON structures using AWS Step Functions Distributed Map. You’ll learn how to use ItemsPointer to reduce the complexity of your state machine definitions, create more flexible workflow designs, and streamline your data processing pipelines—all without writing additional transformation code or <a href="https://aws.amazon.com/lambda/" rel="noopener noreferrer" target="_blank">AWS Lambda</a> functions.</p> 
<table border="2px" cellpadding="10px" style="border-color: #FF9900;"> 
 <tbody> 
  <tr> 
   <td><strong>This post is part of a series of post about AWS Step Functions Distributed Map:</strong><p></p> <p></p> 
    <ul> 
     <li><a href="https://aws.amazon.com/blogs/compute/processing-amazon-s3-objects-at-scale-with-aws-step-functions-distributed-map-s3-prefix/" rel="noopener" target="_blank">Processing Amazon S3 objects at scale with AWS Step Functions Distributed Map S3 prefix</a></li> 
     <li><strong>Optimizing nested JSON array processing using AWS Step Functions Distributed Map</strong></li> 
     <li><a href="https://aws.amazon.com/blogs/compute/orchestrating-big-data-processing-with-aws-step-functions-distributed-map/" rel="noopener" target="_blank">Orchestrating big data processing with AWS Step Functions Distributed Map</a></li> 
    </ul> </td> 
  </tr> 
 </tbody> 
</table> 
<h2>Use case: e-commerce product data enrichment</h2> 
<p>In this e-commerce use case example, you’ll build a sample application that demonstrates processing of product inventory data for an e-commerce application using AWS Step Functions Distributed Map. The application receives a JSON file from an upstream application containing an array of product information. The Step Functions workflow reads the JSON file containing product data from an S3 bucket and iterates over the array to enrich each product data in the array.</p> 
<p>The following diagram presents the AWS Step Functions state machine.</p> 
<div class="wp-caption aligncenter" id="attachment_24981" style="width: 589px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/11/01/computeblog-2493-image-1.png"><img alt="JSON array processing workflow" class="size-full wp-image-24981" height="555" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/11/01/computeblog-2493-image-1.png" style="margin: 10px 0px 10px 0px;" width="579" /></a>
 <p class="wp-caption-text" id="caption-attachment-24981"><em>JSON array processing workflow</em></p>
</div> 
<p>The JSON array is processed using the following workflow:</p> 
<ol> 
 <li>The state machine reads the&nbsp;product-updates.json&nbsp;file from an input S3 bucket. The file contains a JSON array of products.</li> 
 <li>The Distributed Map state in the state machine, selects the JSON array node using <code>ItemsPointer</code> and iterates over the JSON array.</li> 
 <li>For each of the items within the array, the state machine invokes a Lambda function for data enrichment. The Lambda function adds product stock and price information to the product data.</li> 
 <li>The state machine saves the updated product data in an <a href="https://aws.amazon.com/dynamodb/" rel="noopener noreferrer" target="_blank">Amazon DynamoDB</a> table.</li> 
 <li>Finally, the state machine uploads the execution metadata into an output S3 bucket. See limits related to <a href="https://docs.aws.amazon.com/step-functions/latest/dg/service-quotas.html#service-limits-state-machine-executions" rel="noopener noreferrer" target="_blank">state machine executions</a> and <a href="https://docs.aws.amazon.com/step-functions/latest/dg/service-quotas.html#service-limits-task-executions" rel="noopener noreferrer" target="_blank">task executions</a>.</li> 
</ol> 
<p><a href="https://docs.aws.amazon.com/step-functions/latest/dg/state-map-distributed.html" rel="noopener noreferrer" target="_blank">MaxConcurrency</a> can be configured to specify the number of child workflow executions in a Distributed Map that can run in parallel. If not specified, then Step Functions doesn’t limit concurrency and runs 10,000 parallel child workflow executions.</p> 
<p>You can <a href="https://docs.aws.amazon.com/step-functions/latest/dg/input-output-itemreader.html#itemsource-example-json-data" rel="noopener noreferrer" target="_blank">read a JSON file from a S3 bucket</a> using <a href="https://docs.aws.amazon.com/step-functions/latest/dg/input-output-itemreader.html#itemreader-field-contents" rel="noopener noreferrer" target="_blank">ItemReader and its sub-fields</a>. If the JSON file, from the S3 bucket, contains a nested object structure, you can select the specific node with your data set with an&nbsp;<code>ItemsPointer</code>. For example, the following input JSON file:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">{
  "version": "2024.1",
  "timestamp": "2025-09-26T10:49:36.646197",
  "productUpdates": {
    "items": [
      {
        "productId": "PROD-001",
        "name": "Wireless Headphones",
        "price": 79.99,
        "stock": 150,
        "category": "Electronics"
      },
      {
        "productId": "PROD-002",
        "name": "Smart Watch",
        "category": "Electronics"
      },
      …
    ]
  }
}</code></pre> 
</div> 
<p>The following JSONata-based workflow configuration extracts a nested list of<em> products</em>&nbsp;from productUpdates/items:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">"ItemReader": {
   "Resource": "arn:aws:states:::s3:getObject",
   "ReaderConfig": {
      "InputType": "JSON",
      "ItemsPointer": "/productUpdates/items"
   },
   "Arguments": {
      "Bucket": "amzn-s3-demo-bucket",
      "Key": "updates/product-updates.json"
   }
}</code></pre> 
</div> 
<p>For JSONPath-based workflow note that <code>Arguments</code> is replaced with <code>Parameters</code>:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">"ItemReader": {
   "Resource": "arn:aws:states:::s3:getObject",
   "ReaderConfig": {
      "InputType": "JSON",
      "ItemsPointer": "/productUpdates/items"
   },
   "Arguments": {
      "Bucket": "amzn-s3-demo-bucket",
      "Key": "updates/product-updates.json"
   }
}</code></pre> 
</div> 
<p>The&nbsp;<code>ItemReader</code>&nbsp;field is not needed when your dataset is JSON data from a previous step. <code>ItemsPointer</code> is only applicable when the input JSON objects read from an S3 bucket. <a href="https://docs.aws.amazon.com/step-functions/latest/dg/input-output-itemreader.html#itemsource-json-array" rel="noopener noreferrer" target="_blank">If you are using JSON as state input to a Distributed Map</a>, then you can use the <a href="https://docs.aws.amazon.com/step-functions/latest/dg/input-output-itemspath.html" rel="noopener noreferrer" target="_blank"><code>ItemsPath</code> (for JSONPath)</a> or <a href="https://docs.aws.amazon.com/step-functions/latest/dg/transforming-data.html#writing-jsonata-expressions-in-json-strings" rel="noopener noreferrer" target="_blank"><code>Items</code> (for JSONata)</a> field to specify a location in the input that points to JSON array or object used for iterations.</p> 
<h2>Prerequisite</h2> 
<p>To use Step Functions Distributed Map, verify you have:</p> 
<ul> 
 <li>Access to an <a href="https://portal.aws.amazon.com/gp/aws/developer/registration/index.html" rel="noopener noreferrer" target="_blank">AWS account</a> through the AWS Management Console and the <a href="https://aws.amazon.com/cli" rel="noopener noreferrer" target="_blank">AWS Command Line Interface (AWS CLI)</a>. The <a href="https://aws.amazon.com/iam" rel="noopener noreferrer" target="_blank">AWS Identity and Access Management (IAM)</a> user that you use must have permissions to make the necessary AWS service calls and manage AWS resources mentioned in this post. While providing permissions to the IAM user, follow the <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html#grant-least-privilege" rel="noopener noreferrer" target="_blank">principle of least-privilege</a>.</li> 
 <li><a href="https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html" rel="noopener noreferrer" target="_blank">AWS CLI</a> installed and configured. If you are using long-term credentials like access keys, follow <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html" rel="noopener noreferrer" target="_blank">manage access keys for IAM users</a> and <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/securing_access-keys.html" rel="noopener noreferrer" target="_blank">secure access keys</a> for best practices.</li> 
 <li><a href="https://git-scm.com/book/en/v2/Getting-Started-Installing-Git" rel="noopener noreferrer" target="_blank">Git Installed</a></li> 
 <li><a href="https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html" rel="noopener noreferrer" target="_blank">AWS Serverless Application Model (AWS SAM)</a> installed</li> 
 <li>Python 3.13+ installed</li> 
</ul> 
<h2>Set up and run the workflow</h2> 
<p>Run the following steps to deploy the Step Functions state machine.</p> 
<ol> 
 <li>Clone the GitHub repository in a new folder and navigate to the project folder. 
  <div class="hide-language"> 
   <pre><code class="lang-code">git clone https://github.com/aws-samples/sample-stepfunctions-json-array-processor.git
cd sample-stepfunctions-json-array-processor</code></pre> 
  </div> </li> 
 <li>Run the following commands to deploy the application. 
  <div class="hide-language"> 
   <pre><code class="lang-code">sam deploy --guided</code></pre> 
  </div> <p> Enter the following details:</p> 
  <ul> 
   <li>Stack name: Stack name for CloudFormation (for example, stepfunctions-json-array-processor)</li> 
   <li>AWS Region: A supported AWS Region (for example, us-east-1)</li> 
   <li>Accept all other default values.</li> 
  </ul> <p>The outputs from the sam deploy will be used in the subsequent steps.</p></li> 
 <li>Run the following command to generate <code>product-updates.json</code> file containing a nested JSON array of sample products and upload the <code>product-updates.json</code>&nbsp;file to the input S3 bucket. Replace <code>InputBucketName</code> with the value from sam deploy output. 
  <div class="hide-language"> 
   <pre><code class="lang-code">python3 scripts/generate_sample_data.py &lt;InputBucketName&gt;</code></pre> 
  </div> </li> 
 <li>Run the following command to start execution of the Step Functions workflow. Replace the <code>StateMachineArn</code> with the value from <code>sam deploy</code> output. 
  <div class="hide-language"> 
   <pre><code class="lang-code">aws stepfunctions start-execution \
  --state-machine-arn &lt;StateMachineArn&gt; \
  --input '{}'</code></pre> 
  </div> <p>The state machine reads the input <code>product-updates.json</code>&nbsp;file and invokes a Lambda function to update the database for every product in the array after adding price and stock information. The execution metadata is also uploaded into the results bucket.</p></li> 
</ol> 
<h2>Monitor and verify results</h2> 
<p>Run the following steps to monitor and verify the test results.</p> 
<ol> 
 <li>Run the following command to get the details of the execution. Replace <code>executionArn</code> with your state machine ARN. 
  <div class="hide-language"> 
   <pre><code class="lang-code">aws stepfunctions describe-execution --execution-arn &lt;executionArn&gt;</code></pre> 
  </div> <p> Wait until the status shows <code>SUCCEEDED</code>.</p></li> 
 <li>Run the following commands to validate the processed output from <code>ProductCatalogTableName</code> DynamoDB table. Replace the value <code>ProductCatalogTableName</code> with the value from sam deploy output. 
  <div class="hide-language"> 
   <pre><code class="lang-code">aws dynamodb scan --table-name &lt;ProductCatalogTableName&gt;</code></pre> 
  </div> </li> 
 <li>Check that the DynamoDB table contains the enriched product data including price and stock attributes. Example output: 
  <div class="hide-language"> 
   <pre><code class="lang-code">{
    "Items": [
        {
            "ProductId": {
                "S": "PROD-005"
            },
            "lastUpdated": {
                "S": "2025-10-07T20:33:34.507Z"
            },
            "stock": {
                "N": "129"
            },
            "price": {
                "N": "139.25"
            }
        },
        {
            "ProductId": {
                "S": "PROD-003"
            },
            "lastUpdated": {
                "S": "2025-10-07T20:33:34.576Z"
            },
            "stock": {
                "N": "471"
            },
            "price": {
                "N": "40.92"
            }
        },
	      …
    ],
    "Count": 5,
    "ScannedCount": 5,
    "ConsumedCapacity": null
}</code></pre> 
  </div> </li> 
</ol> 
<h2>Clean up</h2> 
<p>To avoid costs, remove all resources you’ve created while following along with this post.</p> 
<p>Run the following command after replacing the <code>&lt;placeholder&gt;</code> variable to delete the resources you deployed for this post’s solution:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">aws s3 rm s3://&lt;InputBucketName&gt; --recursive
aws s3 rm s3://&lt;ResultBucketName&gt; --recursive
sam delete</code></pre> 
</div> 
<h2>Conclusion</h2> 
<p>In this post, you learned how to use Step Functions Distributed Map for extracting array data natively from JSON objects stored in a S3 bucket. By removing custom data extraction code, you can simplify the processing of your large-scale parallel workloads. With ItemsPointer you can extract array data within JSON files stored in a S3 bucket , and with Items(JSONata) or ItemsPath (JSONPath), you can extract arrays from complex JSON state input, adding flexibility to your workflow designs.</p> 
<p>New input sources for Distributed Map are available in all commercial AWS Regions where AWS Step Functions is available. For a complete list of AWS Regions where Step Functions is available, see the <a href="https://aws.amazon.com/about-aws/global-infrastructure/regional-product-services/">AWS Region Table</a>. To get started, you can use the Distributed Map mode today in the <a href="https://console.aws.amazon.com/states/home/" rel="noopener noreferrer" target="_blank">AWS Step Functions console</a>. To learn more, visit the <a href="https://docs.aws.amazon.com/step-functions/latest/dg/input-output-fields-dist-map.html" rel="noopener noreferrer" target="_blank">Step Functions developer guide</a>.</p> 
<p>For more serverless learning resources, visit&nbsp;<a href="https://serverlessland.com/" rel="noopener noreferrer" target="_blank">Serverless Land</a>.</p>
