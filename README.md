# tre-fn-forward

* [GitHub Actions](#github-actions)
* [GitHub Action Secrets](#github-action-secrets)
* [Using 'aws codeartifact login'](#using-aws-codeartifact-login)
* [Using 'aws codeartifact get-authorization token'](#using-aws-codeartifact-get-authorization-token)

## GitHub Actions

| Action | Description |
| --- | --- |
| [`.github/workflows/build-deploy-and-tag-ecr-image.yml`](.github/workflows/build-deploy-and-tag-ecr-image.yml) | Gets next git version tag<br>Builds and deploys Docker image to ECR<br>If successful, tags git with new version |

## GitHub Action Secrets

Secret for main AWS authentication (using OpenID Connect):

| Name                         | Description                                          |
| ---------------------------- | ---------------------------------------------------- |
| AWS_OPEN_ID_CONNECT_ROLE_ARN | ARN of AWS role used to authenticate GitHub with AWS |

Other secrets:

| Name                                | Description                                                              |
| ----------------------------------- | ------------------------------------------------------------------------ |
| AWS_CODEARTIFACT_REPOSITORY_NAME    | Name of AWS CodeArtifact repository to log in to for additional packages |
| AWS_CODEARTIFACT_REPOSITORY_DOMAIN  | Name of AWS CodeArtifact repository's domain                             |
| AWS_CODEARTIFACT_REPOSITORY_ACCOUNT | The AWS account ID that owns the CodeArtifact                            |
| AWS_REGION                          | The AWS region to use for CodeArtifact and ECR                           |

## Using 'aws codeartifact login'

*NOTE:* At time of writing this docker image does not utilise any of the packages in codeartifact, however the testing
classes, do. 

```bash
# Create a Python virtual environment
python3 -m venv .venv
. ./.venv/bin/activate

# Log in to AWS codeartifact
aws codeartifact login \
  --tool pip \
  --repository "${AWS_CODEARTIFACT_REPOSITORY_NAME}" \
  --domain "${AWS_CODEARTIFACT_REPOSITORY_DOMAIN}" \
  --domain-owner "${AWS_CODEARTIFACT_REPOSITORY_ACCOUNT}" \
  --region "${AWS_CODEARTIFACT_REPOSITORY_REGION}" \
  --profile "${AWS_PROFILE}"

# Install required libraries from AWS codeartifact
pip install \
  --require-virtualenv \
  --requirement requirements.txt
```

## Using 'aws codeartifact get-authorization token'

Note that environment variable `PIP_INDEX_URL` is passed (using `--build-arg`)
to the `Dockerfile`, where it is made accessible with `ARG PIP_INDEX_URL`, so
the `RUN` command's `pip install` process will find and use it:

```bash
# Get AWS codeartifact authentication token
CA_AUTH_TOKEN="$( \
  aws codeartifact get-authorization-token \
  --duration "${AUTH_TOKEN_DURATION}" \
  --domain "${CA_DOMAIN}" \
  --domain-owner "${AWS_ACCOUNT_ID}" \
  --query authorizationToken \
  --output text \
)"

# Create pip access URL:
export PIP_INDEX_URL=https://${CA_AUTH_USERNAME}:${CA_AUTH_TOKEN}@${CA_DOMAIN}-${AWS_ACCOUNT_ID}.d.codeartifact.${AWS_REGION}.amazonaws.com/pypi/${CA_REPOSITORY_NAME}/simple/"
```

Example `Dockerfile`

```dockerfile
FROM public.ecr.aws/lambda/python:3.8

RUN yum -y update \
    && yum clean all

# Copy function code
COPY tre_forward.py ${LAMBDA_TASK_ROOT}

# Set the CMD to your handler (could also be done as a parameter override outside of the Dockerfile)
CMD [ "tre_forward.lambda_handler" ]

```
example of a  `docker build`:

```bash
docker build \
  --tag "${REGISTRY}/${REGISTRY_PATH}/${IMAGE_NAME}:${IMAGE_VERSION}" \
  "${DOCKERFILE_DIR}"
```

## Testing
###  Running test_local.py

Execution example:

```bash
export PYTHONPATH=../../lambda_functions/tre-forward
export TRE_OUT_TOPIC_ARN="${tre_out_sns_arn}"

AWS_PROFILE="${aws_profile_deployment}" ./test_local.py \
  --aws_profile_management "${aws_profile_management}" \
  --environment_name "${environment_name}" \
  --test_consignment_s3_bucket "${test_data_bucket_name}" \
  --test_consignment_archive_s3_path "${test_consignment_archive_s3_path}" \
  --test_consignment_checksum_s3_path "${test_consignment_checksum_s3_path}" \
  --test_consignment_type "${consignment_type}" \
  --test_consignment_ref "${consignment_ref}" \
  --message_count "${message_count}"
```

The following arguments can also be used:

* `--empty_event` : sends an empty TRE event instead of a valid one
* `--omit_message_attributes` : does not forward SNS Message Attributes
