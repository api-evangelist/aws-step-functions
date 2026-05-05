---
title: "Testing Step Functions workflows: a guide to the enhanced TestState API"
url: "https://aws.amazon.com/blogs/compute/testing-step-functions-workflows-a-guide-to-the-enhanced-teststate-api/"
date: "Sun, 22 Mar 2026 17:06:38 +0000"
author: "D Surya Sai"
feed_url: "https://aws.amazon.com/blogs/compute/category/application-services/aws-step-functions/feed/"
---
<p><a href="https://aws.amazon.com/step-functions/" rel="noopener noreferrer" target="_blank">AWS Step Functions</a> recently announced new enhancements to local testing capabilities for Step Functions, introducing API-based testing that developers can use to validate workflows before deploying to AWS. As detailed in our Announcement <a href="https://aws.amazon.com/blogs/aws/accelerate-workflow-development-with-enhanced-local-testing-in-aws-step-functions/" rel="noopener noreferrer" target="_blank">blog post</a>, the TestState API transforms Step Functions development by enabling individual state testing in isolation or as complete workflows. This supports mocked responses and actual AWS service integrations, and provides advanced capabilities. These capabilities include Map/Parallel states, error simulation with retry mechanisms, context object validation, and detailed inspection metadata for comprehensive local testing of your serverless application.</p> 
<p>The TestState API can be accessed through multiple interfaces such as <a href="https://aws.amazon.com/cli/" rel="noopener noreferrer" target="_blank">AWS Command Line Interface</a> (AWS CLI), <a href="https://aws.amazon.com/what-is/sdk/" rel="noopener noreferrer" target="_blank">AWS SDK</a>, <a href="https://www.localstack.cloud/" rel="noopener noreferrer" target="_blank">LocalStack</a>. By default, TestState API in AWS CLI and SDK runs against the remote <a href="https://docs.aws.amazon.com/general/latest/gr/step-functions.html#step-functions_region" rel="noopener noreferrer" target="_blank">AWS endpoint</a>, providing validation against the actual Step Functions service infrastructure. We’ve partnered with LocalStack to offer an additional testing endpoint for the TestState API. Developers can use LocalStack for unit testing their workflows by changing the <a href="https://aws.amazon.com/what-is/sdk/" rel="noopener noreferrer" target="_blank">AWS SDK</a> client endpoint configuration to point to LocalStack: <code><em>http://localhost.localstack.cloud:4566/</em></code> instead of <a href="https://docs.aws.amazon.com/general/latest/gr/step-functions.html#step-functions_region" rel="noopener noreferrer" target="_blank">AWS endpoint</a>. This approach provides complete network isolation when needed. For a streamlined development experience, you can also use the <a href="https://docs.localstack.cloud/aws/tooling/vscode-extension/" rel="noopener noreferrer" target="_blank">LocalStack VSCode extension</a> to automatically configure your environment to point to the LocalStack endpoint. This approach is detailed in the AWS <a href="https://aws.amazon.com/blogs/compute/enhance-the-local-testing-experience-for-serverless-applications-with-localstack/" rel="noopener noreferrer" target="_blank">blog post</a>.</p> 
<p>This blog post demonstrates building test suites to unit test your Step Functions workflows using the AWS SDK for Python using the <a href="https://docs.pytest.org/en/stable/" rel="noopener noreferrer" target="_blank">pytest framework</a>. The complete implementation is available in the <a href="https://github.com/aws-samples/sample-stepfunctions-testing-with-testStateAPI/" rel="noopener noreferrer" target="_blank">GitHub repository</a>.</p> 
<h2>Building test cases using the TestState API</h2> 
<p>This example workflow implements a real-world ecommerce order processing system using <a href="https://jsonata.org/" rel="noopener noreferrer" target="_blank">JSONata</a> for advanced data transformations. It incorporates complex Step Functions patterns including distributed Map states, Parallel execution, and waitForTaskToken callback mechanisms. The process validates orders through <a href="https://aws.amazon.com/lambda/" rel="noopener noreferrer" target="_blank">AWS Lambda</a> functions, distributes order item processing with configurable failure tolerance, runs parallel payment and inventory updates, handles human approval workflows using task tokens, then persists orders in Amazon DynamoDB with notification delivery. This workflow demonstrates advanced error handling with multiple Catchers and Retriers, exponential backoff for Lambda throttling and DynamoDB limits, and sophisticated state transitions that were previously challenging to test locally. This makes it the recommended choice for demonstrating the use of enhanced TestState API’s local testing features.</p> 
<p>The complete workflow is available in the <a href="https://github.com/aws-samples/sample-stepfunctions-testing-with-testStateAPI/" rel="noopener noreferrer" target="_blank">GitHub repository</a>, where you can examine the full state machine definition and see how JSONata expressions handle data transformation throughout the execution flow.</p> 
<div class="wp-caption alignnone" id="attachment_25870" style="width: 872px;">
 <img alt="Figure 1: State machine workflow that demonstrates a real-world ecommerce order processing system." class="size-full wp-image-25870" height="1292" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/18/compute-2435-img.png" width="862" />
 <p class="wp-caption-text" id="caption-attachment-25870">Figure 1: State machine workflow that demonstrates a real-world ecommerce order processing system.</p>
