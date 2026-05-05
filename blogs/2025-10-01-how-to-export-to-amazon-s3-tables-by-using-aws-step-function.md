---
title: "How to export to Amazon S3 Tables by using AWS Step Functions Distributed Map"
url: "https://aws.amazon.com/blogs/compute/how-to-export-to-amazon-s3-tables-by-using-aws-step-functions-distributed-map/"
date: "Wed, 01 Oct 2025 16:44:18 +0000"
author: "Chetan Makvana"
feed_url: "https://aws.amazon.com/blogs/compute/category/application-services/aws-step-functions/feed/"
---
<p>Companies running serverless workloads often need to perform extract, transform, and load (ETL) operations on data files stored in <a href="https://aws.amazon.com/s3/" rel="noopener noreferrer" target="_blank">Amazon Simple Storage Service (Amazon S3)</a> buckets. Though traditional approaches such as an <a href="https://docs.aws.amazon.com/lambda/latest/dg/with-s3.html" rel="noopener noreferrer" target="_blank">AWS Lambda trigger for Amazon S3</a> or <a href="https://docs.aws.amazon.com/AmazonS3/latest/userguide/EventNotifications.html" rel="noopener noreferrer" target="_blank">Amazon S3 Event Notifications</a> can handle these operations, they might fall short when workflows require enhanced visibility, control, or human intervention. For example, some processes might need manual review of failed records or explicit approval before proceeding to subsequent stages. Customer orchestration solutions to these issues can prove to be complex and error prone.</p> 
<p><a href="https://aws.amazon.com/step-functions/" rel="noopener noreferrer" target="_blank">AWS Step Functions</a> address these challenges by providing built-in workflow management and monitoring capabilities. The <a href="https://docs.aws.amazon.com/step-functions/latest/dg/state-map-distributed.html" rel="noopener noreferrer" target="_blank">Step Functions Distributed Map</a> feature is designed for high-throughput, parallel data processing workflows so that companies can handle complex ETL jobs, <a href="https://aws.amazon.com/blogs/aws/new-step-functions-support-for-dynamic-parallelism/" rel="noopener noreferrer" target="_blank">fan-out processing</a>, and data visualization at scale. Distributed Map handles each dataset item as an independent child workflow, processing millions of records while maintaining built-in concurrency controls, fault tolerance, and progress tracking. The processed data can be seamlessly exported to various destinations, including <a href="https://aws.amazon.com/s3/features/tables/" rel="noopener noreferrer" target="_blank">Amazon S3 Tables</a> with Apache Iceberg support.</p> 
<p>In this post, we show how to use Step Functions Distributed Map to process Amazon S3 objects and export results to Amazon S3 Tables, creating a scalable and maintainable data processing pipeline.</p> 
<p>See the associated <a href="https://github.com/aws-samples/sample-exporting-to-amazon-s3-tables-with-aws-step-functions-distributed-map" rel="noopener noreferrer" target="_blank">GitHub repository</a> for detailed instructions about deploying this solution as well as sample code.</p> 
<h1>Solution overview</h1> 
<p>Consider a consumer electronics company that regularly participates in industry trade shows and conferences. During these events, interested attendees fill out paper sign-up forms to request product demos, receive newsletters, or join early access programs. After the events, the company’s team scans hundreds of thousands of these forms and uploads them to Amazon S3.Rather than manually reviewing each form, the company wants to automate the extraction of key customer details such as name, email address, mailing address, and interest areas. They’d like to store this structured data in S3 Tables with Apache Iceberg format for downstream analytics and marketing campaign targeting.</p> 
<p>Let’s look at how this post’s solution uses Distributed Map to process PDFs in parallel, extract data using <a href="https://aws.amazon.com/textract/" rel="noopener noreferrer" target="_blank">Amazon Textract</a>, and write the cleaned output directly to S3 Tables. The result is scalable, serverless post-event data onboarding, as shown in the following figure.</p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/09/26/computeblog-2367-arch-diagram.png"><img alt="Solution architecture for automated PDF processing workflow with S3 Tables, EventBridge scheduling, Step Functions Distributed Map" class="alignnone wp-image-24640 size-full" height="331" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/09/26/computeblog-2367-arch-diagram.png" style="margin: 10px 0px 10px 0px;" width="821" /></a></p> 
<p>The data processing workflow as shown in the preceding diagram includes the following steps:</p> 
<ol> 
 <li>A user uploads customer interest forms as scanned PDFs to an Amazon S3 bucket.</li> 
 <li>An <a href="https://docs.aws.amazon.com/eventbridge/latest/userguide/using-eventbridge-scheduler.html" rel="noopener noreferrer" target="_blank">Amazon EventBridge Scheduler</a> rule triggers at regular intervals, initiating a Step Functions workflow execution.</li> 
 <li>The workflow execution activates a Step Functions Distributed Map state, which lists all PDF files uploaded to Amazon S3 since the previous run.</li> 
 <li>The Distributed Map iterates over the list of objects and passes each object’s metadata (<a href="https://docs.aws.amazon.com/AmazonS3/latest/API/API_Object.html" rel="noopener noreferrer" target="_blank">bucket, key, size, entity tag [ETag]</a>) to a child workflow execution.</li> 
 <li>For each object, the child workflow calls Amazon Textract with the provided bucket and key to extract raw text and relevant fields (name, email address, mailing address, interest area) from the PDF.</li> 
 <li>The child workflow sends the extracted data to <a href="https://aws.amazon.com/firehose/" rel="noopener noreferrer" target="_blank">Amazon Data Firehose</a>, which is configured to forward data to S3 Tables.</li> 
 <li>Firehose batches the incoming data from the child workflow and writes it to S3 Tables at a preconfigured time interval of your choosing.</li> 
