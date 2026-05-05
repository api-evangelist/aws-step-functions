---
title: "Orchestrating big data processing with AWS Step Functions Distributed Map"
url: "https://aws.amazon.com/blogs/compute/orchestrating-big-data-processing-with-aws-step-functions-distributed-map/"
date: "Tue, 04 Nov 2025 23:42:01 +0000"
author: "Biswanath Mukherjee"
feed_url: "https://aws.amazon.com/blogs/compute/category/application-services/aws-step-functions/feed/"
---
<p>Developers seek to process and enrich semi-structured big data datasets with durably orchestrated network-based workflows. For example, during quarterly earnings season, finance organizations run thousands of market simulations simultaneously to provide timely insights for scenario planning or risk management—these workloads require coordination between raw datasets and on-premise servers to provide the latest market information.</p> 
<p><a href="https://aws.amazon.com/step-functions/" rel="noopener noreferrer" target="_blank">AWS Step Functions</a> is a visual workflow service capable of orchestrating over 14,000 API actions from over 220 AWS services to build distributed applications. Now, Step Functions <a href="https://docs.aws.amazon.com/step-functions/latest/dg/state-map-distributed.html" rel="noopener noreferrer" target="_blank">Distributed Map</a> streamlines big data dataset transformation by processing <a href="https://aws.amazon.com/athena/" rel="noopener noreferrer" target="_blank">Amazon Athena</a> data manifest and Parquet files directly. Using its Distributed Map feature, you can process large scale datasets by running concurrent iterations across data entries in parallel. In Distributed mode, the Map state processes the items in the dataset in iterations called <em>child workflow executions</em>. You can specify the number of child workflow executions that can run in parallel. Each child workflow execution has its own, separate execution history from that of the parent workflow. By default, Step Functions runs 10,000 parallel child workflow executions in parallel.</p> 
<p>Distributed Map can process AWS Athena data manifest and Parquet files directly, eliminating the need for custom pre-processing. You also now have visibility into your Distributed Map usage with new <a href="https://docs.aws.amazon.com/step-functions/latest/dg/procedure-cw-metrics.html#resource-count-metrics-map-run" rel="noopener noreferrer" target="_blank">Amazon CloudWatch metrics</a>: Approximate Open Map Runs Count, Open Map Run Limit, and Approximate Map Runs Backlog Size.</p> 
<p>In this post, you’ll learn how to use AWS Step Functions Distributed Map to process Athena data manifest and Parquet files through a step-by-step demonstration.</p> 
<table border="2px" cellpadding="10px" style="border-color: #FF9900;"> 
 <tbody> 
  <tr> 
   <td><strong>This post is part of a series of post about AWS Step Functions Distributed Map:</strong><p></p> <p></p> 
    <ul> 
     <li><a href="https://aws.amazon.com/blogs/compute/processing-amazon-s3-objects-at-scale-with-aws-step-functions-distributed-map-s3-prefix/" rel="noopener" target="_blank">Processing Amazon S3 objects at scale with AWS Step Functions Distributed Map S3 prefix</a></li> 
     <li><a href="https://aws.amazon.com/blogs/compute/optimizing-nested-json-array-processing-using-aws-step-functions-distributed-map/" rel="noopener" target="_blank">Optimizing nested JSON array processing using AWS Step Functions Distributed Map</a></li> 
     <li><strong>Orchestrating big data processing with AWS Step Functions Distributed Map</strong></li> 
    </ul> </td> 
  </tr> 
 </tbody> 
</table> 
<h2>Use case: IoT sensor data processing</h2> 
<p>You’ll build a sample application that demonstrates processing IoT sensor data in Parquet format using Step Functions Distributed Map. These Parquet data files and a manifest file containing the list of the data files are exported from Athena. The data temperature, humidity, and lbattery level from different devices. The following table shows sample of sensor data:</p> 
<div class="wp-caption aligncenter" id="attachment_24926" style="width: 1042px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/10/29/computeblog-2460-image-1.png"><img alt="Example IoT sensor data" class="size-full wp-image-24926" height="486" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/10/29/computeblog-2460-image-1.png" style="margin: 10px 0px 10px 0px;" width="1032" /></a>
 <p class="wp-caption-text" id="caption-attachment-24926"><em>Example IoT sensor data</em></p>
