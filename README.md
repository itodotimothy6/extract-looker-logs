## Overview

A Python script that extracts [System Activity](https://docs.looker.com/admin-options/system-activity) data from the last 10 minutes, formats the data as Audit Logs, and exports the logs to Cloud Logging. The data formatting/mapping is best effort. See [data mapping](#gcp-audit-log-fields-to-looker-system-activity-mapping) below.

**_NOTE:_**  You can schedule this script to run every 10 minutes using a cron job or equivalent to continually create and export logs.

## Requirements
- Looker Instance in which you have Admin or `see_system_activity` permission
- Google Cloud Project with Cloud Logging API enabled
- [pyenv](https://github.com/pyenv/pyenv#installation) installed

## Local Deployment

- Create [Looker API credentials](https://docs.looker.com/reference/api-and-integration/api-auth) and set the below environment variables
  ```
  export LOOKERSDK_BASE_URL="<Your API URL>"
  export LOOKERSDK_CLIENT_ID="<Your Client ID>"
  export LOOKERSDK_CLIENT_SECRET="<Your Client Secret>"
  ```

- Create and configure a [service account](https://cloud.google.com/logging/docs/reference/libraries#setting_up_authentication) to write log entries to Cloud Logging and download the keys
  ```
  export GOOGLE_APPLICATION_CREDENTIALS="<Service Account Key Path>"
  ```

- Clone the repo
  ```
  git clone https://github.com/itodotimothy6/extract-looker-logs.git
  cd extract-looker-logs/
  ```
  
- Setup python virtual environment 
  ```
  pyenv install 3.8.2
  pyenv local 3.8.2
  python -m venv .venv
  source .venv/bin/activate
  ```

- Install dependencies 
  ```
  pip install looker-sdk
  pip install google-cloud-logging
  ```


- Run `main.py`
  ```
  python main.py
  ```


## Cloud Scheduler with Cloud Functions Deployment

These steps will use Cloud Functions to pull the Looker logs every 10 minutes using a Cloud Scheduler Job. 
Looker's API credentials are stored in Cloud Secret Manager

1. Enable APIs required:
   - Cloud Functions
   - Cloud Scheduler
   - Pub/Sub
   - Secret manager


2. Grant IAM permissions to the account used by Cloud functions. 
By default, Cloud Functions use App engine Default Service Account: <project-id>@appspot.gserviceaccount.com
Assign IAM roles:
   - Secret Manager Secret Accessor 
   - Logging Admin

3. Set helper env variables:
```
export PROJECT_ID=<GCP Project ID>
export REGION=<Region>
export LOCATION=<location> # Example: us-central1
export LOOKER_LOGS_TOPIC=<Topic name>
export LOOKERSDK_CLIENT_ID=<API Client ID>
export LOOKERSDK_CLIENT_SECRET=<API Client Secret>
export LOOKERSDK_BASE_URL=<Looker URL>
```

4. Create Pub/Sub topic
```
gcloud pubsub topics create ${LOOKER_LOGS_TOPIC}
```

5. Create Cloud Scheduler to run every 10 mins (Cron job)
```
gcloud scheduler jobs create pubsub Looker-collector-job --schedule="*/10 * * * *" --topic=${LOOKER_LOGS_TOPIC} --message-body="Pulling logs" --location=${LOCATION}
```

6. Store Looker's API credentials on Secrets manager

```
gcloud secrets create "looker_client_id" \
    --replication-policy "automatic" \
    --data-file - <<< ${LOOKERSDK_CLIENT_ID}

gcloud secrets create "looker_client_secret" \
    --replication-policy "automatic" \
    --data-file - <<< ${LOOKERSDK_CLIENT_SECRET}
```


7. Create Cloud function

- Clone this repository to a local path:

```
git clone https://github.com/elpeme/extract-looker-logs.git
cd extract-looker-logs
```
- Create the Cloud Function with source as the local path:

```
gcloud functions deploy looker_logs_collector \
--region=${REGION} \
--no-allow-unauthenticated \
--ingress-settings=internal-and-gclb \
--runtime=python39 \
--trigger-topic=${LOOKER_LOGS_TOPIC} \
--set-env-vars=LOOKERSDK_BASE_URL=https://cloudcenatt.cloud.looker.com \
--set-secrets="LOOKERSDK_CLIENT_ID=looker_client_id:latest" \
--set-secrets="LOOKERSDK_CLIENT_SECRET=looker_client_secret:latest" \
--source=. \
--entry-point=looker_collector_main
```





**_NOTE:_** Step 8 is optional. It is used to create the Cloud Function sourced from Google Source Repository mirrored from Github instead of using the local path

8. Create Cloud Function sourced from Google Source Repository

- Follow instructions to mirror repository:
  https://cloud.google.com/source-repositories/docs/mirroring-a-github-repository

- Create env variables:
```
export FUNCTION_REPOSITORY=github_elpeme_extract-looker-logs
export BRANCH=WIP-cloud-functions
export REPO_SOURCE=https://source.developers.google.com/projects/${PROJECT_ID}/repos/${FUNCTION_REPOSITORY}/moveable-aliases/${BRANCH}/paths//
```

- Create cloud function  
```
gcloud functions deploy looker_logs_collector \
--region=${REGION} \
--no-allow-unauthenticated \
--ingress-settings=internal-and-gclb \
--runtime=python39 \
--trigger-topic=${LOOKER_LOGS_TOPIC} \
--set-env-vars=LOOKERSDK_BASE_URL=https://cloudcenatt.cloud.looker.com \
--set-secrets="LOOKERSDK_CLIENT_ID=looker_client_id:latest" \
--set-secrets="LOOKERSDK_CLIENT_SECRET=looker_client_secret:latest" \
--entry-point=looker_collector_main \
--source=${REPO_SOURCE} \
```

## GCP Audit Log Fields to Looker System Activity Mapping

| GCP Audit Log Field       | Looker System Actvity Field or Value|
| -----------               | -----------                 |
| [logName](https://cloud.google.com/logging/docs/reference/v2/rest/v2/LogEntry#:~:text=Fields-,logName,-string) | `looker_system_activity_logs` |
| [timestamp](https://cloud.google.com/logging/docs/reference/v2/rest/v2/LogEntry#:~:text=reported%20the%20error.-,timestamp,-string) | [event.created](https://docs.looker.com/admin-options/tutorials/events#:~:text=for%20example%2C%20create_dashboard-,created,-Date%20and%20time) |
| [resource.type](https://cloud.google.com/logging/docs/reference/v2/rest/v2/MonitoredResource#:~:text=Fields-,type,-string)  | `looker_system_activity_logs`  |
| [insertId](https://cloud.google.com/logging/docs/reference/v2/rest/v2/LogEntry#:~:text=is%20LogSeverity.DEFAULT.-,insertid,-string)  | [event.id](https://docs.looker.com/admin-options/tutorials/events#:~:text=Description-,id,-Unique%20numeric%20identifier)  |
| `protoPayload.status` | [event.attribute.status](https://docs.looker.com/admin-options/tutorials/events#:~:text=Trigger-,Attributes,-add_external_email_to_scheduled_task) |
| `protoPayload.authenticationInfo`  | [event.user_id](https://docs.looker.com/admin-options/tutorials/events#:~:text=of%20the%20event-,user_id,-Unique%20numeric%20ID), [event.sudo_user_id](https://docs.looker.com/admin-options/tutorials/events#:~:text=for%20example%2C%20dashboard-,sudo_user_id,-Unique%20numeric%20ID)  |
| `protoPayload.authorizationInfo`  | `permission_set.permissions`  |
| `protoPayload.methodName`  | [event.name](https://docs.looker.com/admin-options/tutorials/events#:~:text=triggered%20the%20event-,name,-Name%20of%20the) |
| `protoPayload.response` | [event_attributes](https://docs.looker.com/admin-options/tutorials/events#:~:text=Trigger-,Attributes,-add_external_email_to_scheduled_task) |
