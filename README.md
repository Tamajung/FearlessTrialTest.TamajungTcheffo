
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.16"
    }
  }

  required_version = ">= 1.2.0"
}

provider "aws" {
  region  = "us-west-2"
}
resource "aws_apigatewayv2_api" "main" {
  name          = "main"
  protocol_type = "HTTP"
}

resource "aws_apigatewayv2_stage" "dev" {
  api_id = aws_apigatewayv2_api.main.id

  name        = "dev"
  auto_deploy = true
}
resource "aws_apigatewayv2_integration" "lambda_hello" {
  api_id = aws_apigatewayv2_api.main.id

  integration_uri    = aws_lambda_function.hello.invoke_arn
  integration_type   = "AWS_PROXY"
  integration_method = "POST"
}

resource "aws_apigatewayv2_route" "get_hello" {
  api_id = aws_apigatewayv2_api.main.id

  route_key = "GET /hello"
  target    = "integrations/${aws_apigatewayv2_integration.lambda_hello.id}"

}
resource "aws_lambda_permission" "api_gw" {
  statement_id  = "AllowExecutionFromAPIGateway"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.hello.function_name
  principal     = "apigateway.amazonaws.com"

  source_arn = "${aws_apigatewayv2_api.main.execution_arn}/*/*"
}

output "hello_base_url" {
  value = aws_apigatewayv2_stage.dev.invoke_url
}
resource "aws_iam_role" "SSLExpiryRole" {
  name = "SSLExpiryRole"
  assume_role_policy = "${file("SSLExpiryRole.json")}"
}

resource "aws_iam_policy_attachment" "AWSLambdaBasicExecutionRole-attachment" {
  name = "SSLExpiryRolePolicy"
  roles = [
    "${aws_iam_role.SSLExpiryRole.name}"
  ]
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_lambda_function" "SSLExpiry" {
  description = ""
  function_name = "SSLExpiry"
  handler = "ssl_expiry_lambda.main"
  runtime = "python3.6"
  filename = "ssl-expiry-check.zip"
  timeout = 120
  source_code_hash = "${base64sha256(file("ssl-expiry-check.zip"))}"
  role = "${aws_iam_role.SSLExpiryRole.arn}"

  environment {
    variables = {
      HOSTLIST = "${var.HOSTLIST}"
      EXPIRY_BUFFER = "${var.EXPIRY_BUFFER}"
    }
  }
}

resource "aws_cloudwatch_event_rule" "SSLExpirySchedule" {
  name = "SSLExpirySchedule"
  depends_on = [
    "aws_lambda_function.SSLExpiry"
  ]
  schedule_expression = "rate(1 day)"
}

resource "aws_cloudwatch_event_target" "SSLExpiry" {
  target_id = "SSLExpiry"
  rule = "${aws_cloudwatch_event_rule.SSLExpirySchedule.name}"
  arn = "${aws_lambda_function.SSLExpiry.arn}"
}

resource "aws_lambda_permission" "SSLExpirySchedule" {
  statement_id = "AllowExecutionFromCloudWatch"
  action = "lambda:InvokeFunction"
  function_name = "${aws_lambda_function.SSLExpiry.function_name}"
  principal = "events.amazonaws.com"
  source_arn = "${aws_cloudwatch_event_rule.SSLExpirySchedule.arn}"
}