</div> 
<p>Your objective is to use the Athena data manifest file, get the list of Parquet files, and iterate over the data in the files to detect anomalies and also stream the processed data through <a href="https://aws.amazon.com/firehose/" rel="noopener noreferrer" target="_blank">Amazon Kinesis Data Firehose</a> to an Amazon S3 bucket for further analytics using Athena queries. Following is the criteria to detect anomaly:</p> 
<ul> 
 <li>Low battery conditions: less than 20%</li> 
 <li>Humidity anomalies: more than 95% or less than 5%</li> 
 <li>Temperature spikes: more than 35°C or less than -10°C</li> 
</ul> 
<p>The following diagram represents the AWS Step Functions state machine:</p> 
<div class="wp-caption aligncenter" id="attachment_24927" style="width: 1320px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/10/29/computeblog-2460-image-2.png"><img alt="Parquet files processing workflow" class="size-full wp-image-24927" height="1212" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/10/29/computeblog-2460-image-2.png" style="margin: 10px 0px 10px 0px;" width="1310" /></a>
 <p class="wp-caption-text" id="caption-attachment-24927"><em>Parquet files processing workflow</em></p>
</div> 
<ol> 
 <li>The Distributed Map runs an Athena query which generates Parquet data files and an Athena manifest file (csv). The manifest file contains the list of Parquet data files.</li> 
 <li>Distributed Map processes these Parquet data files in parallel using child workflow executions. You can control the number of child workflow executions that can run in parallel using MaxConcurrency parameter. See <a href="https://docs.aws.amazon.com/step-functions/latest/dg/service-quotas.html" rel="noopener noreferrer" target="_blank">Step Functions service quotas</a> to learn more about concurrency limits.</li> 
 <li>Each child workflow execution invokes an <a href="https://aws.amazon.com/lambda/" rel="noopener noreferrer" target="_blank">AWS Lambda</a> function to process the respective Parquet file. The Lambda function processes individual sensor readings and detects anomalies according to the preceeding logic and returns a processed sensor data summary response.</li> 
 <li>The child workflow sends the summary response record to Amazon Kinesis firehose stream which stores the results in a specified Amazon S3 results bucket.</li> 
