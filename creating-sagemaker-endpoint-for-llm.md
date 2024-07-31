### Step-by-Step Guide to Using a Hugging Face Model in SageMaker

#### Step 1: Setting Up the SageMaker Notebook Instance

1. **Log in to AWS Console**: Navigate to the AWS Management Console and select the SageMaker service.

2. **Create a Notebook Instance**:
   - Click on **Notebook instances** in the left menu, then click **Create notebook instance**.
   - Provide a name for your notebook instance, for example, `code-llama-notebook`.
   - Choose an instance type, such as `ml.t2.medium`.
   - For the IAM role, create a new role with SageMaker permissions or use an existing one.
   - Click **Create notebook instance**.

3. **Open the Notebook Instance**:
   - Once the instance is in `InService` status, click **Open Jupyter** next to your notebook instance.

#### Step 2: Installing Required Libraries

1. **Install Hugging Face Transformers and Datasets**:
   - Run the following code in a notebook cell:

     ```python
     !pip install transformers
     !pip install datasets
     ```

#### Step 3: Loading and Saving the Model

1. **Load the Hugging Face Model**:
   - Run the following code to load the CodeLlama model:

     ```python
     from transformers import AutoModelForCausalLM, AutoTokenizer

     model_name = "codellama/CodeLlama-7b-Instruct-hf"
     model = AutoModelForCausalLM.from_pretrained(model_name)
     tokenizer = AutoTokenizer.from_pretrained(model_name)

     model.save_pretrained("/opt/ml/model")
     tokenizer.save_pretrained("/opt/ml/model")
     ```

2. **Create Inference Script**:
   - Create a new file named `inference.py` with the following content:

     ```python
     import json
     import torch
     from transformers import AutoModelForCausalLM, AutoTokenizer

     def model_fn(model_dir):
         model = AutoModelForCausalLM.from_pretrained(model_dir)
         tokenizer = AutoTokenizer.from_pretrained(model_dir)
         return model, tokenizer

     def predict_fn(input_data, model_and_tokenizer):
         model, tokenizer = model_and_tokenizer
         inputs = tokenizer.encode(input_data['inputs'], return_tensors='pt')
         outputs = model.generate(inputs, max_length=200)
         result = tokenizer.decode(outputs[0], skip_special_tokens=True)
         return result

     def input_fn(request_body, request_content_type):
         if request_content_type == 'application/json':
             return json.loads(request_body)
         raise ValueError("Unsupported content type: {}".format(request_content_type))

     def output_fn(prediction, response_content_type):
         if response_content_type == 'application/json':
             return json.dumps(prediction)
         raise ValueError("Unsupported content type: {}".format(response_content_type))
     ```

#### Step 4: Deploying the Model

1. **Create a Training Job**:
   - Zip the model directory and upload it to an S3 bucket.

     ```sh
     tar -czvf model.tar.gz /opt/ml/model
     aws s3 cp model.tar.gz s3://your-bucket-name/model/
     ```

2. **Create a SageMaker Model**:
   - In the SageMaker console, click **Models** in the left menu, then **Create model**.
   - Provide a model name, e.g., `code-llama-model`.
   - For the **Container** section, select **Use a model URL from Amazon S3** and enter the S3 path to your `model.tar.gz`.
   - For the **IAM role**, choose the role you created earlier.
   - Click **Create model**.

3. **Create an Endpoint Configuration**:
   - Click **Endpoint configurations** in the left menu, then **Create endpoint configuration**.
   - Provide a name, e.g., `code-llama-endpoint-config`.
   - Add a model by selecting the model you just created.
   - Click **Create endpoint configuration**.

4. **Create an Endpoint**:
   - Click **Endpoints** in the left menu, then **Create endpoint**.
   - Provide an endpoint name, e.g., `code-llama-endpoint`.
   - Select the endpoint configuration you just created.
   - Click **Create endpoint** and wait for it to be in `InService` status.

#### Step 5: Securing the SageMaker Endpoint

1. **Create an IAM Role for Lambda**:
   - Go to the IAM console, and click **Roles** then **Create role**.
   - Select **Lambda** and click **Next**.
   - Attach the policy `AmazonSageMakerFullAccess` and any other necessary policies.
   - Click **Next**, give the role a name (e.g., `LambdaSageMakerRole`), and create the role.

2. **Modify Endpoint Policy**:
   - Go to the SageMaker console, select your endpoint, and click **Edit policy**.
   - Add a resource policy to restrict access to the Lambda role. Example policy:

     ```json
     {
         "Version": "2012-10-17",
         "Statement": [
             {
                 "Effect": "Allow",
                 "Principal": {
                     "AWS": "arn:aws:iam::your-account-id:role/LambdaSageMakerRole"
                 },
                 "Action": "sagemaker:InvokeEndpoint",
                 "Resource": "arn:aws:sagemaker:region:your-account-id:endpoint/code-llama-endpoint"
             }
         ]
     }
     ```

#### Step 6: Creating the Lambda Function

1. **Create a Lambda Function**:
   - Go to the Lambda console and click **Create function**.
   - Choose **Author from scratch**, name the function (e.g., `QueryCodeLlamaEndpoint`), and choose the runtime as Python 3.9.
   - Under **Permissions**, select the role you created earlier (`LambdaSageMakerRole`).

2. **Add Lambda Function Code**:
   - In the Lambda console, add the following code to your function:

     ```python
     import json
     import boto3

     def lambda_handler(event, context):
         runtime = boto3.client('runtime.sagemaker')
         endpoint_name = 'code-llama-endpoint'
         prompt = event.get('prompt', '')

         response = runtime.invoke_endpoint(
             EndpointName=endpoint_name,
             ContentType='application/json',
             Body=json.dumps({"inputs": prompt})
         )
         result = json.loads(response['Body'].read().decode())

         return {
             'statusCode': 200,
             'body': json.dumps(result)
         }
     ```

#### Step 7: Testing the Lambda Function

1. **Create a Test Event**:
   - In the Lambda console, click **Test** and create a new test event.
   - Use the following JSON for the test event:

     ```json
     {
         "prompt": "Create a Terraform module for an S3 bucket with versioning enabled."
     }
     ```

2. **Run the Test**:
   - Click **Test** and check the response. You should see the generated Terraform module in the output.
