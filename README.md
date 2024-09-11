# go-lambda-apigateway-otel

## Introduction

This example is based on the following sample code:

https://github.com/aws-samples/sessions-with-aws-sam/tree/master/go-al2 

When instrumenting a Go Lambda function that's fronted by AWS API Gateway with OpenTelemetry, 
trace context is not automatically propagated.  So if the client of the Lambda function 
is instrumented with OpenTelemetry, and the traceparent header is added to the HTTP request 
by the OpenTelemetry SDK, this will *not* be read automatically by the
[OpenTelemetry AWS Lambda Instrumentation for Golang implementation](https://github.com/open-telemetry/opentelemetry-go-contrib/tree/instrumentation/github.com/aws/aws-lambda-go/otellambda/example/v0.53.0/instrumentation/github.com/aws/aws-lambda-go/otellambda).

To get trace context propagation working successfully in this scenario, we need to tell 
the otellambda instrumentation how to extract the traceparent header from the
APIGatewayProxyRequest event object that gets passed to the handler.  This is done
with the following code: 

````
var traceparent = http.CanonicalHeaderKey("traceparent")

func customEventToCarrier(eventJSON []byte) propagation.TextMapCarrier {
	var request events.APIGatewayProxyRequest
	_ = json.Unmarshal(eventJSON, &request)

	var header = http.Header{
		traceparent: []string{request.Headers["traceparent"]},
	}

	return propagation.HeaderCarrier(header)
}
````

Then we need to specify this function when we instrument the handler
with otellambda: 

````
	lambda.Start(otellambda.InstrumentHandler(handler,
		otellambda.WithEventToCarrier(customEventToCarrier)))
````

Refer to the [documentation](https://github.com/open-telemetry/opentelemetry-go-contrib/tree/main/instrumentation/github.com/aws/aws-lambda-go/otellambda) 
for further explanation on WithEventToCarrier. 

## Prerequisites

* Go 1.18+
* Docker
* AWS SAM CLI

## Steps to run the example

### Clone the repo

````
git clone https://github.com/dmitchsplunk/go-lambda-apigateway-otel.git
cd go-lambda-apigateway-otel/hello-world
````

### Init and Install Go modules

````
go mod init hello-world
go get github.com/aws/aws-lambda-go/events
go get github.com/signalfx/splunk-otel-go/distro
go get -u go.opentelemetry.io/contrib/instrumentation/github.com/aws/aws-lambda-go/otellambda
````

### Authenticate with AWS

````
export AWS_ACCESS_KEY_ID="..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_SESSION_TOKEN="...”
````

### Build and Deploy

````
cd ..
sam build
sam deploy --guided
````

### Add splunk-go-otel instrumentation

Use the AWS console to find your new lambda function, and 
add the splunk-otel-go instrumentation as described [here](https://docs.splunk.com/observability/en/gdi/get-data-in/serverless/aws/otel-lambda-layer/instrument-lambda-functions.html).


### Test

To test the function and ensure trace context propagation is working, 
build a simple upstream client in Java, Python, etc. 
that simply invokes this Lambda function via the API gateway URl that was 
provided as part of the sam deploy command.  Instrument the client with 
OpenTelemetry, then run in a few times to invoke the Lambda function. 
