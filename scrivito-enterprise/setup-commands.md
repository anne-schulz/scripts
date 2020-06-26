
```
profile='hhn'
account_id=`aws account-id --profile ${profile}`
site_name='hhn-web'
region='eu-central-1'
stg='-staging'

```

# CloudFront

```
aws cloudfront create-cloud-front-origin-access-identity --profile ${profile} --cloud-front-origin-access-identity-config CallerReference=`date +%s%H%M`,Comment=access-identity-${site_name}-staging

access_id_stg=`aws cloudfront list-cloud-front-origin-access-identities --profile ${profile} --query "CloudFrontOriginAccessIdentityList.Items[?Comment == 'access-identity-${site_name}-staging'][Id]" --output text`

aws cloudfront create-cloud-front-origin-access-identity --profile ${profile} --cloud-front-origin-access-identity-config CallerReference=`date +%s%S%M`,Comment=access-identity-${site_name}

access_id=`aws cloudfront list-cloud-front-origin-access-identities --profile ${profile} --query "CloudFrontOriginAccessIdentityList.Items[?Comment == 'access-identity-${site_name}'][Id]" --output text`
```

# S3 Bucket

## Create Buckets

```
aws s3api create-bucket --profile ${profile} --region ${region} --bucket ${site_name}${stg} --create-bucket-configuration LocationConstraint=${region}

aws s3api create-bucket --profile ${profile} --region ${region} --bucket ${site_name} --create-bucket-configuration LocationConstraint=${region}
```

## Bucket Policy

```
policy_stg="{
  \"Version\":\"2012-10-17\",
  \"Id\":\"access-policy\",
  \"Statement\":[
    {
      \"Sid\":\"Grant access-identity-${site_name}-staging for cdn access to ${site_name}${stg}\",
      \"Effect\":\"Allow\",
      \"Principal\":{
        \"AWS\":\"arn:aws:iam::cloudfront:user/CloudFront Origin Access Identity ${access_id_stg}\"
      },
      \"Action\":\"s3:GetObject\",
      \"Resource\":\"arn:aws:s3:::${site_name}${stg}/*\"
    }
  ]
}"
aws s3api put-bucket-policy  --profile ${profile} --region ${region} --bucket ${site_name}${stg} --policy "${policy_stg}"

policy="{
  \"Version\":\"2012-10-17\",
  \"Id\":\"access-policy\",
  \"Statement\":[
    {
      \"Sid\":\"Grant access-identity-${site_name} for cdn access to ${site_name}\",
      \"Effect\":\"Allow\",
      \"Principal\":{
        \"AWS\":\"arn:aws:iam::cloudfront:user/CloudFront Origin Access Identity ${access_id}\"
      },
      \"Action\":\"s3:GetObject\",
      \"Resource\":\"arn:aws:s3:::${site_name}/*\"
    }
  ]
}"
aws s3api put-bucket-policy  --profile ${profile} --region ${region} --bucket ${site_name} --policy "${policy}"
```

## Block public access

```
aws s3api put-public-access-block --profile ${profile} --region ${region} --bucket ${site_name}${stg} --public-access-block-configuration '{
  "BlockPublicAcls": true,
  "IgnorePublicAcls": true,
  "BlockPublicPolicy": true,
  "RestrictPublicBuckets": true
}'

aws s3api put-public-access-block --profile ${profile} --region ${region} --bucket ${site_name} --public-access-block-configuration '{
  "BlockPublicAcls": true,
  "IgnorePublicAcls": true,
  "BlockPublicPolicy": true,
  "RestrictPublicBuckets": true
}'
```

## Activate bucket versioning

```
aws s3api put-bucket-versioning --profile ${profile} --region ${region} --bucket ${site_name}${stg}  --versioning-configuration Status=Enabled

aws s3api put-bucket-versioning --profile ${profile} --region ${region} --bucket ${site_name}  --versioning-configuration Status=Enabled
```

## Enable encryption

```
aws s3api put-bucket-encryption --profile ${profile} --region ${region} --bucket ${site_name}$stg} --server-side-encryption-configuration '{"Rules": [{ "ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "AES256"} }] }'

aws s3api put-bucket-encryption --profile ${profile} --region ${region} --bucket ${site_name} --server-side-encryption-configuration '{"Rules": [{ "ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "AES256"} }] }'
```

# Lambda

create lambda functions via UI

A. in us-east-1
1. ${profile}-viewer-request
2. ${site_name}-staging-origin-response
3. ${site_name}-origin-response

B. in ${region}
1. ${profile}-deployment-authorizer
2. ${site_name}-staging-trigger-codepipeline
3. ${site_name}-trigger-codepipeline