</ol> 
<p>With data now structured and accessible in S3 Tables, users can easily analyze them using standard SQL queries with <a href="https://aws.amazon.com/athena/" rel="noopener noreferrer" target="_blank">Amazon Athena</a> or business intelligence like <a href="https://aws.amazon.com/quicksight/" rel="noopener noreferrer" target="_blank">Amazon QuickSight.</a></p> 
<h2>The data-processing workflow</h2> 
<p>EventBridge Scheduler starts new Step Functions workflows at regular intervals. The timeline for this schedule is flexible. However, When setting up your schedule, make sure the frequency aligns with how far back your state machine is configured to look for PDFs. For example, if your state machine checks for PDFs from the past week, you’d want to schedule it to run weekly. The Step Functions workflow subsequently performs the following three steps (note that these steps are steps 4, 5, 6, and 7 in the preceding workflow diagram:</p> 
<ol> 
 <li>Extract relevant user data from the PDFs.</li> 
 <li>Send the extracted user data to Firehose.</li> 
 <li>Write the data to S3 Tables in Apache Iceberg table format.</li> 
</ol> 
<p>The following diagram illustrates this workflow.</p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/09/26/computeblog-2367-step-function-workflow.png"><img alt="Screenshot of AWS Step Function workflow execution showing processing pipeline from S3 ingenstion through Kinesis batch output" class="alignnone wp-image-24639 size-full" height="963" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/09/26/computeblog-2367-step-function-workflow.png" style="margin: 10px 0px 10px 0px;" width="422" /></a></p> 
<p>Let’s look at each step of the preceding workflow in more detail.</p> 
<h3>Extract relevant user data from PDF documents</h3> 
<p>Step Functions uses Distributed Map to process PDFs concurrently in parallel child workflows. It accepts input from <a href="https://aws.amazon.com/documentdb/what-is-json/" rel="noopener noreferrer" target="_blank">JSON</a>, <a href="https://jsonlines.org/" rel="noopener noreferrer" target="_blank">JSONL</a>, CSV, Parquet files, Amazon S3 manifest files stored in Amazon S3 (used to specify particular files for processing), or an <a href="https://docs.aws.amazon.com/firehose/latest/dev/dynamic-partitioning-s3bucketprefix.html" rel="noopener noreferrer" target="_blank">Amazon S3 bucket prefix</a> (allows iteration over file metadata for all objects under that prefix). The Step Functions automatically handles parallelization by splitting the dataset and running child workflows for each item, with the <a href="https://docs.aws.amazon.com/step-functions/latest/dg/input-output-itembatcher.html" rel="noopener noreferrer" target="_blank">ItemBatcher</a> field allowing to group multiple PDFs into a single child workflow execution (e.g., 10 PDFs per batch) to optimize performance and cost.</p> 
<p>The following screenshot of the Step Functions console shows the configuration for Distributed Map. For example, we have configured Distributed Map to process 10 customer interest PDFs in a single child workflow.</p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/09/26/computeblog-2367-distributed-map-config.png"><img alt="A screenshot of AWS Step Functions console showing Distributed Map state configuration " class="alignnone wp-image-24638 size-full" height="716" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/09/26/computeblog-2367-distributed-map-config.png" style="margin: 10px 0px 10px 0px;" width="830" /></a></p> 
<p>The following image shows one example of these scanned PDFs, which includes the customer information that this post’s solution processes.</p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/09/26/computeblog-2367-sample-pdf.png"><img alt="A screenshot showing sample PDF" class="alignnone wp-image-24637 size-full" height="1262" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/09/26/computeblog-2367-sample-pdf.png" style="margin: 10px 0px 10px 0px;" width="1426" /></a></p> 
<p>Each child workflow then calls the <a href="https://docs.aws.amazon.com/textract/latest/dg/API_AnalyzeDocument.html" rel="noopener noreferrer" target="_blank">Amazon Textract AnalyzeDocument API</a> with specific queries to extract customer information.</p> 
<div class="hide-language"> 
 <pre><code class="lang-json">{
  "Document": {
    "S3Object": {
      "Bucket": "&lt;input PDFs bucket&gt;",
      "Name": "{% $states.input.Key %}"
    }
  },
  "FeatureTypes": [
    "QUERIES"
  ],
  "QueriesConfig": {
    "Queries": [
      {
        "Alias": "full_name",
        "Text": "What is the customer's name?"
      },
      {
        "Alias": "phone_number",
        "Text": "What is the customer’s phone number?"
      },
      {
        "Alias": "mailing_address",
        "Text": "What is the customer’s mailing address?"
      },
      {
        "Alias": "interest",
        "Text": "What is the customer’s interest?"
      }
    ]
  }
}</code></pre> 
</div> 
<p>The API analyzes each scanned PDF and returns a JSON structure containing the extracted customer information.</p> 
<h3>Send the extracted user data to Firehose</h3> 
<p>The child workflow then uses a <a href="https://docs.aws.amazon.com/firehose/latest/APIReference/API_PutRecordBatch.html" rel="noopener noreferrer" target="_blank">Firehose PutRecordBatch API action</a> with <a href="https://docs.aws.amazon.com/step-functions/latest/dg/integrate-services.html" rel="noopener noreferrer" target="_blank">service integrations</a> to queue the extracted customer information for further processing. The <code>PutRecordBatch</code> action request includes the Firehose stream name and the data records. The data records include a data blob from step 1 that contains extracted customer information, as shown in the following example.</p> 
<div class="hide-language"> 
 <pre><code class="lang-json">{
  "DeliveryStreamName": "put_raw_form_data_100",
  "Records": [
    {
      "Data": "{\"full_name\":\"Anthony Ayala\",\"phone_number\":\"001-384-925-0701\",\"mailing_address\":\"38548 Joshua Wall Suite 974, East Heatherfort, OH 32669\",\"interest\":\"Fitness Trackers\",\"processed_date\":\"2025-05-01\"}"
    },
    {
      "Data": "{\"full_name\":\"Becky Williams\",\"phone_number\":\"+1-283-499-2466\",\"mailing_address\":\"227 King Forge Suite 241, East Nathanland, PR 05687\",\"interest\":\"Al Assistants\",\"processed_date\":\"2025-05-01\"}"
    }
  ]
}</code></pre> 
</div> 
<h3>Write the data to S3 Tables in Apache Iceberg table format</h3> 
<p>Firehose efficiently manages data buffering, format conversion, and reliable delivery to various destinations, including Apache Iceberg, raw files in Amazon S3, <a href="https://aws.amazon.com/opensearch-service/" rel="noopener noreferrer" target="_blank">Amazon OpenSearch Service</a>, or <a href="https://docs.aws.amazon.com/firehose/latest/dev/create-destination.html" rel="noopener noreferrer" target="_blank">any of the other supported destinations</a>. Apache Iceberg tables can be either self-managed in Amazon S3 or hosted in S3 Tables. Though self-managed Iceberg tables require manual optimization—such as compaction and snapshot expiration—S3 Tables automatically optimize storage for large-scale analytics workloads, improving query performance and reducing storage costs.</p> 
<p>Firehose simplifies the process of streaming data by configuring a delivery stream, selecting a data source, and setting an Iceberg table as the destination. After you’ve set it up, the Firehose stream is ready to deliver data. The delivered data can be queried from S3 Tables by using Athena, as shown in the following screenshot of the Athena console.</p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/09/26/computeblog-2367-athena-console-query.png"><img alt="A screenshot of the Athena console showing a query to select the data we just uploaded" class="alignnone wp-image-24636 size-full" height="1046" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/09/26/computeblog-2367-athena-console-query.png" style="margin: 10px 0px 10px 0px;" width="3732" /></a></p> 
<p>The query results include all processed customer data from the PDFs, as shown in the following screenshot.</p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/09/26/computeblog-2367-athena-console-query-result.png"><img alt="A screenshot of the Athena console showing the results of the query we just ran" class="alignnone wp-image-24635 size-full" height="1326" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2025/09/26/computeblog-2367-athena-console-query-result.png" style="margin: 10px 0px 10px 0px;" width="3008" /></a></p> 
<p>This integration demonstrates a powerful, code-free solution for transforming raw PDF forms into enriched, queryable data in an Iceberg table. You can use these data for further analysis.</p> 
<h1>Conclusion</h1> 
<p>In this post, we showed how to build a scalable, serverless solution for processing PDF documents and exporting the extracted data to S3 Tables by using Step Functions Distributed Map. This architecture offers several key benefits such as reliability, cost-effectiveness, visibility, and maintainability. By leveraging AWS services such as Step Functions, Amazon Textract, Firehose, and S3 Tables, companies can automate their document processing workflows while ensuring optimal performance and operational excellence. This solution can be adapted for various use cases beyond customer interest forms, such as invoice processing, application forms, or any scenario requiring structured data extraction from documents at scale.</p> 
<p>Though this example focuses on processing PDF data and writing to S3 Tables, Distributed Map can handle various input sources including JSON, JSONL, CSV, and Parquet files in Amazon S3; items in <a href="https://aws.amazon.com/dynamodb/" rel="noopener noreferrer" target="_blank">Amazon DynamoDB</a> tables; Athena query results; and all paginated AWS List APIs. Similarly, through <a href="https://docs.aws.amazon.com/step-functions/latest/dg/integrate-services.html" rel="noopener noreferrer" target="_blank">Step Functions service integrations</a>, you can write results to multiple destinations such as <a href="https://docs.aws.amazon.com/step-functions/latest/dg/connect-ddb.html" rel="noopener noreferrer" target="_blank">DynamoDB tables by using the PutItem service integration</a>.</p> 
<p>To get started with this solution, see the associated <a href="https://github.com/aws-samples/sample-exporting-to-amazon-s3-tables-with-aws-step-functions-distributed-map" rel="noopener noreferrer" target="_blank">GitHub repository</a> for deployment instructions and sample code.</p>