</div> 
<p>Effective Step Functions testing requires a systematic approach to TestState API integration that provides state validation, error simulation, and assertion capabilities. The testing framework is built using Python’s pytest framework, using <a href="https://docs.pytest.org/en/stable/explanation/fixtures.html" rel="noopener noreferrer" target="_blank">fixtures</a> to automatically provide pre-configured runner instances that handle TestState API client initialization and state machine definition loading. This eliminates repetitive setup code and provides consistent test environments. The enhanced TestState API supports both mock integrations and actual integrations with AWS services, providing flexibility in testing strategies. For this demonstration, you use mock integrations to showcase how a complete local testing can be achieved without having any resources deployed to AWS accounts.</p> 
<p>This framework is built for demonstration purposes, and you can similarly build your own testing frameworks using other programming languages like <a href="https://www.java.com/en/" rel="noopener noreferrer" target="_blank">Java</a>, <a href="https://nodejs.org/en" rel="noopener noreferrer" target="_blank">Node.js</a>. The testing framework uses method chaining patterns to create readable test cases with comprehensive assertion methods, automatic output chaining between state executions, and error simulation for testing retry mechanisms, backoff intervals, and catch blocks across AWS service error conditions.</p> 
<p>The following test implementations demonstrate the testing capabilities that are achievable with the enhanced TestState API in local development environments. The test cases are run against the preceding Statemachine.</p> 
<h3><strong>Test Case 1: Lambda throttling and retry mechanism testing</strong></h3> 
<p>Service integrations with Statemachines like AWS Lambda, Amazon <a href="https://aws.amazon.com/dynamodb/" rel="noopener noreferrer" target="_blank">DynamoDB</a> may face throttling depending on their usage. A key capability of the enhanced TestState API is its ability to simulate retry mechanisms with control over retry counts and backoff intervals. This test demonstrates the enhanced TestState API’s retry testing capabilities through the <code>stateConfiguration.retrierRetryCount</code>&nbsp;<a href="https://docs.aws.amazon.com/step-functions/latest/apireference/API_TestState.html#StepFunctions-TestState-request-stateConfiguration" rel="noopener noreferrer" target="_blank">parameter</a> and <code>inspectionData.errorDetails</code> <a href="https://docs.aws.amazon.com/step-functions/latest/apireference/API_InspectionErrorDetails.html" rel="noopener noreferrer" target="_blank">response fields</a>. This response field provides <code>retryBackoffIntervalSeconds</code> for validating exponential backoff calculations, <code>retryIndex</code> for tracking retry attempt sequences, and <code>catchIndex</code> for identifying which error handler processed the exception. These enhanced inspection capabilities enable validation of retry logic, <a href="https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/" rel="noopener noreferrer" target="_blank">backoff strategies</a>, and error propagation patterns across complex state machine workflows.</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">def test_lambda_throttling_retry_mechanism(self, runner):
"""Test retry mechanism for Lambda.TooManyRequestsException"""
throttling_error = {
"Error": "Lambda.TooManyRequestsException",
"Cause": "Request rate exceeded"
}

# Test first retry attempt
(runner
.with_input({"orderId": "order-retry-test"})
.with_mock_error(throttling_error)
.with_retrier_retry_count(0)
.execute("ValidateOrder")
.assert_retriable()
.assert_error("Lambda.TooManyRequestsException"))

# Verify exponential backoff calculation
response = runner.get_response()
error_details = response['inspectionData']['errorDetails']
assert error_details['retryBackoffIntervalSeconds'] == 2

# Test retry exhaustion
(runner
.with_retrier_retry_count(3)
.execute("ValidateOrder")
.assert_caught_error()
.assert_next_state("ValidationFailed"))</code></pre> 
</div> 
<h3><strong>Test Case 2: Map state testing with tolerance thresholds</strong></h3> 
<p><a href="https://docs.aws.amazon.com/step-functions/latest/dg/state-map.html" rel="noopener noreferrer" target="_blank">Distributed Map states</a> present unique testing challenges due to their parallel processing nature and failure tolerance capabilities. The enhanced TestState API provides specialized configuration options for testing these complex scenarios.</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">def test_map_state_tolerated_failure_threshold(self, runner):
"""Test Map state with tolerated failure threshold"""
test_input = {
"orderId": "order-map-test",
"orderItems": [
{"itemId": "item-1"}, {"itemId": "item-2"}, 
{"itemId": "item-3"}, {"itemId": "item-4"}
]
}

