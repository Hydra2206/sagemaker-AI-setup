# sagemaker-AI-setup
Setting up SM AI at production level, domains, userprofiles, RBAC, ABAC, etc

####STEPS TO SETUP SM-AI, DOMAINS, USERPROFILES, ABAC, RBAC, ROLES, IAM USER...

export REGION=ap-south-1
aws ec2 describe-vpcs   --filters "Name=isDefault,Values=true"   --query "Vpcs[0].VpcId"   --output text   --region ${REGION}
aws ec2 describe-subnets   --filters "Name=vpc-id,Values=vpc-0ab89050739097343"   --query "Subnets[].SubnetId"   --output text   --region ${REGION}
vim trust.json
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Principal": { "Service": "sagemaker.amazonaws.com" },
          "Action": "sts:AssumeRole"
        }
      ]
    }
aws iam create-role   --role-name SageMakerDomainExecutionRole   --assume-role-policy-document file://trust.json
aws iam attach-role-policy   --role-name SageMakerDomainExecutionRole   --policy-arn arn:aws:iam::aws:policy/AmazonSageMakerFullAccess
aws sagemaker create-domain   --domain-name my-sagemaker-domain   --auth-mode IAM   --vpc-id <DEFAULT_VPC_ID>   --subnet-ids <SUBNET1> <SUBNET2>   --app-network-access-type VpcOnly   --default-user-settings "{\"ExecutionRole\": \"<ROLE_ARN>\"}"   --region ${REGION}
aws sagemaker create-user-profile   --domain-id <DOMAIN_ID>   --user-profile-name alice-profile   --tags Key=studiouserid,Value=alice123   --region ${REGION}
aws iam create-user --user-name alice-iam-user
aws iam tag-user   --user-name alice-iam-user   --tags Key=studiouserid,Value=alice123
vim sagemaker-abac.json
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Sid": "AllowConsoleListAndDescribe",
          "Effect": "Allow",
          "Action": [
            "sagemaker:ListDomains",
            "sagemaker:ListUserProfiles",
            "sagemaker:ListApps",
            "sagemaker:DescribeDomain",
            "sagemaker:DescribeUserProfile",
            "sagemaker:ListTags"
          ],
          "Resource": "*"
        },
        {
          "Sid": "AllowPresignedUrlWhenTagMatches",
          "Effect": "Allow",
          "Action": [
            "sagemaker:CreatePresignedDomainUrl"
          ],
          "Resource": "*",
          "Condition": {
            "StringEquals": {
              "sagemaker:ResourceTag/studiouserid": "${aws:PrincipalTag/studiouserid}"
            }
          }
        }
      ]
    }
aws iam create-policy   --policy-name SageMaker-Studio-ABAC   --policy-document file://sagemaker-abac.json
aws iam attach-user-policy   --user-name alice-iam-user   --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/SageMaker-Studio-ABAC


