# LambdaTest HyperExecute Trigger via GitLab CI

This project shows how to trigger a LambdaTest Test Manager run using the HyperExecute API from a GitLab CI/CD pipeline.

---

## Prerequisites

Before starting, make sure you have:

- A GitLab project
- A LambdaTest Test Manager Test Run ID
- LambdaTest Basic Auth credentials (Base64 encoded)

---

## Step 1: Create a GitLab Project

1. Go to GitLab.
2. Click **New Project**.
3. Select **Create blank project**.
4. Enter a project name.
5. Click **Create project**.

---

## Step 2: Add CI/CD Variables

Go to:

Project → Settings → CI/CD → Variables → Add variable

Add the following variables:

### Required

Variable name:
LT_TM_BASE64_AUTH  
Value:
Your Base64 auth string  
(Mark as Masked)

Variable name:
TEST_RUN_ID  
Value:
Your LambdaTest test run ID

### Optional

Variable name:
LT_REGION  
Value:
centralindia

Variable name:
LT_MOBILE_REGION  
Value:
ap

---

## Step 3: Create the GitLab pipeline file

In the root of your repository, create a file named:

.gitlab-ci.yml

Paste the following content:

```yaml
stages:
  - trigger

trigger_lambdatest_hyperexecute:
  stage: trigger
  image: alpine:3.20
  before_script:
    - apk add --no-cache curl jq
  script:
    - |
      jq -n \
        --arg test_run_id "$TEST_RUN_ID" \
        --arg title "${CI_PROJECT_NAME}-${CI_COMMIT_REF_NAME}-${CI_PIPELINE_ID}" \
        --arg region "${LT_REGION:-centralindia}" \
        --arg mobile_region "${LT_MOBILE_REGION:-ap}" \
        '{
          test_run_id: $test_run_id,
          concurrency: 1,
          title: $title,
          console_log: false,
          network_logs: false,
          network_full_har: false,
          region: $region,
          mobile_region: $mobile_region,
          retry_on_failure: true,
          max_retries: 1
        }' > payload.json

    - |
      echo "Triggering LambdaTest HyperExecute run..."
      http_code=$(curl -sS -o response.json -w "%{http_code}" \
        --location "https://test-manager-api.lambdatest.com/api/atm/v1/hyperexecute" \
        -H "Content-Type: application/json" \
        -H "Authorization: Basic ${LT_TM_BASE64_AUTH}" \
        --data @payload.json)

      echo "HTTP Status: $http_code"
      cat response.json | jq .

      if [ "$http_code" -lt 200 ] || [ "$http_code" -ge 300 ]; then
        echo "ERROR: Failed to trigger LambdaTest run"
        exit 1
      fi

    - |
      RUN_ID=$(jq -r '.run_id // .id // .build_id // empty' response.json)
      echo "RUN_ID=$RUN_ID" | tee lt.env
      echo "Triggered RUN_ID=$RUN_ID"

  artifacts:
    when: always
    paths:
      - payload.json
      - response.json
    reports:
      dotenv: lt.env
    expire_in: 7 days
```

Commit this file

---

## Step 4: Run the pipeline
- Go to CI/CD → Pipelines
- Click Run pipeline
- Select your branch
- Click Run pipeline

---

## Step 5: Verify the run

- Open the pipeline
- Click the job: trigger_lambdatest_hyperexecute
- Check the logs

---

## Step 6: Download artifacts (optional)

From the job page, download:
- payload.json
- response.json

These contain the request and API response.

---

## Notes
- Do not store credentials in the repository.
- Always use GitLab CI/CD variables for authentication.
- This pipeline only triggers the run.
- You can extend it to wait for test completion using a status API.