# Test normal Map state execution
map_success_result = [
{"itemId": "item-1", "processed": True},
{"itemId": "item-2", "processed": True}
]

(runner
.with_input(test_input)
.with_mock_result(map_success_result)
.execute("ProcessOrderItems")
.assert_succeeded()
.assert_next_state("ParallelProcessing"))

# Test tolerance threshold exceeded scenario
tolerance_error = {
"Error": "States.ExceedToleratedFailureThreshold",
"Cause": "Map state exceeded tolerated failure threshold"
}

(runner
.with_input(test_input)
.with_mock_error(tolerance_error)
.execute("ProcessOrderItems")
.assert_caught_error()
.assert_next_state("ValidationFailed"))</code></pre> 
</div> 
<p>This test demonstrates the enhanced TestState API’s Map state testing capabilities through the <code>stateConfiguration.mapIterationFailureCount</code> <a href="https://docs.aws.amazon.com/step-functions/latest/apireference/API_TestState.html#StepFunctions-TestState-request-stateConfiguration" rel="noopener noreferrer" target="_blank">parameter</a> for simulating iteration failures. The API provides comprehensive <a href="https://docs.aws.amazon.com/step-functions/latest/apireference/API_TestState.html#API_TestState_ResponseSyntax" rel="noopener noreferrer" target="_blank">inspection data</a> including <code>inspectionData.afterItemSelector</code> for validating <code>ItemSelector</code> transformations, <code>inspectionData.afterItemBatcher</code> for batch processing validation, <code>inspectionData.toleratedFailureCount</code> and <code>inspectionData.toleratedFailurePercentage</code> for threshold verification. When the specified failure count exceeds the configured tolerance, the API correctly returns <code>States.ExceedToleratedFailureThreshold</code>, enabling testing of Map state resilience patterns.</p> 
<h3><strong>Test Case 3: WaitForCallback pattern testing</strong></h3> 
<p>The <a href="https://docs.aws.amazon.com/step-functions/latest/dg/connect-to-resource.html#connect-wait-token" rel="noopener noreferrer" target="_blank">waitForCallback</a> integration requires context object construction to simulate realistic execution environments, particularly for human approval workflows.</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">def test_context_object_usage_in_jsonata_expressions(self, runner):
"""Test Context object usage in waitForTaskToken scenarios"""
test_input = {
"orderId": "order-context-test",
"amount": 125.0
}

context_data = {
"Task": {"Token": "ahbdgftgehbdcndsjnwjkhas327yr4hendc73yehdb723y"},
"Execution": {
"Id": "arn:aws:states:us-east-1:123456789012:execution:test:exec-123"
},
"State": {
"Name": "WaitForApproval",
"EnteredTime": "2025-01-15T10:45:00Z"
}
}

mock_result = {
"approved": True,
"taskToken": "ahbdgftgehbdcndsjnwjkhas327yr4hendc73yehdb723y"
}

(runner
.with_input(test_input)
.with_context(context_data)
.with_mock_result(mock_result)
.execute("WaitForApproval")
.assert_succeeded()
.assert_next_state("CheckApproval"))

# Verify JSONata expressions processed context correctly
response = runner.get_response()
after_args = json.loads(response['inspectionData']['afterArguments'])
assert after_args['Payload']['taskToken'] == context_data['Task']['Token']</code></pre> 
</div> 
<p>This test demonstrates the enhanced TestState API’s support for <code>waitForCallback</code> integrations through the `context` parameter for realistic Context object simulation. The API enables comprehensive testing of JSONata expressions that reference <code>$states.context.Task.Token</code>, <code>$states.context.Execution.Id</code>, and other context fields. The <code>inspectionData.afterArguments</code> <a href="https://docs.aws.amazon.com/step-functions/latest/apireference/API_TestState.html#API_TestState_ResponseSyntax" rel="noopener noreferrer" target="_blank">response field</a> validates that JSONata expressions correctly processed the context data, while the API automatically handles the complexity of task token embedding in service integration payloads for <code>waitForCallback</code> testing scenarios.</p> 
<h3><strong>Test Case 4: Happy path testing – complete workflow validation</strong></h3> 
<p>Happy path testing validates that workflows execute correctly under normal operating conditions. The enhanced TestState API allows you to chain state executions together, automatically passing outputs between states to simulate a complete workflow execution.</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">def test_complete_order_processing_workflow(self, runner):
"""Integration test: Complete happy path workflow using method chaining"""
test_input = {
"orderId": "order-12345",
"amount": 150.75,
"customerEmail": "customer@example.com",
"orderItems": [
{"itemId": "item-1", "quantity": 2, "price": 50.25}
]
}

