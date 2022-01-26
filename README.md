# save-stack-outputs

Save CloudFormation stack outputs to job-level env vars for use by other steps. Inspired by [deep-mm/set-variables](https://github.com/deep-mm/set-variables).

This is designed for common use cases such as passing variables from a frontend stack to a backend stack and vice versa, or passing stack output values to a set of integration tests.

## Example usage

In this case, some components of my-app need to share information that is not known until the respective stack has been deployed.

1. The backend needs to know the URL where the frontend is deployed, so that it can configure things like CORS and OAuth callback URLs accordingly.
1. When building the frontend website, it needs to know some things from the backend stack such as the API URL(s), IDP config (e.g. Auth/Token URL, User Pool ID, etc.)

```yaml

- name: Deploy frontend infra
  working-directory: frontend
  run: |
      sam deploy --stack-name my-app-frontend-infra \
          --s3-bucket $ARTIFACT_BUCKET \
          --no-fail-on-empty-changeset \
          --no-confirm-changeset \
          --capabilities CAPABILITY_IAM \

##### Use the save-stack-outputs action here. #####
- name: Save frontend stack outputs
  uses: puremcc/save-stack-outputs@v1
  with:
    stack-name: my-app-frontend-infra

## Github Workflow Output:
#
# Saving outputs from stack 'my-app-frontend-infrae' to the following env vars:
#   BucketName: my-app-website-bucket
#   WebsiteUrl: https://abcd1234.cloudfront.net
#   DistributionId: ABCDE12345

# Now we can use $WebsiteUrl for the 'AppHostUrl' parameter override when 
# deploying our backend stack.

- name: Deploy backend
  working-directory: backend
  run: |
    sam deploy --stack-name my-app-backend-infra \
        --s3-bucket $ARTIFACT_BUCKET \
        --no-fail-on-empty-changeset \
        --no-confirm-changeset \
        --capabilities CAPABILITY_IAM \
        --parameter-overrides "AppHostUrl=$WebsiteUrl"

##### Use the save-stack-outputs action here. #####
- name: Save backend stack outputs
  uses: puremcc/save-stack-outputs@v1
  with:
    stack-name: my-app-backend-infra

## Github Workflow Output:
#
# Saving outputs from stack 'my-app-backend-infra' to the following env vars:
#   UserPoolId: us-east-1_aBcD1234
#   UserPoolAuthDomain: my-app-backend-infra.auth.us-east-1.amazoncognito.com
#   UserPoolWebClientId: 8ja2f1lhj36db3ima7qpuvdlp
#   UserPoolRegion: us-east-1
#   ApiUrl: https://1234abcd5678.execute-api.us-east-1.amazonaws.com

# Frontend JS compiler can now use the environment variables that were set from 
# the stack outputs in the previous step.
- name: Build frontend for Dev
  working-directory: frontend
  run: npm run build
```
