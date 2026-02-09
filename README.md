LambdaTest HyperExecute Trigger via GitLab CI

This project demonstrates how to trigger a LambdaTest Test Manager run using the HyperExecute API from a GitLab CI/CD pipeline.

Prerequisites

Before setting up the pipeline, make sure you have:

A GitLab project

A valid LambdaTest Test Manager Test Run ID

LambdaTest Basic Auth (Base64 encoded) credentials

Step 1: Create a GitLab Project

Go to GitLab → New Project

Select Create blank project

Enter a project name

Click Create project

Step 2: Add CI/CD Variables

Go to:

Project → Settings → CI/CD → Variables → Add variable

Add the following:

Required variables
Key	Value	Notes
LT_TM_BASE64_AUTH	Your Base64 auth string	Masked: Yes
TEST_RUN_ID	Your LambdaTest test run id	Required
Optional variables
Key	Default	Description
LT_REGION	centralindia	Web test region
LT_MOBILE_REGION	ap	Mobile test region
Step 3: Create .gitlab-ci.yml

Create a file in the root of the repository named:

.gitlab-ci.yml


Paste the following configuration:
stages:
  - trigger

trigger_lambdatest_hyperexecute:
  stage: trigger
  image: alpine:3.20
  before_script:
    - apk add --no-cache curl jq
  script:
    # 1) Build JSON payload (VALID JSON - no comments)
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

    # 2) Call LambdaTest API and capture response + status code
    - |
      echo "Triggering LambdaTest HyperExecute run..."
      http_code=$(curl -sS -o response.json -w "%{http_code}" \
        --location "https://test-manager-api.lambdatest.com/api/atm/v1/hyperexecute" \
        -H "Content-Type: application/json" \
        -H "Authorization: Basic ${LT_TM_BASE64_AUTH}" \
        --data @payload.json)

      echo "HTTP Status: $http_code"
      cat response.json | jq .

      # 3) Fail pipeline if trigger API didn't succeed
      if [ "$http_code" -lt 200 ] || [ "$http_code" -ge 300 ]; then
        echo "ERROR: Failed to trigger LambdaTest run"
        exit 1
      fi

    # 4) Extract an id (best-effort; depends on API response fields)
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

  # Optional: make it manual so you click-to-run
  when: manual

  Commit the file to the repository.

Step 4: Run the Pipeline

Go to CI/CD → Pipelines

Click Run pipeline

Select the branch

Click Run pipeline

Step 5: Verify Execution

Open the running pipeline

Click the job: trigger_lambdatest_hyperexecute

Check the logs:

HTTP status should be 200 or 201

Response JSON will be printed

Step 6: Download Artifacts (Optional)

From the job page, download:

payload.json → request sent to API

response.json → API response

Notes

Do not store credentials in the repository.

Always use GitLab CI/CD variables for authentication.

The pipeline currently triggers the run only.

You can extend it to poll for completion using a status API.
