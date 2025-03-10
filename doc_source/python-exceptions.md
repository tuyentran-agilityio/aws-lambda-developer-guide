# AWS Lambda function errors in Python<a name="python-exceptions"></a>

**Note**  
End of support for the Python 2\.7 runtime started on July 15, 2021\. For more information, see [Runtime support policy](runtime-support-policy.md)\.

When your code raises an error, Lambda generates a JSON representation of the error\. This error document appears in the invocation log and, for synchronous invocations, in the output\.

This page describes how to view Lambda function invocation errors for the Python runtime using the Lambda console and the AWS CLI\.

**Topics**
+ [How it works](#python-exceptions-how)
+ [Using the Lambda console](#python-exceptions-console)
+ [Using the AWS Command Line Interface \(AWS CLI\)](#python-exceptions-cli)
+ [Error handling in other AWS services](#python-exceptions-other-services)
+ [Error examples](#python-exceptions-examples)
+ [Sample applications](#python-exceptions-samples)
+ [What's next?](#python-exceptions-next-up)

## How it works<a name="python-exceptions-how"></a>

When you invoke a Lambda function, Lambda receives the invocation request and validates the permissions in your execution role, verifies that the event document is a valid JSON document, and checks parameter values\.

If the request passes validation, Lambda sends the request to a function instance\. The [Lambda runtime](lambda-runtimes.md) environment converts the event document into an object, and passes it to your function handler\. 

If Lambda encounters an error, it returns an exception type, message, and HTTP status code that indicates the cause of the error\. The client or service that invoked the Lambda function can handle the error programmatically, or pass it along to an end user\. The correct error handling behavior depends on the type of application, the audience, and the source of the error\.

The following list describes the range of status codes you can receive from Lambda\.

**`2xx`**  
A `2xx` series error with a `X-Amz-Function-Error` header in the response indicates a Lambda runtime or function error\. A `2xx` series status code indicates that Lambda accepted the request, but instead of an error code, Lambda indicates the error by including the `X-Amz-Function-Error` header in the response\.

**`4xx`**  
A `4xx` series error indicates an error that the invoking client or service can fix by modifying the request, requesting permission, or by retrying the request\. `4xx` series errors other than `429` generally indicate an error with the request\. 

**`5xx`**  
A `5xx` series error indicates an issue with Lambda, or an issue with the function's configuration or resources\. `5xx` series errors can indicate a temporary condition that can be resolved without any action by the user\. These issues can't be addressed by the invoking client or service, but a Lambda function's owner may be able to fix the issue\.

For a complete list of invocation errors, see [InvokeFunction errors](API_Invoke.md#API_Invoke_Errors)\.

## Using the Lambda console<a name="python-exceptions-console"></a>

You can invoke your function on the Lambda console by configuring a test event and viewing the output\. The output is captured in the function's execution logs and, when [active tracing](services-xray.md) is enabled, in AWS X\-Ray\.

**To invoke a function on the Lambda console**

1. Open the [Functions page](https://console.aws.amazon.com/lambda/home#/functions) of the Lambda console\.

1. Choose the function to test, and choose **Test**\.

1. Under **Test event**, select **New event**\.

1. Select a **Template**\.

1. For **Name**, enter a name for the test\. In the text entry box, enter the JSON test event\.

1. Choose **Save changes**\.

1. Choose **Test**\.

The Lambda console invokes your function [synchronously](invocation-sync.md) and displays the result\. To see the response, logs, and other information, expand the **Details** section\.

## Using the AWS Command Line Interface \(AWS CLI\)<a name="python-exceptions-cli"></a>

The AWS CLI is an open\-source tool that enables you to interact with AWS services using commands in your command line shell\. To complete the steps in this section, you must have the following:
+ [AWS CLI – Install version 2](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
+ [AWS CLI – Quick configuration with `aws configure`](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html)

When you invoke a Lambda function in the AWS CLI, the AWS CLI splits the response into two documents\. The AWS CLI response is displayed in your command prompt\. If an error has occurred, the response contains a `FunctionError` field\. The invocation response or error returned by the function is written to an output file\. For example, `output.json` or `output.txt`\.

The following [invoke](https://docs.aws.amazon.com/cli/latest/reference/lambda/invoke.html) command example demonstrates how to invoke a function and write the invocation response to an `output.txt` file\.

```
aws lambda invoke   \
  --function-name my-function   \
      --cli-binary-format raw-in-base64-out  \
          --payload '{"key1": "value1", "key2": "value2", "key3": "value3"}' output.txt
```

The cli\-binary\-format option is required if you are using AWS CLI version 2\. You can also configure this option in your [AWS CLI config file](https://docs.aws.amazon.com/cli/latest/userguide/cliv2-migration.html#cliv2-migration-binaryparam)\.

You should see the AWS CLI response in your command prompt:

```
{
    "StatusCode": 200,
    "FunctionError": "Unhandled",
    "ExecutedVersion": "$LATEST"
}
```

You should see the function invocation response in the `output.txt` file\. In the same command prompt, you can also view the output in your command prompt using:

```
cat output.txt
```

You should see the invocation response in your command prompt\.

```
{"errorMessage": "'action'", "errorType": "KeyError", "stackTrace": ["  File \"/var/task/lambda_function.py\", line 36, in lambda_handler\n    result = ACTIONS[event['action']](event['number'])\n"]}
```

Lambda also records up to 256 KB of the error object in the function's logs\. For more information, see [AWS Lambda function logging in Python](python-logging.md)\.

## Error handling in other AWS services<a name="python-exceptions-other-services"></a>

When another AWS service invokes your function, the service chooses the invocation type and retry behavior\. AWS services can invoke your function on a schedule, in response to a lifecycle event on a resource, or to serve a request from a user\. Some services invoke functions asynchronously and let Lambda handle errors, while others retry or pass errors back to the user\.

For example, API Gateway treats all invocation and function errors as internal errors\. If the Lambda API rejects the invocation request, API Gateway returns a `500` error code\. If the function runs but returns an error, or returns a response in the wrong format, API Gateway returns a 502 error code\. To customize the error response, you must catch errors in your code and format a response in the required format\.

We recommend using AWS X\-Ray to determine the source of an error and its cause\. X\-Ray allows you to find out which component encountered an error, and see details about the errors\. The following example shows a function error that resulted in a `502` response from API Gateway\.

![\[\]](http://docs.aws.amazon.com/lambda/latest/dg/images/tracemap-apig-502.png)

For more information, see [Instrumenting Python code in AWS Lambda](python-tracing.md)\.

## Error examples<a name="python-exceptions-examples"></a>

The following section shows common errors you may receive when creating, updating, or invoking your function using the Python [Lambda runtimes](lambda-runtimes.md)\.

**Example Runtime exception – ImportError**  

```
{
  "errorMessage": "Unable to import module 'lambda_function': Cannot import name '_imaging' from 'PIL' (/var/task/PIL/__init__.py)",
  "errorType": "Runtime.ImportModuleError"
}
```

This error is a result of using the AWS Command Line Interface \(AWS CLI\) to upload a deployment package that contains a C or C\+\+ library\. For example, the [Pillow \(PIL\)](https://pypi.org/project/Pillow/), [numpy](https://pypi.org/project/numpy/), or [pandas](https://pypi.org/project/pandas/) library\.

We recommend using the AWS SAM CLI [sam build](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-using-build.html) command with the `--use-container` option to create your deployment package\. Using the AWS SAM CLI with this option creates a Docker container with a Lambda\-like environment that is compatible with Lambda\.

**Example JSON serialization error – Runtime\.MarshalError**  

```
{
  "errorMessage": "Unable to marshal response: Object of type AttributeError is not JSON serializable",
  "errorType": "Runtime.MarshalError"
}
```

This error can be the result of the base64\-encoding mechanism you are using in your function code\. For example:

```
import base64
encrypted_data = base64.b64encode(payload_enc).decode("utf-8")
```

This error can also be the result of not specifying your \.zip file as a binary file when you created or updated your function\. We recommend using the [fileb://](https://docs.aws.amazon.com/cli/latest/userguide/cli-usage-parameters-file.html#cli-usage-parameters-file-binary) command option to upload your deployment package \(\.zip file\)\.

```
aws lambda create-function --function-name my-function --zip-file fileb://my-deployment-package.zip --handler lambda_function.lambda_handler --runtime python3.8 --role arn:aws:iam::your-account-id:role/lambda-ex
```

## Sample applications<a name="python-exceptions-samples"></a>

The GitHub repository for this guide includes sample applications that demonstrate the use of the errors\. Each sample application includes scripts for easy deployment and cleanup, an AWS Serverless Application Model \(AWS SAM\) template, and supporting resources\.

**Sample Lambda applications in Python**
+ [blank\-python](https://github.com/awsdocs/aws-lambda-developer-guide/tree/main/sample-apps/blank-python) – A Python function that shows the use of logging, environment variables, AWS X\-Ray tracing, layers, unit tests and the AWS SDK\.

## What's next?<a name="python-exceptions-next-up"></a>
+ Learn how to show logging events for your Lambda function on the [AWS Lambda function logging in Python](python-logging.md) page\.