</ol> 
<p>The following Athena Start QueryExecution state runs an <a href="https://docs.aws.amazon.com/athena/latest/ug/unload.html" rel="noopener noreferrer" target="_blank">UNLOAD</a> query to generate data files in Parquet format and a manifest file in CSV. The output will be stored in the S3 bucket specified in the UNLOAD query and the manifest file will be stored in the S3 bucket configured for the Athena workgroup.</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">{
  "QueryLanguage": "JSONata",
  "States": {
	   "Athena StartQueryExecution": {
	    "Type": "Task",
	        "Resource": "arn:aws:states:::athena:startQueryExecution.sync",
	        "Arguments": {
		"QueryString": "UNLOAD (WRITE_YOUR_SELECT_QUERY_HERE) TO 'S3_URI_FOR_STORING_DATA_OBJECT' WITH (format = 'JSON')",
		"WorkGroup": "primary"
	},
	"Output": {
	"ManifestObjectKey": "{% $join([$states.result.QueryExecution.ResultConfiguration.OutputLocation, '-manifest.csv']) %}"
},
“Next”: “Next State”
…
}</code></pre> 
</div> 
<p>The following <a href="https://docs.aws.amazon.com/step-functions/latest/dg/input-output-itemreader.html" rel="noopener noreferrer" target="_blank">ItemReader&nbsp;</a>is configured to use a manifest type of “ATHENA_DATA” with “PARQUET” data input.</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">{
  "QueryLanguage": "JSONata",
  "States": {
    ...
    "Map": {
        ...
        "ItemReader": {
        	"Resource": "arn:aws:states:::s3:getObject",
   	"ReaderConfig": {
      		"ManifestType": "ATHENA_DATA",
      		"InputType": "PARQUET"
   	},
   	"Arguments": {
      		"Bucket":"Bucket": "{% $split($substringAfter($states.input.ManifestObjectKey, 's3://'), '/')[0] %}",,
      		"Key": "{% $substringAfter($substringAfter($states.input.ManifestObjectKey, 's3://'), '/') %}"
   	}
	    },
        ...
    }
}</code></pre> 
</div> 
<p>Additional supported InputType options are CSV and JSONL. All objects referenced in a single manifest file must have the same InputType format. You specify the Amazon S3 bucket location of Athena manifest CSV file under Arguments.</p> 
<p>The <a href="https://docs.aws.amazon.com/step-functions/latest/dg/input-output-contextobject.html#contextobject-map" rel="noopener noreferrer" target="_blank"><em>context object</em></a> contains information in a JSON structure about your state machine and execution. Your workflows can reference the context object in a JSONata expression with&nbsp;$states.context.</p> 
<p>Within a&nbsp;<a href="https://docs.aws.amazon.com/step-functions/latest/dg/state-map.html" rel="noopener noreferrer" target="_blank">Map state</a>, the Context object includes the following data:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">"Map": {
   "Item": {
      "Index" : Number,
      "Key"   : "String", // Only valid for JSON objects
      "Value" : "String",
      "Source": "String"
   }
}</code></pre> 
</div> 
<p>For each&nbsp;Map&nbsp;state iteration,&nbsp;Index contains the index number for the array item that is being currently processed,&nbsp;Key is available only when iterating over JSON objects, Value contains the array item being processed, and&nbsp;Source&nbsp;contains one of the following:</p> 
<ul> 
 <li>For state input, the value will be :&nbsp;STATE_DATA</li> 
 <li>For&nbsp;Amazon S3 LIST_OBJECTS_V2&nbsp;with&nbsp;Transformation=NONE, the value will show the S3 URI for the bucket. For example:&nbsp;S3://amzn-s3-demo-bucket.</li> 
 <li>For all the other input types, the value will be the Amazon S3 URI. For example:&nbsp;S3://amzn-s3-demo-bucket/object-key.</li> 
</ul> 
<p>Using this newly introduced Source field in the context object, you can connect the child executions with the source object.</p> 
<h2>Prerequisites</h2> 
<ul> 
 <li>Access to an <a href="https://portal.aws.amazon.com/gp/aws/developer/registration/index.html" rel="noopener noreferrer" target="_blank">AWS account</a> through the AWS Management Console and the <a href="https://aws.amazon.com/cli" rel="noopener noreferrer" target="_blank">AWS Command Line Interface (AWS CLI)</a>. The <a href="https://aws.amazon.com/iam" rel="noopener noreferrer" target="_blank">AWS Identity and Access Management (IAM)</a> user that you use must have permissions to make the necessary AWS service calls and manage AWS resources mentioned in this post. While providing permissions to the IAM user, follow the <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html#grant-least-privilege" rel="noopener noreferrer" target="_blank">principle of least-privilege</a>.</li> 
 <li><a href="https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html" rel="noopener noreferrer" target="_blank">AWS CLI</a> installed and configured. If you are using long-term credentials like access keys, follow <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html" rel="noopener noreferrer" target="_blank">manage access keys for IAM users</a> and <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/securing_access-keys.html" rel="noopener noreferrer" target="_blank">secure access keys</a> for best practices.</li> 
 <li><a href="https://git-scm.com/book/en/v2/Getting-Started-Installing-Git" rel="noopener noreferrer" target="_blank">Git Installed</a></li> 
 <li><a href="https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html" rel="noopener noreferrer" target="_blank">AWS Serverless Application Model (AWS SAM)</a> installed</li> 
 <li>Python 3.13+ installed</li> 
</ul> 
<h2>Set up the state machine and sample data</h2> 
<p>Run the following steps to deploy the Step Functions state machine.</p> 
<ol> 
 <li>Clone the GitHub repository in a new folder and navigate to the project root folder. 
  <div class="hide-language"> 
   <pre><code class="lang-code">git clone https://github.com/aws-samples/sample-stepfunctions-athena-manifest-parquet-file-processor.git
