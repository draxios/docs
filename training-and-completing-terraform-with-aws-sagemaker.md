### SageMaker Terraform

Have to start with stuff before code, so here's starting terraform for AWS SageMaker.

### Terraform Configuration

```hcl
provider "aws" {
  region = "us-east-1"
}

# VPC
resource "aws_vpc" "sagemaker_vpc" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "sagemaker-vpc"
  }
}

resource "aws_subnet" "private_subnet_a" {
  vpc_id     = aws_vpc.sagemaker_vpc.id
  cidr_block = "10.0.1.0/24"
  availability_zone = "us-east-1a"

  tags = {
    Name = "sagemaker-private-subnet-a"
  }
}

resource "aws_subnet" "private_subnet_b" {
  vpc_id     = aws_vpc.sagemaker_vpc.id
  cidr_block = "10.0.2.0/24"
  availability_zone = "us-east-1b"

  tags = {
    Name = "sagemaker-private-subnet-b"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.sagemaker_vpc.id

  tags = {
    Name = "sagemaker-igw"
  }
}

# Route Table
resource "aws_route_table" "rt" {
  vpc_id = aws_vpc.sagemaker_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = {
    Name = "sagemaker-rt"
  }
}

# Route Table Association
resource "aws_route_table_association" "a" {
  subnet_id      = aws_subnet.private_subnet_a.id
  route_table_id = aws_route_table.rt.id
}

resource "aws_route_table_association" "b" {
  subnet_id      = aws_subnet.private_subnet_b.id
  route_table_id = aws_route_table.rt.id
}

# S3 Bucket
resource "aws_s3_bucket" "sagemaker_bucket" {
  bucket = "sagemaker-model-storage-us-east-1"
  acl    = "private"

  tags = {
    Name        = "sagemaker-model-storage"
    Environment = "dev"
  }
}

# IAM Role for SageMaker
resource "aws_iam_role" "sagemaker_execution" {
  name = "sagemaker_execution_role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect = "Allow",
        Principal = {
          Service = "sagemaker.amazonaws.com"
        },
        Action = "sts:AssumeRole"
      }
    ]
  })

  tags = {
    Name = "sagemaker_execution_role"
  }
}

resource "aws_iam_role_policy" "sagemaker_execution_policy" {
  role = aws_iam_role.sagemaker_execution.id

  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect = "Allow",
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents",
          "sagemaker:CreateTrainingJob",
          "sagemaker:CreateModel",
          "sagemaker:CreateEndpointConfig",
          "sagemaker:CreateEndpoint",
          "sagemaker:InvokeEndpoint"
        ],
        Resource = "*"
      }
    ]
  })
}

# SageMaker Training Job
resource "aws_sagemaker_training_job" "terraform_training" {
  name                      = "terraform-training-job"
  role_arn                  = aws_iam_role.sagemaker_execution.arn
  algorithm_specification {
    training_image = "763104351884.dkr.ecr.us-east-1.amazonaws.com/pytorch-training:1.7.1-cpu-py36-ubuntu18.04"
    training_input_mode = "File"
  }
  input_data_config {
    channel_name = "train"
    data_source {
      s3_data_source {
        s3_data_type = "S3Prefix"
        s3_uri       = "s3://${aws_s3_bucket.sagemaker_bucket.bucket}/train"
        s3_data_distribution_type = "FullyReplicated"
      }
    }
  }
  output_data_config {
    s3_output_path = "s3://${aws_s3_bucket.sagemaker_bucket.bucket}/output"
  }
  resource_config {
    instance_type   = "ml.m5.large"
    instance_count  = 1
    volume_size_in_gb = 10
  }
  stopping_condition {
    max_runtime_in_seconds = 3600
  }
  hyper_parameters = {
    "epochs" = "3"
  }
}

# SageMaker Model
resource "aws_sagemaker_model" "sagemaker_model" {
  name                 = "sagemaker-model"
  execution_role_arn   = aws_iam_role.sagemaker_execution.arn
  primary_container {
    image             = "763104351884.dkr.ecr.us-east-1.amazonaws.com/pytorch-inference:1.7.1-cpu-py36-ubuntu18.04"
    model_data_url    = "s3://${aws_s3_bucket.sagemaker_bucket.bucket}/output/model.tar.gz"
    environment = {
      SAGEMAKER_CONTAINER_LOG_LEVEL = "20"
      SAGEMAKER_PROGRAM             = "inference.py"
      SAGEMAKER_REGION              = "us-east-1"
    }
  }
}

# SageMaker Endpoint Configuration
resource "aws_sagemaker_endpoint_configuration" "sagemaker_endpoint_config" {
  name = "sagemaker-endpoint-config"

  production_variants {
    variant_name           = "AllTraffic"
    model_name             = aws_sagemaker_model.sagemaker_model.name
    initial_instance_count = 1
    instance_type          = "ml.m5.large"
  }
}

# SageMaker Endpoint
resource "aws_sagemaker_endpoint" "sagemaker_endpoint" {
  name = "sagemaker-endpoint"
  endpoint_config_name = aws_sagemaker_endpoint_configuration.sagemaker_endpoint_config.name
}

# Outputs
output "sagemaker_endpoint_name" {
  value = aws_sagemaker_endpoint.sagemaker_endpoint.name
}
```

### Detailed Walkthrough

1. **Set Up VPC and Subnets**
   - Create a VPC and two private subnets to host your SageMaker infrastructure.
   - Set up an Internet Gateway and Route Table to allow outbound internet access from the subnets.

2. **Create S3 Bucket**
   - Create an S3 bucket to store your training data and model artifacts.
   - Upload your training data to the `s3://${aws_s3_bucket.sagemaker_bucket.bucket}/train` directory.

3. **Create IAM Role and Policy**
   - Create an IAM role that SageMaker can assume.
   - Attach a policy to the role that grants necessary permissions for SageMaker operations.

4. **Set Up SageMaker Training Job**
   - Define a SageMaker training job that uses a PyTorch training image.
   - Specify the input data location (`s3://${aws_s3_bucket.sagemaker_bucket.bucket}/train`).
   - Configure the output data location (`s3://${aws_s3_bucket.sagemaker_bucket.bucket}/output`).
   - Configure the instance type and resource limits for the training job.

5. **Create SageMaker Model**
   - Define a SageMaker model using the output from the training job (`s3://${aws_s3_bucket.sagemaker_bucket.bucket}/output/model.tar.gz`).

6. **Set Up SageMaker Endpoint Configuration**
   - Create an endpoint configuration that specifies the model and the instance type for the endpoint.

7. **Create SageMaker Endpoint**
   - Create a SageMaker endpoint using the endpoint configuration.

8. **Deploy the Configuration**
   - Use the `terraform apply` command to deploy the configuration and set up your SageMaker infrastructure.

9. **Query the Model**
   - Use the SageMaker endpoint to query the model for completions.
   - You can use the AWS SDK for Python (boto3) to send requests to the endpoint and get responses.

### Sample Python Code to Query the Model

```python
import boto3

sagemaker_runtime = boto3.client('sagemaker-runtime', region_name='us-east-1')

response = sagemaker_runtime.invoke_endpoint(
    EndpointName='sagemaker-endpoint',
    Body=b'{"prompt": "resource \\"aws_instance\\" \\"example\\" {"}',
    ContentType='application/json'
)

result = response['Body'].read().decode('utf-8')
print(result)
```

This sample Python script sends a prompt to the SageMaker endpoint and prints the generated Terraform code completion. Adjust the `Body` parameter with your specific prompt.