## Update function configuration

```
for function_name in ${profile}-viewer-request ${site_name}${stg}-origin-response ${site_name}-origin-response
do
  aws lambda update-function-configuration --profile ${profile} --region us-east-1 --function-name ${function_name} --timeout 5
done
```

## Tag functions

```
aws lambda tag-resource --profile ${profile} --region ${region} --resource arn:aws:lambda:${region}:${account_id}:function:${site_name}${stg}-trigger-codepipeline --tags method=POST,resource=/${site_name}-staging,gateway=${profile}-deployment

aws lambda tag-resource --profile ${profile} --region ${region}  --resource arn:aws:lambda:${region}:${account_id}:function:${site_name}-trigger-codepipeline --tags method=POST,resource=/${site_name},gateway=${profile}-deployment
```

# ACM

```
aws acm request-certificate --profile ${profile} --validation-method DNS --region us-east-1 --domain-name *.infopark.io

certificate_arn=`acm list-certificates --profile ${profile} --region us-east-1 --query "CertificateSummaryList[?DomainName == '*.infopark.io'].CertificateArn" --output text`
```

# API Gateway

```
profile='hhn' account_id=`aws account-id --profile ${profile}` site_name='hhn-web' region='eu-central-1' stg='-staging' envsubst < deployment-apigw-swagger.json > ${profile}-deployment-apigateway.json
```

create API gateway via UI by importing the file

```
gateway_id=`aws apigateway get-rest-apis --profile ${profile} --region eu-central-1 --query "items[?name == '{profile}-deployment'].id" --output text`
```

## Grant permissions

so that the API endpoints are allowed to invoke the lambda functions they reference

```
aws lambda add-permission --profile ${profile} --region ${region} --function-name "arn:aws:lambda:${region}:${account_id}:function:${site_name}${stg}-trigger-codepipeline" --source-arn "arn:aws:execute-api:${region}:${account_id}:${gateway_id}/*/POST/${site_name}-staging" --principal apigateway.amazonaws.com --statement-id 1f7f9cd6-47ac-4b89-927a-edb9db591da0 --action lambda:InvokeFunction

aws lambda add-permission --profile ${profile} --region ${region} --function-name "arn:aws:lambda:${region}:${account_id}:function:${site_name}-trigger-codepipeline" --source-arn "arn:aws:execute-api:${region}:${account_id}:${gateway_id}/*/POST/${site_name}" --principal apigateway.amazonaws.com --statement-id 1f7f9cd6-47ac-4b89-927a-edb9db591da2 --action lambda:InvokeFunction

authorizer_id=`aws apigateway get-authorizers --profile ${profile} --region ${region} --rest-api-id ${gateway_id} --query "items[?name == '${profile}-deployment-authorizer'].id" --output text`
aws lambda add-permission --profile ${profile} --region ${region} --function-name "arn:aws:lambda:${region}:${account_id}:function:${profile}-deployment-authorizer" --source-arn "arn:aws:execute-api:${region}:${account_id}:${gateway_id}/authorizers/${authorizer_id}" --principal apigateway.amazonaws.com --statement-id 1f7f9cd6-47ac-4b89-927a-edb9db591da6 --action lambda:InvokeFunction
```

# CloudFront - create distributions for staging and production with a template (each)

```
profile='hhn' account_id=`aws account-id --profile ${profile}` site_name='hhn-web' region='eu-central-1' stg='-staging' access_id_stg=`aws cloudfront list-cloud-front-origin-access-identities --profile ${profile} --query "CloudFrontOriginAccessIdentityList.Items[?Comment == 'access-identity-${site_name}-staging'][Id]" --output text` certificate_arn=`aws acm list-certificates --profile ${profile} --region us-east-1 --query "CertificateSummaryList[?DomainName == '*.infopark.io'].CertificateArn" --output text` envsubst < cloudfront-distribution-config-staging.json > ${site_name}-cfdc-stg.json
aws cloudfront create-distribution --profile ${profile} --distribution-config file://${site_name}-cfdc-stg.json

profile='hhn' account_id=`aws account-id --profile ${profile}` site_name='hhn-web' region='eu-central-1' stg='-staging' access_id=`aws cloudfront list-cloud-front-origin-access-identities --profile ${profile} --query "CloudFrontOriginAccessIdentityList.Items[?Comment == 'access-identity-${site_name}'][Id]" --output text` certificate_arn=`aws acm list-certificates --profile ${profile} --region us-east-1 --query "CertificateSummaryList[?DomainName == '*.infopark.io'].CertificateArn" --output text` envsubst <cloudfront-distribution-config.json > ${site_name}-cfdc.json
aws cloudfront create-distribution --profile ${profile} --distribution-config file://${site_name}-cfdc.json
```