cd sample-stepfunctions-athena-manifest-parquet-file-processor</code></pre> 
  </div> </li> 
 <li>Run the following command to install required Python dependencies for the Lambda function. 
  <div class="hide-language"> 
   <pre><code class="lang-code">python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install -r requirements.txt</code></pre> 
  </div> </li> 
 <li>Build the application. 
  <div class="hide-language"> 
   <pre><code class="lang-code">sam build</code></pre> 
  </div> </li> 
 <li>Deploy the application 
  <div class="hide-language"> 
   <pre><code class="lang-code">sam deploy --guided</code></pre> 
  </div> </li> 
 <li>Enter the following details: 
  <ul> 
   <li>Stack name: The CloudFormation stack name (for example, sfn-parquet-file-processor)</li> 
   <li>AWS Region: A supported AWS Region (for example, us-east-1)</li> 
   <li>Keep rest of the components to default values.</li> 
  </ul> <p>Note the outputs from the AWS SAM deploy. You will use them in the subsequent steps.</p></li> 
 <li>Run the following command to generate sample data in csv format and upload it to an S3 bucket. Replace <code>&lt;IoTDataBucketName&gt;</code> with the value from <code>sam deploy</code> ouptut. 
  <div class="hide-language"> 
   <pre><code class="lang-code">python3 scripts/generate_sample_data.py &lt;IoTDataBucketName&gt;</code></pre> 
  </div> </li> 
</ol> 
<h2>Create the Athena database and tables</h2> 
<p>Before you can run queries, you must set up an Athena database and table for your data.</p> 
<ol> 
 <li>From <a href="https://console.aws.amazon.com/athena/home">Amazon Athena console</a>, navigate to workgoups, select the workgroup named “primary”. Select Edit from Actions. In the query result configuration section, select the options as follows: 
  <ol type="a"> 
   <li>Management of query results – select customer managed</li> 
   <li>Location of query results – enter s3://&lt;IoTDataBucketName&gt;. Replace <code>&lt;IoTDataBucketName&gt;</code> with the value from <code>sam deploy</code> output.</li> 
   <li>Choose <strong>Save</strong> to save the changes to the workgroup</li> 
  </ol> </li> 
 <li>Select Query editor tab and run the following commands to create database and tables 
  <div class="hide-language"> 
   <pre><code class="lang-sql">CREATE DATABASE `iotsensordata`;</code></pre> 
  </div> </li> 
 <li>Create an Athena table in database iotsensordata that references the S3 bucket containing the raw sensor data. In this case it will be <code>&lt;IoTDataBucketName&gt;</code>. Replace <code>&lt;IoTDataBucketName&gt;</code> with the value from <code>sam deploy</code> output. 
  <div class="hide-language"> 
   <pre><code class="lang-sql">CREATE EXTERNAL TABLE IF NOT EXISTS `iotsensordata`.`iotsensordata` 
