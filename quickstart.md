# **Simplify application authentication with mutual TLS in Amazon ECS by using Application Load Balancer**



## **Prerequisites**

* An active AWS account with access to deploy AWS CloudFormation stacks. Make sure that you have AWS Identity and Access Management (IAM) [user or role permissions](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/control-access-with-iam.html) to deploy CloudFormation.
* AWS Command Line Interface (AWS CLI) [installed](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html). [Configure](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html) your AWS credentials on your local machine or in your environment by either using the AWS CLI or by setting the environment variables in the `~/.aws/credentials` file.
* OpenSSL [installed](https://www.openssl.org/).
* jq [installed](https://jqlang.github.io/jq/).
* Docker [installed](https://www.docker.com/get-started/).
* Familiarity with the AWS services described in [Tools](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/simplify-application-authentication-with-mutual-tls-in-amazon-ecs.html#simplify-application-authentication-with-mutual-tls-in-amazon-ecs-tools).
* Knowledge of Docker and NGINX.

**Limitations**

* Mutual TLS for Application Load Balancer only supports X.509v3 client certificates. X.509v1 client certificates are not supported.
* The CloudFormation template that is provided in this pattern’s code repository doesn’t include provisioning a CodeBuild project as part of the stack.
* This pattern is sample code for demonstration. It is not intended for production use without additional security review and hardening. In particular, the sample builds each service's client certificate and private key into its container image. For production, store the private key in AWS Secrets Manager and inject it at task start, restrict who can pull the images and use ECS Exec, and automate certificate rotation and revocation.

**How the deployment works**

The CloudFormation stack enables mutual TLS itself: it creates the Elastic Load Balancing trust store and sets `MutualAuthentication` to `verify` on both listeners. The trust store is built from a CA bundle in the stack's S3 bucket, so you deploy the stack in two passes:

1. **First pass:** deploy without a CA bundle. The stack creates the network, load balancer, buckets and repositories. Both ECS services run **zero tasks**, so nothing is reachable before mutual TLS is enforced.
2. **Second pass:** after you upload the CA bundle and push your images, update the stack with the CA bundle key, the image URIs and `EnableECSExec=true` (used to test the pattern). The trust store is created, both listeners switch to mutual TLS verify mode, and the services start.



## **Create the repository**

To create a Git repository to contain the Dockerfile and the `buildspec.yaml` files, use the following steps:

* Create a folder in your virtual environment. Name it with your project name.
* Open a terminal on your local machine, and navigate to this folder.
* To clone the [mTLS-with-Application-Load-Balancer-in-Amazon-ECS](https://github.com/aws-samples/mTLS-with-Application-Load-Balancer-in-Amazon-ECS) repository to your project directory, enter the following command:

```
git clone https://github.com/aws-samples/mTLS-with-Application-Load-Balancer-in-Amazon-ECS.git
```

The repository's `.gitignore` excludes `*.pem`, `*.key`, `*.crt`, `*.cert` and `*.csr` files so that certificates and private keys you generate are never committed.



## **Create CA and generate certificates**

**Create a private CA in AWS Private CA.**

To create a private certificate authority (CA), run the following commands in your terminal. Replace the values in the example variables with your own values.

```
export AWS_DEFAULT_REGION="us-west-2"
export SERVICES_DOMAIN="example.com"

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
ROOT_CA_CSR=`aws acm-pca get-certificate-authority-csr \
    --certificate-authority-arn ${ROOT_CA_ARN} \
    --query Csr --output text`
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
    --domain-name "*.${SERVICES_DOMAIN}" \
    --certificate-authority-arn ${ROOT_CA_ARN} \
    --query CertificateArn --output text`
```

**Use the private CA to issue a client certificate.**

Generate the client private keys and certificate signing requests (CSRs) for the two services in a directory that only you can read. The keys are not protected by a passphrase because NGINX and curl in the container read them non-interactively, so treat these files as secrets:

```
mkdir -p client-certs && chmod 700 client-certs && cd client-certs
umask 077
openssl req -out client_csr1.pem -new -newkey rsa:2048 -nodes -keyout client_private-key1.pem -subj "/CN=service-one.${SERVICES_DOMAIN}"
openssl req -out client_csr2.pem -new -newkey rsa:2048 -nodes -keyout client_private-key2.pem -subj "/CN=service-two.${SERVICES_DOMAIN}"
```

To issue a certificate for each service, run the following commands to use the private CA that you created. The client certificates are valid for 365 days; plan to reissue them before they expire, because the listeners reject expired client certificates.

```
SERVICE_ONE_CERT_ARN=`aws acm-pca issue-certificate \
    --certificate-authority-arn ${ROOT_CA_ARN} \
    --csr fileb://client_csr1.pem \
    --signing-algorithm "SHA256WITHRSA" \
    --validity Value=365,Type="DAYS" --query CertificateArn --output text` 
echo "SERVICE_ONE_CERT_ARN: ${SERVICE_ONE_CERT_ARN}"
aws acm-pca get-certificate \
    --certificate-authority-arn ${ROOT_CA_ARN} \
    --certificate-arn ${SERVICE_ONE_CERT_ARN} \
     | jq -r '.Certificate' > client_cert1.cert
SERVICE_TWO_CERT_ARN=`aws acm-pca issue-certificate \
    --certificate-authority-arn ${ROOT_CA_ARN} \
    --csr fileb://client_csr2.pem \
    --signing-algorithm "SHA256WITHRSA" \
    --validity Value=365,Type="DAYS" --query CertificateArn --output text` 
echo "SERVICE_TWO_CERT_ARN: ${SERVICE_TWO_CERT_ARN}"
aws acm-pca get-certificate \
    --certificate-authority-arn ${ROOT_CA_ARN} \
    --certificate-arn ${SERVICE_TWO_CERT_ARN} \
     | jq -r '.Certificate' > client_cert2.cert
cd ..
```

## **Provision AWS services**

**Deploy the CloudFormation template (first pass).**

To provision the virtual private cloud (VPC), Amazon ECS cluster, Amazon ECS services, Application Load Balancer, Amazon S3 buckets and Amazon Elastic Container Registry (Amazon ECR) repositories, deploy the CloudFormation template with the ACM certificate that you requested. Leave `TrustStoreCABundleS3Key` empty on this pass; the services start with zero tasks.

```
aws cloudformation deploy \
    --stack-name ecs-mtls \
    --template-file apg-infra.yaml \
    --capabilities CAPABILITY_IAM \
    --parameter-overrides CertificateArn=${TLS_CERTIFICATE_ARN}
```

**Get variables.**

To retrieve the resource details and store them as variables, use the following commands:

```
stack_output() {
  aws cloudformation describe-stacks --stack-name ecs-mtls --output text \
    --query "Stacks[0].Outputs[?OutputKey=='$1'].OutputValue"
}
export LoadBalancerDNS=$(stack_output LoadBalancerDNS)
export ECRRepositoryUri=$(stack_output ECRRepositoryUri)
export ECRRepositoryServiceOneUri=$(stack_output ECRRepositoryServiceOneUri)
export ECRRepositoryServiceTwoUri=$(stack_output ECRRepositoryServiceTwoUri)
export ClusterName=$(stack_output ClusterName)
export BucketName=$(stack_output BucketName)
```

**Upload the CA certificate and client certificates to Amazon S3.**

The bucket blocks public access, enforces TLS, and is encrypted. Upload the CA bundle for the trust store, and each service's client certificate, private key and the CA certificate for the image builds:

```
# CA bundle for the ALB trust store
aws s3 cp ca-cert.pem s3://$BucketName/acm-trust-store/ca-cert.pem
# for service one
aws s3 cp client-certs/client_cert1.cert s3://$BucketName/serviceone/
aws s3 cp client-certs/client_private-key1.pem s3://$BucketName/serviceone/
aws s3 cp ca-cert.pem s3://$BucketName/serviceone/
# for service two
aws s3 cp client-certs/client_cert2.cert s3://$BucketName/servicetwo/
aws s3 cp client-certs/client_private-key2.pem s3://$BucketName/servicetwo/
aws s3 cp ca-cert.pem s3://$BucketName/servicetwo/
```

After the upload, delete the local copies of the private keys and CSRs:

```
rm -f client-certs/client_private-key*.pem client-certs/client_csr*.pem
```

**Create a CodeBuild project.**

To use a CodeBuild project to create the Docker images for your Amazon ECS services, do the following:
Sign in to the AWS Management Console, and open the CodeBuild console at [https://console.aws.amazon.com/codesuite/codebuild/](https://console.aws.amazon.com/codesuite/codebuild/).
Create a new project. For **Source**, choose the Git repository that you created. For information about different kinds of Git repository integration, see [Working with connections](https://docs.aws.amazon.com/dtconsole/latest/userguide/connections.html) in the AWS documentation.
Confirm that **Privileged** mode is enabled. To build Docker images, this mode is necessary. Otherwise, the image will not build successfully.
Use the custom `buildspec.yaml` file shared for each service.
Provide values for the project name and description.

Give the CodeBuild service role only the permissions it needs: `s3:GetObject` and `s3:ListBucket` on the service's own prefix in the bucket (`serviceone/` or `servicetwo/`), push access to that service's ECR repository, and `ecr:GetAuthorizationToken`.

**Build the Docker images.**

You can use CodeBuild to perform the image build process. As part of the process, the Docker image is built and pushed to the Amazon ECR repository. The repositories use immutable tags, so each build pushes a unique tag (the short commit hash) instead of `latest`. For details about the template and the code, see [Additional information](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/simplify-application-authentication-with-mutual-tls-in-amazon-ecs.html#simplify-application-authentication-with-mutual-tls-in-amazon-ecs-additional).

The Dockerfile copies the client certificate and private key to `/usr/local/share/ca-certificates/client.crt` and `/usr/local/share/ca-certificates/client.key`. Also copy the CA certificate so that the container can verify the load balancer's server certificate, for example `COPY ca-cert.pem /usr/local/share/ca-certificates/ca-cert.pem`.

Pin the Dockerfile's base image by digest instead of a mutable tag such as `latest`, so that a changed upstream image cannot enter your build unnoticed. To find the digest, run `docker pull public.ecr.aws/nginx/nginx:<version>` and use the `Digest` it prints, for example `FROM public.ecr.aws/nginx/nginx:<version>@sha256:<digest>`. For production, consider an [Amazon ECR pull through cache](https://docs.aws.amazon.com/AmazonECR/latest/userguide/pull-through-cache.html) and [Amazon Inspector enhanced scanning](https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-scanning-enhanced.html).

(Optional) To build locally for test purposes, use the following commands:

```
export IMAGE_TAG=$(git rev-parse --short HEAD)
# login to ECR
aws ecr get-login-password | docker login --username AWS --password-stdin $ECRRepositoryUri
# build image for service one
cd service1
aws s3 cp s3://$BucketName/serviceone/ . --recursive
docker build -t $ECRRepositoryServiceOneUri:$IMAGE_TAG .
docker push $ECRRepositoryServiceOneUri:$IMAGE_TAG
rm -f client_private-key*.pem
# build image for service two
cd ../service2
aws s3 cp s3://$BucketName/servicetwo/ . --recursive
docker build -t $ECRRepositoryServiceTwoUri:$IMAGE_TAG .
docker push $ECRRepositoryServiceTwoUri:$IMAGE_TAG
rm -f client_private-key*.pem
cd ..
```

## **Enable mutual TLS and start the services**

**Update the CloudFormation stack (second pass).**

Update the stack with the CA bundle key and the image URIs you pushed. CloudFormation creates the trust store from `acm-trust-store/ca-cert.pem`, enables mutual TLS (`Mode=verify`, expired client certificates rejected) on both listeners with the `ELBSecurityPolicy-TLS13-1-2-2021-06` TLS policy, updates both task definitions to your images, and starts one task per service.

This tutorial tests mutual TLS from inside the tasks with ECS Exec, so the command also sets `EnableECSExec=true`. ECS Exec must be on when a task starts, which is why it is set on this pass. The template defaults it to `false`, because ECS Exec gives an interactive shell in the task, including read access to the client private key.

```
aws cloudformation deploy \
    --stack-name ecs-mtls \
    --template-file apg-infra.yaml \
    --capabilities CAPABILITY_IAM \
    --parameter-overrides \
        CertificateArn=${TLS_CERTIFICATE_ARN} \
        TrustStoreCABundleS3Key=acm-trust-store/ca-cert.pem \
        ServiceOneImage=${ECRRepositoryServiceOneUri}:${IMAGE_TAG} \
        ServiceTwoImage=${ECRRepositoryServiceTwoUri}:${IMAGE_TAG} \
        EnableECSExec=true
```

Because the listeners, trust store and task definitions are managed by the stack, use the same command with new values when you rotate the CA bundle or deploy new images. If you rotate the CA bundle, upload it under a new object key and pass that key, so that CloudFormation updates the trust store.

## **Access the application**

Use the Amazon ECS console to view the task. When the task status has been updated to **Running**, select the task. In the **Task** section, copy the task ID.

**Test your application**

Use ECS Exec to access the tasks. Grant `ecs:ExecuteCommand` only to the people who test the pattern, scoped to this cluster.
For service one, use the following command:

```
container="nginx"

ECS_EXEC_TASK_ARN="<TASK ARN>"
aws ecs execute-command --cluster $ClusterName \
    --task $ECS_EXEC_TASK_ARN \
    --container $container \
    --interactive \
    --command "/bin/bash"
```

The load balancer's server certificate is issued for `*.<your SERVICES_DOMAIN>`, not for the load balancer's DNS name. The commands below use curl's `--connect-to` option to send the request to the load balancer while verifying its certificate against a name the certificate covers (here `mtls.example.com`; replace `example.com` with your `SERVICES_DOMAIN`). `--cacert` makes curl verify the server certificate against your private CA, so the connection fails if anything other than your load balancer answers.

In the service one task’s container, call the listener that points to service two:

```
curl -vs https://mtls.example.com:8090 \
    --connect-to mtls.example.com:8090:<internal-alb-dns-name>:8090 \
    --cacert /usr/local/share/ca-certificates/ca-cert.pem \
    --key /usr/local/share/ca-certificates/client.key \
    --cert /usr/local/share/ca-certificates/client.crt
```

In the service two task’s container, call the listener that points to service one:

```
curl -vs https://mtls.example.com:8080 \
    --connect-to mtls.example.com:8080:<internal-alb-dns-name>:8080 \
    --cacert /usr/local/share/ca-certificates/ca-cert.pem \
    --key /usr/local/share/ca-certificates/client.key \
    --cert /usr/local/share/ca-certificates/client.crt
```

To confirm that mutual TLS is enforced, repeat either command without the `--key` and `--cert` options. The load balancer rejects the TLS handshake because no client certificate is presented.

Do not use curl's `-k` (`--insecure`) option. It disables server certificate validation, which lets any endpoint impersonate the load balancer.

**Turn off ECS Exec after testing**

When you finish testing, run the second-pass `aws cloudformation deploy` command again without `EnableECSExec=true` (it defaults to `false`). While ECS Exec is on, grant `ecs:ExecuteCommand` only to the people who test the pattern, scoped to this cluster.

## **Production considerations**

This pattern is sample code. Before you adapt it for production, address the following, in addition to the limitations listed at the start of this guide:

* **Client private keys.** Don't build keys into images or store them as plain objects in S3. Store each service's key in AWS Secrets Manager, encrypted with a customer managed AWS KMS key, and inject it at task start through the task definition `secrets` parameter. Add Amazon ECR repository policies so that only the task execution role and the build role can pull images.
* **Who can issue trusted certificates.** Anyone who can call `acm-pca:IssueCertificate` on the CA, or `acm:ExportCertificate` on certificates it issued, can create a client identity that the load balancer accepts. Issue client certificates from a subordinate CA, keep the root CA in a separate account, grant these actions only to a dedicated automation role, and alarm on them in AWS CloudTrail.
* **The trust store bundle.** Anyone who can write `acm-trust-store/*` in the bucket and update the stack or call `elbv2:ModifyTrustStore` can add a CA to the trust store. Restrict `s3:PutObject` and `s3:DeleteObject` on that prefix to your deployment role, and alarm on `ModifyTrustStore` and on writes to the prefix.
* **Per-service authorization.** Mutual TLS authenticates the client but doesn't decide which client may call which service: in this sample, both listeners trust the same CA. Check the client identity in each service, for example the `X-Amzn-Mtls-Clientcert-Subject` header that the load balancer adds, against a list of allowed callers. Alternatively, issue each service's clients from a separate subordinate CA and give each listener its own trust store.
* **Revocation and rotation.** Publish certificate revocation lists (CRLs) from AWS Private CA, add them to the trust store, use short-lived client certificates, and automate rotation.
* **Build pipeline.** Give the CodeBuild service role access only to its own S3 prefix and ECR repository, protect the source branch, and don't build untrusted pull requests.
* **Network and audit.** Use VPC endpoints and restrict task egress, enable VPC Flow Logs, CloudTrail S3 data events on the bucket, and Amazon GuardDuty, and run at least two tasks per service.