# IAM

```
aws iam create-role --profile ${profile} --role-name AWSCodePipelineServiceRole --path /service-role/ --assume-role-policy-document file://iam/codepipeline-trust-relationship.json

aws iam create-role --profile ${profile} --role-name codebuild-${site_name}-service-role --assume-role-policy-document file://iam/codebuild-trust-relationship.json

profile='hhn' account_id=`aws account-id --profile ${profile}` site_name='hhn-web' region='eu-central-1' envsubst < iam/codebuild-BasePolicy.json >iam/${profile}-codebuild-Basepolicy.json

profile='hhn' account_id=`aws account-id --profile ${profile}` site_name='hhn-web' region='eu-central-1' envsubst < iam/codebuild-S3-ReadWriteBucket.json >iam/${profile}-codebuild-S3-ReadWriteBucket.json

profile='hhn' account_id=`aws account-id --profile ${profile}` site_name='hhn-web' region='eu-central-1' envsubst < iam/codebuild-SSM-KMS-GetParameters.json >iam/${profile}-codebuild-SSM-KMS-GetParameters.json

profile='hhn' account_id=`aws account-id --profile ${profile}` site_name='hhn-web' region='eu-central-1' envsubst < iam/codebuild-Lambda-UpdateRelatedFunctions.json >iam/${profile}-codebuild-Lambda-UpdateRelatedFunctions.json

profile='hhn' account_id=`aws account-id --profile ${profile}` site_name='hhn-web' region='eu-central-1' cfd_id=`aws cloudfront list-distributions --profile ${profile} --query "DistributionList.Items[?Comment=='${site_name}'].Id" --output text` cfd_id_stg=`aws cloudfront list-distributions --profile ${profile} --query "DistributionList.Items[?Comment=='${site_name}-staging'].Id" --output text` envsubst < iam/codebuild-CloudFront-Update-Invalidate.json >iam/${profile}-codebuild-CloudFront-Update-Invalidate.json




```

# CodeBuild