(`deviceid` string, 
`timestamp` string,
`temperature` double,
`humidity` double,
`batterylevel` double,
`latitude` double,
`longitude` double
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'
WITH SERDEPROPERTIES ('field.delim' = ',')
STORED AS INPUTFORMAT 'org.apache.hadoop.mapred.TextInputFormat' OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION 's3://&lt;IoTDataBucketName&gt;/daily-data/'
TBLPROPERTIES (
 'classification' = 'csv',
 'skip.header.line.count' = '1'
);</code></pre> 
  </div> </li> 
 <li>Create an Athena table in database iotsensordata that references the S3 bucket having the analytics results streamed from Kinesis Data Firehose. Replace <code>&lt;IoTAnalyticsResultsBucket&gt;</code> with value from <code>sam deploy</code> output. And replace <code>&lt;year&gt;</code> with the current year (e.g 2025). 
  <div class="hide-language"> 
   <pre><code class="lang-sql">CREATE EXTERNAL TABLE IF NOT EXISTS iotsensordata.iotsensordataanalytics (deviceid string, analysisDate string, readingTimestamp string, readingsCount int, metrics struct&lt; temperature: double, humidity: double, batterylevel: double, latitude: double, longitude: double &gt;, anomalies array &lt;string&gt;, anomalyCount int, healthStatus string, timestamp string )
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
WITH SERDEPROPERTIES ( 'ignore.malformed.json' = 'FALSE', 'dots.in.keys' = 'FALSE', 'case.insensitive' = 'TRUE'
)
STORED AS INPUTFORMAT 'org.apache.hadoop.mapred.TextInputFormat' OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION 's3://&lt;IoTAnalyticsResultsBucket&gt;/&lt;year&gt;/'
TBLPROPERTIES ('classification' = 'json', 'typeOfData'='file');</code></pre> 
  </div> </li> 
</ol> 
<h2>Start your state machine</h2> 
<p>Now that you have data ready and Athena set up for queries, start your state machine to retrieve and process the data.</p> 
<ol> 
 <li>Run the following command to start execution of the Step Functions. Replace the <code>&lt;StateMachineArn&gt;</code> and <code>&lt;IoTDataBucketName&gt;</code> with the value from sam deploy output.. 
  <div class="hide-language"> 
   <pre><code class="lang-code">aws stepfunctions start-execution \
  --state-machine-arn &lt;StateMachineArn&gt; \
  --input '{ "IoTDataBucketName": "&lt;IoTDataBucketName&gt;"}'</code></pre> 
  </div> <p>The Step Functions state machine has the Athena <code>StartQueryExecution</code> state which has an <code>UNLOAD</code> query that generates the sensor data files in a parquet format and a manifest file in CSV format. The manifest will have 5 rows referencing the 5 parquet files. The state machine will process these 5 parquet files in one map run.</p></li> 
 <li>Run the following command to get the details of the execution. Replace the <code>executionArn</code> from the previous command. 
  <div class="hide-language"> 
   <pre><code class="lang-code">aws stepfunctions describe-execution --execution-arn &lt;executionArn&gt;</code></pre> 
  </div> </li> 
 <li>After you see the status <code>SUCCEEDED</code>, run the following command from Athena query editor to check the processed output from Kinesis Data Firehose that was streamed to S3 bucket referenced by the Athena table created in step 4 of the preceding section. 
  <div class="hide-language"> 
   <pre><code class="lang-sql">SELECT * FROM iotsensordata.iotsensordataanalytics WHERE anomalycount = 1;</code></pre> 
  </div> </li> 
</ol> 
<p>If any of the sensor data exceeds the thresholds, the healthstatus attribute will be set to “anomalies_detected”. The workflow produced a summary table of metadata which you can now query for reporting.</p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/10/29/computeblog-2460-image-3.png"><img alt="Output from Athena Query Editor" class="size-full wp-image-24928" height="540" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/10/29/computeblog-2460-image-3.png" style="margin: 10px 0px 10px 0px;" width="2164" /></a></p> 
<h2>Review workflow performance</h2> 
<p>Using the following observability metrics, you can review key performance behavior of your data processing workflow.<br /> The&nbsp;AWS/States&nbsp;namespace includes the following new metrics for all Step Functions Map Runs.</p> 
<ul> 
 <li><code>OpenMapRunLimit</code>: This is the maximum number of open Map Runs allowed in the AWS account. The default value is 1,000 runs and is a hard limit. For more information, see&nbsp;<a href="https://docs.aws.amazon.com/step-functions/latest/dg/service-quotas.html#service-limits-accounts" rel="noopener noreferrer" target="_blank">Quotas related to accounts</a>.</li> 
 <li><code>ApproximateOpenMapRunCount</code>: This metric tracks the approximate number of Map Runs currently in progress within an account. Configuring an alarm on this metric using the Maximum statistic with a threshold of 900 or higher can help you take proactive action before reaching the <code>OpenMapRunLimit</code> of 1,000. This metric enables operational teams to implement preventive measures, such as staggering new executions or optimizing workflow concurrency, to maintain system stability and prevent backlog accumulation.</li> 
 <li><code>ApproximateMapRunBacklogSize</code>: This metric shows up when the <code>ApproximateOpenMapRunCount</code> has reached 1,000 and there are backlogged Map Runs waiting to be executed. Backlogged Map Runs wait at the&nbsp;<a href="https://docs.aws.amazon.com/step-functions/latest/apireference/API_MapRunStartedEventDetails.html" rel="noopener noreferrer" target="_blank">MapRunStarted&nbsp;</a>event until the total number of open Map Runs is less than the quota.</li> 
</ul> 
<p>The following graph shows an example of these new metrics. Use the maximum statistic to visualize these metrics. <code>ApproximateMapRunBacklogSize</code> metrics appear after accounts start getting throttled on the <code>OpenMapRunLimit</code> limit. The <code>OpenMapRun</code> (orange line) is the account hard limit of 1,000 shown as a static line. The <code>ApproximateOpenMapRunCount</code> (violet line) is the current number of active <code>OpenMap</code> runs. The <code>ApproximateMapRunBacklogSize</code> (green line) indicates the map runs waiting in backlog to be processed. When the <code>ApproximateOpenMapRunCount</code> is lower than 1000 (<code>OpenMapRun</code> limit) there are no map runs in backlog. However, when the count reaches the <code>OpenMapRun</code> limit, the backlog of map runs starts to build up. After the active runs complete, the backlog will start to drain out and new runs will begin execution.</p> 
<div class="wp-caption aligncenter" id="attachment_24929" style="width: 2316px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/10/29/computeblog-2460-image-4.png"><img alt="Graphed metrics from Amazon CloudWatch" class="size-full wp-image-24929" height="614" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/10/29/computeblog-2460-image-4.png" style="margin: 10px 0px 10px 0px;" width="2306" /></a>
 <p class="wp-caption-text" id="caption-attachment-24929"><em>Graphed metrics from Amazon CloudWatch</em></p>
</div> 
<h2>Clean up</h2> 
<p>To avoid costs, remove all resources created for this post once you’re done. From the Athena query editor, run the following commands:</p> 
<div class="hide-language"> 
 <pre><code class="lang-sql">DROP TABLE `iotsensordata`.`iotsensordata`;
DROP TABLE `iotsensordata`.`iotsensordataanalytics`;
DROP DATABASE `iotsensordata`;
</code></pre> 
</div> 
<p>Run the following commands from the AWS CLI after replacing the &lt;placeholder&gt; variable to delete the resources you deployed for this post’s solution:</p> 
<div class="hide-language"> 
 <pre><code class="lang-sql">aws s3 rm s3://&lt;IoTDataBucketName&gt; --recursive
aws s3 rm s3://&lt;IoTAnalyticsResultsBucketName&gt; --recursive
sam delete</code></pre> 
</div> 
<h2>Conclusion</h2> 
<p>With this update, Distributed Map now supports additional data inputs, so you can orchestrate large-scale analytics and ETL workflows. You can now process Amazon Athena data manifest and Parquet files directly, eliminating the need for custom pre-processing. You also now have visibility into your Distributed Map usage with the following metrics: Approximate Open Map Runs Count, Open Map Run Limit, and Approximate Map Runs Backlog Size.</p> 
<p>New input sources for Distributed Map are available in all commercial AWS Regions where AWS Step Functions is available. For a complete list of AWS Regions where Step Functions is available, see the <a href="https://aws.amazon.com/about-aws/global-infrastructure/regional-product-services/">AWS Region Table</a>. The improved observability of your Distributed Map usage with new metrics is available in all AWS Regions. To get started, you can use the Distributed Map mode today in the <a href="https://console.aws.amazon.com/states/home/" rel="noopener noreferrer" target="_blank">AWS Step Functions console</a>. To learn more, visit the Step Functions <a href="https://docs.aws.amazon.com/step-functions/latest/dg/input-output-fields-dist-map.html" rel="noopener noreferrer" target="_blank">developer guide.</a></p> 
<p>For more serverless learning resources, visit&nbsp;<a href="https://serverlessland.com/" rel="noopener noreferrer" target="_blank">Serverless Land</a>.</p>