# Test ValidateOrder state
(runner
.with_input(test_input)
.with_mock_result({"statusCode": 200, "isValid": True})
.execute("ValidateOrder")
.assert_succeeded()
.assert_next_state("CheckValidation"))

# Test CheckValidation choice state (no mock needed)
validation_output = runner.get_output()
(runner
.with_input(validation_output)
.clear_mocks()
.execute("CheckValidation")
.assert_succeeded()
.assert_next_state("ProcessOrderItems"))</code></pre> 
</div> 
<p>This test demonstrates how the TestState API maintains state context between executions, enabling realistic workflow simulation. The <code>get_output()</code> method retrieves the processed output from one state to use as input for the next, mimicking actual Step Functions execution behavior.</p> 
<p><em><strong>Note</strong>: The code snippet above shows only the first two states of the complete workflow test for brevity. The full test code with all states (<code>ProcessOrderItems</code>, <code>ParallelProcessing</code>, <code>WaitForApproval</code>, <code>CheckApproval</code>, <code>SaveOrderDetails</code>, and <code>SendNotification</code>) can be viewed in the complete </em><a href="https://github.com/aws-samples/sample-stepfunctions-testing-with-testStateAPI/" rel="noopener noreferrer" target="_blank"><em>GitHub repository</em></a><em>, demonstrating end-to-end workflow validation using the same method chaining pattern.</em></p> 
<h2><strong>Integration with modern CI/CD pipelines</strong></h2> 
<p>In this section, we will explore how to integrate the previous unit tests in a CI CD pipeline to enable local testing.</p> 
<p>The sample <a href="https://github.com/aws-samples/sample-stepfunctions-testing-with-testStateAPI/" rel="noopener noreferrer" target="_blank">repository</a> includes a GitHub Actions workflow that demonstrates how TestState API testing integrates into continuous integration and continuous delivery (CI/CD) pipelines. The workflow (<code>.github/workflows/test-and-deploy.yml</code>) provides a two-step process that validates before any AWS resources are deployed using <a href="https://aws.amazon.com/serverless/sam/" rel="noopener noreferrer" target="_blank">AWS Serverless Application Model</a> (AWS SAM).</p> 
<p>The CI/CD pipeline follows the following pattern:</p> 
<ol> 
 <li><strong>Unit Tests</strong>: Executes the complete TestState API test suite using <code>pytest tests/unit_test.py -v</code></li> 
 <li><strong>SAM Deploy</strong>: Deploys AWS resources using <a href="https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-cli-command-reference-sam-build.html" rel="noopener noreferrer" target="_blank">sam build</a> and <a href="https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-cli-command-reference-sam-deploy.html" rel="noopener noreferrer" target="_blank">sam deploy</a></li> 
</ol> 
<p>To enable the GitHub Actions workflow to deploy resources to your AWS account, configure these AWS credentials in your GitHub repository settings. For detailed setup instructions, see the AWS <a href="https://aws.amazon.com/blogs/compute/using-github-actions-to-deploy-serverless-applications/" rel="noopener noreferrer" target="_blank">blog post</a>.</p> 
<p>Following are the required secrets to be configured in GitHub repository settings:</p> 
<ul> 
 <li><code>AWS_ACCESS_KEY_ID</code></li> 
 <li><code>AWS_SECRET_ACCESS_KEY</code></li> 
 <li><code>AWS_REGION</code></li> 
</ul> 
<p>In production environments, you can typically extend this basic pipeline to include additional stages. The enhanced pipeline often begins with deploying to a development account first, followed by integration testing against deployed resources. The final stage involves moving to production with proper approval gates and security scanning compliance checks.</p> 
<h2>Conclusion</h2> 
<p>The enhanced TestState API enables testing Step Functions workflows locally without requiring AWS deployments that accelerated development cycles, and reduce testing times. This post demonstrates how to implement testing for state types including Map states with tolerance thresholds, retry mechanisms with exponential backoff, and <code>waitForTaskToken</code> patterns with context object simulation using mock integrations for isolated testing.</p> 
<p>By integrating TestState API testing into CI/CD pipelines, you can validate workflow logic before deployment, reducing the risk of production issues. The GitHub Actions workflow example demonstrates an implementation that runs tests and deploys resources in a controlled sequence. The complete code examples and testing framework are available in the <a href="https://github.com/aws-samples/sample-stepfunctions-testing-with-testStateAPI/" rel="noopener noreferrer" target="_blank">GitHub repository</a> to implement similar testing practices for Step Functions workflows.</p> 
<hr style="width: 80%;" />