```
aws codebuild create-project --profile ${profile} --region ${region} --name ${site_name}-Build --source type=CODEPIPELINE,buildspec=codePipeline/buildspec.yml,insecureSsl=false --artifacts type=CODEPIPELINE,name=${site_name}-Build,packaging=NONE,encryptionDisabled=false --environment type=LINUX_CONTAINER,image=aws/codebuild/amazonlinux2-x86_64-standard:3.0,computeType=BUILD_GENERAL1_SMALL,environmentVariables=[],privilegedMode=false,imagePullCredentialsType=CODEBUILD --service-role arn:aws:iam::${account_id}:role/codebuild-${site_name}-service-role --description "Build"

aws codebuild create-project --profile ${profile} --region ${region} --name ${site_name}-Test-EsLint --source type=CODEPIPELINE,buildspec=codePipeline/testspec-eslint.yml,insecureSsl=false --artifacts type=CODEPIPELINE,name=${site_name}-Test-EsLint,packaging=NONE,encryptionDisabled=false --environment type=LINUX_CONTAINER,image=aws/codebuild/amazonlinux2-x86_64-standard:3.0,computeType=BUILD_GENERAL1_SMALL,environmentVariables=[],privilegedMode=false,imagePullCredentialsType=CODEBUILD --service-role arn:aws:iam::${account_id}:role/codebuild-${site_name}-service-role --description "Test EsLint"

aws codebuild create-project --profile ${profile} --region ${region} --name ${site_name}-Test-EsCheck --source type=CODEPIPELINE,buildspec=codePipeline/testspec-escheck.yml,insecureSsl=false --artifacts type=CODEPIPELINE,name=${site_name}-Test-EsCheck,packaging=NONE,encryptionDisabled=false --environment type=LINUX_CONTAINER,image=aws/codebuild/amazonlinux2-x86_64-standard:3.0,computeType=BUILD_GENERAL1_SMALL,environmentVariables=[],privilegedMode=false,imagePullCredentialsType=CODEBUILD --service-role arn:aws:iam::${account_id}:role/codebuild-${site_name}-service-role --description "Test EsCheck"

aws codebuild create-project --profile ${profile} --region ${region} --name ${site_name}-Test-Unit --source type=CODEPIPELINE,buildspec=codePipeline/testspec-unit-tests.yml,insecureSsl=false --artifacts type=CODEPIPELINE,name=${site_name}-Test-Unit,packaging=NONE,encryptionDisabled=false --environment type=LINUX_CONTAINER,image=aws/codebuild/amazonlinux2-x86_64-standard:3.0,computeType=BUILD_GENERAL1_SMALL,environmentVariables=[],privilegedMode=false,imagePullCredentialsType=CODEBUILD --service-role arn:aws:iam::${account_id}:role/codebuild-${site_name}-service-role --description "Test Unit"

aws codebuild create-project --profile ${profile} --region ${region} --name ${site_name}-Deploy --source type=CODEPIPELINE,buildspec=codePipeline/deployspec.yml,insecureSsl=false --artifacts type=CODEPIPELINE,name=${site_name}-Deploy,packaging=NONE,encryptionDisabled=false --environment type=LINUX_CONTAINER,image=aws/codebuild/amazonlinux2-x86_64-standard:3.0,computeType=BUILD_GENERAL1_SMALL,environmentVariables=[],privilegedMode=false,imagePullCredentialsType=CODEBUILD --service-role arn:aws:iam::${account_id}:role/codebuild-${site_name}-service-role --description "Deploy"

aws codebuild create-project --profile ${profile} --region ${region} --name ${site_name}-staging-Build --source type=CODEPIPELINE,buildspec=codePipeline/buildspec.yml,insecureSsl=false --artifacts type=CODEPIPELINE,name=${site_name}-staging-Build,packaging=NONE,encryptionDisabled=false --environment type=LINUX_CONTAINER,image=aws/codebuild/amazonlinux2-x86_64-standard:3.0,computeType=BUILD_GENERAL1_SMALL,environmentVariables=[],privilegedMode=false,imagePullCredentialsType=CODEBUILD --service-role arn:aws:iam::${account_id}:role/codebuild-${site_name}-service-role --description "Build staging"

aws codebuild create-project --profile ${profile} --region ${region} --name ${site_name}-staging-Test-EsLint --source type=CODEPIPELINE,buildspec=codePipeline/testspec-eslint.yml,insecureSsl=false --artifacts type=CODEPIPELINE,name=${site_name}-staging-Test-EsLint,packaging=NONE,encryptionDisabled=false --environment type=LINUX_CONTAINER,image=aws/codebuild/amazonlinux2-x86_64-standard:3.0,computeType=BUILD_GENERAL1_SMALL,environmentVariables=[],privilegedMode=false,imagePullCredentialsType=CODEBUILD --service-role arn:aws:iam::${account_id}:role/codebuild-${site_name}-service-role --description "Test EsLint staging"

aws codebuild create-project --profile ${profile} --region ${region} --name ${site_name}-staging-Test-EsCheck --source type=CODEPIPELINE,buildspec=codePipeline/testspec-escheck.yml,insecureSsl=false --artifacts type=CODEPIPELINE,name=${site_name}-staging-Test-EsCheck,packaging=NONE,encryptionDisabled=false --environment type=LINUX_CONTAINER,image=aws/codebuild/amazonlinux2-x86_64-standard:3.0,computeType=BUILD_GENERAL1_SMALL,environmentVariables=[],privilegedMode=false,imagePullCredentialsType=CODEBUILD --service-role arn:aws:iam::${account_id}:role/codebuild-${site_name}-service-role --description "Test EsCheck staging"

aws codebuild create-project --profile ${profile} --region ${region} --name ${site_name}-staging-Test-Unit --source type=CODEPIPELINE,buildspec=codePipeline/testspec-unit-tests.yml,insecureSsl=false --artifacts type=CODEPIPELINE,name=${site_name}-staging-Test-Unit,packaging=NONE,encryptionDisabled=false --environment type=LINUX_CONTAINER,image=aws/codebuild/amazonlinux2-x86_64-standard:3.0,computeType=BUILD_GENERAL1_SMALL,environmentVariables=[],privilegedMode=false,imagePullCredentialsType=CODEBUILD --service-role arn:aws:iam::${account_id}:role/codebuild-${site_name}-service-role --description "Test Unit staging"

aws codebuild create-project --profile ${profile} --region ${region} --name ${site_name}-staging-Deploy --source type=CODEPIPELINE,buildspec=codePipeline/deployspec.yml,insecureSsl=false --artifacts type=CODEPIPELINE,name=${site_name}-staging-Deploy,packaging=NONE,encryptionDisabled=false --environment type=LINUX_CONTAINER,image=aws/codebuild/amazonlinux2-x86_64-standard:3.0,computeType=BUILD_GENERAL1_SMALL,environmentVariables=[],privilegedMode=false,imagePullCredentialsType=CODEBUILD --service-role arn:aws:iam::${account_id}:role/codebuild-${site_name}-service-role --description "Deploy staging"

```
