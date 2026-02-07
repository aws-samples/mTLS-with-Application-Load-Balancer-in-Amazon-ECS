# **Simplify application authentication with mutual TLS in Amazon ECS by using Application Load Balancer**



## **Prerequisites**

* An active AWS account with access to deploy AWS CloudFormation stacks. Make sure that you have AWS Identity and Access Management (IAM) [user or role permissions](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/control-access-with-iam.html) to deploy CloudFormation.
* AWS Command Line Interface (AWS CLI) [installed](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html). [Configure](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html) your AWS credentials on your local machine or in your environment by either using the AWS CLI or by setting the environment variables in the `~/.aws/credentials` file.
* OpenSSL [installed](https://www.openssl.org/).
* Docker [installed](https://www.docker.com/get-started/).
* Familiarity with the AWS services described in [Tools](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/simplify-application-authentication-with-mutual-tls-in-amazon-ecs.html#simplify-application-authentication-with-mutual-tls-in-amazon-ecs-tools).
* Knowledge of Docker and NGINX.

**Limitations**

* Mutual TLS for Application Load Balancer only supports X.509v3 client certificates. X.509v1 client certificates are not supported.
* The CloudFormation template that is provided in this pattern’s code repository doesn’t include provisioning a CodeBuild project as part of the stack.




## **Create the repository**

To create a Git repository to contain the Dockerfile and the `buildspec.yaml` files, use the following steps:

* Create a folder in your virtual environment. Name it with your project name.
* Open a terminal on your local machine, and navigate to this folder.
* To clone the [mTLS-with-Application-Load-Balancer-in-Amazon-ECS](https://github.com/aws-samples/mTLS-with-Application-Load-Balancer-in-Amazon-ECS) repository to your project directory, enter the following command:

```
git clone https://github.com/aws-samples/mTLS-with-Application-Load-Balancer-in-Amazon-ECS.git
```



## **Create CA and generate certificates**

Verify that you have an Amazon ECS cluster with two services running. To retrieve the resource details and store them as variables, use the following commands:

**Create a private CA in AWS Private CA.**

To create a private certificate authority (CA), run the following commands in your terminal. Replace the values in the example variables with your own values.

```
export AWS_DEFAULT_REGION="us-west-2"
export SERVICES_DOMAIN="www.example.com"

export ROOT_CA_ARN=`aws acm-pca create-certificate-authority \
    --certificate-authority-type ROOT \
    --certificate-authority-configuration \
    "KeyAlgorithm=RSA_2048,
    SigningAlgorithm=SHA256WITHRSA,
    Subject={
        Country=US,
        State=WA,
        Locality=Seattle,
        Organization=Build on AWS,
        OrganizationalUnit=mTLS Amazon ECS and ALB Example,
        CommonName=${SERVICES_DOMAIN}}" \
        --query CertificateAuthorityArn --output text`
```

**Create and install your private CA certificate.**

* Generate a certificate signing request (CSR).

```
`ROOT_CA_CSR=`aws acm-pca get-certificate-authority-csr \
`*`--certificate-authority-arn ${ROOT_CA_ARN} \
--query Csr --output text``*
```

* Issue the root certificate.

```
AWS_CLI_VERSION=$(aws --version 2>&1 | cut -d/ -f2 | cut -d. -f1)
[[ ${AWS_CLI_VERSION} -gt 1 ]] && ROOT_CA_CSR="$(echo ${ROOT_CA_CSR} | base64)"
ROOT_CA_CERT_ARN=`aws acm-pca issue-certificate \
    --certificate-authority-arn ${ROOT_CA_ARN} \
    --template-arn arn:aws:acm-pca:::template/RootCACertificate/V1 \
    --signing-algorithm SHA256WITHRSA \
    --validity Value=10,Type=YEARS \
    --csr "${ROOT_CA_CSR}" \
    --query CertificateArn --output text`
```

* Retrieve the root certificate.

```
ROOT_CA_CERT=`aws acm-pca get-certificate \
    --certificate-arn ${ROOT_CA_CERT_ARN} \
    --certificate-authority-arn ${ROOT_CA_ARN} \
    --query Certificate --output text`
# store for later use
aws acm-pca get-certificate \
    --certificate-arn ${ROOT_CA_CERT_ARN} \
    --certificate-authority-arn ${ROOT_CA_ARN} \
    --query Certificate --output text > ca-cert.pem
```

* Import the root CA certificate to install it on the CA.

```
[[ ${AWS_CLI_VERSION} -gt 1 ]] && ROOT_CA_CERT="$(echo ${ROOT_CA_CERT} | base64)"
aws acm-pca import-certificate-authority-certificate \
    --certificate-authority-arn $ROOT_CA_ARN \
    --certificate "${ROOT_CA_CERT}"
```

**Request a managed certificate.**

To request a private certificate in AWS Certificate Manager to use with your private ALB, use the following command:

```
export TLS_CERTIFICATE_ARN=`aws acm request-certificate \
    --domain-name "*.${DOMAIN_DOMAIN}" \
    --certificate-authority-arn ${ROOT_CA_ARN} \
    --query CertificateArn --output text`
```

**Use the private CA to issue a client certificate.**

To create a certificate signing request (CSR) for the two services, use the following AWS CLI command:
`openssl req -out client_csr1.pem -new -newkey rsa:2048 -nodes -keyout client_private-key1.pem`
`openssl req -out client_csr2.pem -new -newkey rsa:2048 -nodes -keyout client_private-key2.pem`
This command returns the CSR and the private key for the two services.
To issue a certificate for the services, run the following commands to use the private CA that you created:

```
SERVICE_ONE_CERT_ARN=`aws acm-pca issue-certificate \
    --certificate-authority-arn ${ROOT_CA_ARN} \
    --csr fileb://client_csr1.pem \
    --signing-algorithm "SHA256WITHRSA" \
    --validity Value=5,Type="YEARS" --query CertificateArn --output text` 
echo "SERVICE_ONE_CERT_ARN: ${SERVICE_ONE_CERT_ARN}"
aws acm-pca get-certificate \
    --certificate-authority-arn ${ROOT_CA_ARN} \
    --certificate-arn ${SERVICE_ONE_CERT_ARN} \
     | jq -r '.Certificate' > client_cert1.cert
SERVICE_TWO_CERT_ARN=`aws acm-pca issue-certificate \
    --certificate-authority-arn ${ROOT_CA_ARN} \
    --csr fileb://client_csr2.pem \
    --signing-algorithm "SHA256WITHRSA" \
    --validity Value=5,Type="YEARS" --query CertificateArn --output text` 
echo "SERVICE_TWO_CERT_ARN: ${SERVICE_TWO_CERT_ARN}"
aws acm-pca get-certificate \
    --certificate-authority-arn ${ROOT_CA_ARN} \
    --certificate-arn ${SERVICE_TWO_CERT_ARN} \
     | jq -r '.Certificate' > client_cert2.cert
```

## **Provision AWS services**

**Provision AWS services with the CloudFormation template.**

To provision the virtual private cloud (VPC), Amazon ECS cluster, Amazon ECS services, Application Load Balancer, and Amazon Elastic Container Registry (Amazon ECR), use the CloudFormation template.

**Get variables.**

Verify that you have an Amazon ECS cluster with two services running. To retrieve the resource details and store them as variables, use the following commands:

```
export LoadBalancerDNS=$(aws cloudformation describe-stacks --stack-name ecs-mtls \
--output text \
--query 'Stacks[0].Outputs[?OutputKey==`LoadBalancerDNS`].OutputValue')
export ECRRepositoryUri=$(aws cloudformation describe-stacks --stack-name ecs-mtls \
--output text \
--query 'Stacks[0].Outputs[?OutputKey==`ECRRepositoryUri`].OutputValue')
export ECRRepositoryServiceOneUri=$(aws cloudformation describe-stacks --stack-name ecs-mtls \
--output text \
--query 'Stacks[0].Outputs[?OutputKey==`ECRRepositoryServiceOneUri`].OutputValue')
export ECRRepositoryServiceTwoUri=$(aws cloudformation describe-stacks --stack-name ecs-mtls \
--output text \
--query 'Stacks[0].Outputs[?OutputKey==`ECRRepositoryServiceTwoUri`].OutputValue')
export ClusterName=$(aws cloudformation describe-stacks --stack-name ecs-mtls \
--output text \
--query 'Stacks[0].Outputs[?OutputKey==`ClusterName`].OutputValue')
export BucketName=$(aws cloudformation describe-stacks --stack-name ecs-mtls \
--output text \
--query 'Stacks[0].Outputs[?OutputKey==`BucketName`].OutputValue')
export Service1ListenerArn=$(aws cloudformation describe-stacks --stack-name ecs-mtls \
--output text \
--query 'Stacks[0].Outputs[?OutputKey==`Service1ListenerArn`].OutputValue')
export Service2ListenerArn=$(aws cloudformation describe-stacks --stack-name ecs-mtls \
--output text \
--query 'Stacks[0].Outputs[?OutputKey==`Service2ListenerArn`].OutputValue')
```

**Create a CodeBuild project.**

To use a CodeBuild project to create the Docker images for your Amazon ECS services, do the following:
Sign in to the AWS Management Console, and open the CodeBuild console at [https://console.aws.amazon.com/codesuite/codebuild/](https://docs.aws.amazon.com/dtconsole/latest/userguide/connections.html).
Create a new project. For **Source**, choose the Git repository that you created. For information about different kinds of Git repository integration, see [Working with connections](https://docs.aws.amazon.com/dtconsole/latest/userguide/connections.html) in the AWS documentation.
Confirm that **Privileged **mode is enabled. To build Docker images, this mode is necessary. Otherwise, the image will not build successfully.
Use the custom `buildspec.yaml` file shared for each service.
Provide values for the project name and description.

**Build the Docker images.**

You can use CodeBuild to perform the image build process. CodeBuild needs permissions to interact with Amazon ECR and to work with Amazon S3.
As part of the process, the Docker image is built and pushed to the Amazon ECR registry. For details about the template and the code, see [Additional information](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/simplify-application-authentication-with-mutual-tls-in-amazon-ecs.html#simplify-application-authentication-with-mutual-tls-in-amazon-ecs-additional).
(Optional) To build locally for test purposes, use the following command:

```
# login to ECR
aws ecr get-login-password | docker login --username AWS --password-stdin $ECRRepositoryUri
# build image for service one
cd /service1
aws s3 cp s3://$BucketName/serviceone/ service1/ --recursive
docker build -t $ECRRepositoryServiceOneUri .
docker push $ECRRepositoryServiceOneUri
# build image for service two
cd ../service2
aws s3 cp s3://$BucketName/servicetwo/ service2/ --recursive
docker build -t $ECRRepositoryServiceTwoUri .
docker push $ECRRepositoryServiceTwoUri
```

## **Enable mutual TLS**

**Upload the CA certificate to Amazon S3.**

To upload the CA certificate to the Amazon S3 bucket, use the following example command:

```
aws s3 cp ca-cert.pem s3://$BucketName/acm-trust-store/
```

**Create the trust store.**

To create the trust store, use the following example command:

```
TrustStoreArn=`aws elbv2 create-trust-store --name acm-pca-trust-certs \
    --ca-certificates-bundle-s3-bucket $BucketName \
    --ca-certificates-bundle-s3-key acm-trust-store/ca-cert.pem --query 'TrustStores[].TrustStoreArn' --output text`
```

**Upload client certificates.**

To upload client certificates to Amazon S3 for Docker images, use the following example command:

```
# for service one
aws s3 cp client_cert1.cert s3://$BucketName/serviceone/
aws s3 cp client_private-key1.pem s3://$BucketName/serviceone/
# for service two
aws s3 cp client_cert2.cert s3://$BucketName/servicetwo/
aws s3 cp client_private-key2.pem s3://$BucketName/servicetwo/
```

**Modify the listener.**

To enable mutual TLS on the ALB, modify the HTTPS listeners by using the following commands:

```
aws elbv2 modify-listener \
    --listener-arn $Service1ListenerArn \
    --certificates CertificateArn=$TLS_CERTIFICATE_ARN_TWO \
    --ssl-policy ELBSecurityPolicy-2016-08 \
    --protocol HTTPS \
    --port 8080 \
    --mutual-authentication Mode=verify,TrustStoreArn=$TrustStoreArn,IgnoreClientCertificateExpiry=false
aws elbv2 modify-listener \
    --listener-arn $Service2ListenerArn \
    --certificates CertificateArn=$TLS_CERTIFICATE_ARN_TWO \
    --ssl-policy ELBSecurityPolicy-2016-08 \
    --protocol HTTPS \
    --port 8090 \
    --mutual-authentication Mode=verify,TrustStoreArn=$TrustStoreArn,IgnoreClientCertificateExpiry=false
```

## **Update the services**

**Update the Amazon ECS task definition.**

To update the Amazon ECS task definition, modify the `image` parameter in the new revision.
To get the values for the respective services, update the task definitions with the new Docker images Uri that you built in the previous steps: `echo $ECRRepositoryServiceOneUri` or `echo $ECRRepositoryServiceTwoUri`


```
"containerDefinitions": [
        {
            "name": "nginx",
            "image": "public.ecr.aws/nginx/nginx:latest",   # <----- change to new Uri
            "cpu": 0,
```

Update the service with the latest task definition. This task definition is the blueprint for the newly built Docker images, and it contains the client certificate that’s required for the mutual TLS authentication.
To update the service, use the following procedure:

1. Open the Amazon ECS console at https://console.aws.amazon.com/ecs/v2.
2. On the **Clusters** page, choose the cluster.
3. On the cluster details page, in the **Services** section, select the checkbox next to the service, and then choose **Update**.
4. To have your service start a new deployment, select **Force new deployment**.
5. For **Task definition**, choose the task definition family and the latest revision.
6. Choose **Update**.

Repeat the steps for the other service.


## **Access the application**

Use the Amazon ECS console to view the task. When the task status has been updated to **Running**, select the task. In the **Task **section, copy the task ID.

**Test your application**

To test your application, use ECS Exec to access the tasks.
For service one, use the following command:

```
container="nginx"

ECS_EXEC_TASK_ARN="<TASK ARN>"
aws ecs execute-command *--cluster $ClusterName \*
*--task $ECS_EXEC_TASK_ARN \*
*--container $container \*
*--interactive \*
*--command "/bin/bash"*
```

In the service one task’s container, use the following command to enter the internal load balancer `url `and the listener port that points to service two. Then specify the path to the client certificate to test the application:

```
curl -kvs https://<internal-alb-url>:8090 --key /usr/local/share/ca-certificates/client.key --cert /usr/local/share/ca-certificates/client.crt
```

In the service two task’s container, use the following command to enter the internal load balancer `url` and the listener port that points to service one. Then specify the path to the client certificate to test the application:

```
curl -kvs https://<internal-alb-url>:8080 --key /usr/local/share/ca-certificates/client.key --cert /usr/local/share/ca-certificates/client.crt
```

###### Note

The `-k` flag in the curl commands (as part of `-kvs`) disables SSL certificate validation. You can remove this flag when using an SSL certificate that matches your domain name, enabling proper certificate validation.


