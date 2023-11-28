# terraform-lambda-deploy-function
## main.tf
### terraform block
```
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.16"
    }
    # random = {
    #   source  = "hashicorp/random"
    #   version = "~> 3.1.0"
    # }
    # archive = {
    #   source  = "hashicorp/archive"
    #   version = "~> 2.2.0"
    # }
  }

  required_version = ">= 1.2.0"
}
```

### Provider block
```
provider "aws" {
  region = var.aws_region
}
```
The region will be defined inside variables.tf

### Create Random Bucket Name
```
resource "random_pet" "lambda_bucket_name" {
  prefix = "sg-terraform-demo"
  length = 4
}

resource "aws_s3_bucket" "lambda_bucket" {
  bucket        = random_pet.lambda_bucket_name.id
  force_destroy = true
}
```

### Define Bucket Ownership Control
```
resource "aws_s3_bucket_ownership_controls" "example" {
  bucket = aws_s3_bucket.lambda_bucket.id
  rule {
    object_ownership = "BucketOwnerPreferred"
  }
}
```

### Define Bucket Public Access Block
```
resource "aws_s3_bucket_public_access_block" "example" {
  bucket = aws_s3_bucket.lambda_bucket.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

### Define Bucket ACL
```
resource "aws_s3_bucket_acl" "example" {
  depends_on = [
    aws_s3_bucket_ownership_controls.example,
    aws_s3_bucket_public_access_block.example,
  ]

  bucket = aws_s3_bucket.lambda_bucket.id
  acl    = "public-read"
}
```

### Archive Source Code Lambda Function
```
data "archive_file" "lambda_sg_demo" {
  type = "zip"

  source_dir  = "${path.module}/sg-demo"
  output_path = "${path.module}/sg-demo.zip"
}
```

### Define Bucket Object
```
resource "aws_s3_object" "lambda_sg_demo" {
  bucket = aws_s3_bucket.lambda_bucket.id

  key    = "sg-demo.zip"
  source = data.archive_file.lambda_sg_demo.output_path

  etag = filemd5(data.archive_file.lambda_sg_demo.output_path)
}
```

### Define lambda Function
```
resource "aws_lambda_function" "sg_demo" {
  function_name = "SGDemo"

  s3_bucket = aws_s3_bucket.lambda_bucket.id
  s3_key    = aws_s3_object.lambda_sg_demo.key

  runtime = "nodejs18.x"
  handler = "demo.handler"

  source_code_hash = data.archive_file.lambda_sg_demo.output_base64sha256

  role = aws_iam_role.lambda_exec.arn
}
```

### Define Cloudwatch logGroup
```
resource "aws_cloudwatch_log_group" "sg_demo" {
  name = "/aws/lambda/${aws_lambda_function.sg_demo.function_name}"

  retention_in_days = 30
}
```

### Define AWS IAM Role Lambda Execution
```
resource "aws_iam_role" "lambda_exec" {
  name = "serverless_lambda"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Sid    = ""
      Principal = {
        Service = "lambda.amazonaws.com"
      }
      }
    ]
  })
}
```

### Define Role Policy
```
resource "aws_iam_role_policy_attachment" "lambda_policy" {
  role       = aws_iam_role.lambda_exec.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}
```

## variables.tf