# GoogleCloud SecurityCommandCenter to AzureSentinel Integration (Sweet Connector)

This connector allows you to send security alerts from Google Cloud Security Command Center to Microsoft Azure Sentinel Log Analytics Workspace in almost realtime.
If you have created a better version of this integration, please do contribute by creating a pull request!
If you have any feedback, or need any help in setting this up, please reach out to amiacs@gmail.com.

### Architecture Flow

![Alt text](architecture.png?raw=true "Integration Architecture")

In the above diagram:
1. SCC streams security alerts to a PubSub queue
2. A Cloud function is setup that subscribes to the PubSub queue and gets triggered by EventArc
3. The Cloud Function calls the Data Collection API of Azure Sentinel with HMAC-SHA256 authorization
4. On the Azure Sentinel side, the security alert is routed to a custom table in the Log Analytics Workspace

End-to-end latency from when an alert is triggered to when it appears in Sentinel is within a couple of seconds. It does take some time (10-15 minutes) for the first SCC alert to show up in Sentinel, as the Sentinel table gets initialized.

### Step-by-Step Setup Instructions

1. Create a continuous pubsub export of SCC Alerts
2. Set up Azure Sentinel 
   - Create a Log Analytics Workspace
3. Create a Cloud Function in Google Cloud
   - Download python source code from this Github repository (you don’t need to modify the code)
   - Create .env file and provide credentials OR put the credentials in GCP secrets manager
   - Setup EventArc trigger and deploy the function
4. Trigger an SCC Alert and run a query on the Sentinel Log Table to view the finding

#### Create a continuous pubsub export of SCC Alerts
1. Go to GCP Console -> Security Command Center -> Settings -> Continuous Exports
2. Create a new PubSub Export as shown in the screenshot
3. You could also use a query to filter out only certain events (e.g. critical/high severity) that would be exported to the PubSub 

![Alt text](screenshots/1.png?raw=true "SCC PubSub Export")

#### Setting up Azure Sentinel Log Analytics Workspace 1/2
1. Go to Azure Console -> Log Analytics Workspaces -> Create
2. Create a new Workspace or use an existing one
3. After creation, the Workspace would look like as in the screenshot
4. Take note of the Workspace ID

![Alt text](screenshots/2.png?raw=true "Log Analytics Workspace")

#### Setting up Azure Sentinel Log Analytics Workspace 2/2
1. Go to Azure Console -> Log Analytics Workspaces -> Agents Management
2. Expand Log analytics agents instructions
3. Take a note of the Primary or Secondary key (either of them). This will be required to construct the SHA256-HMAC authorization header to call the Sentinel data collection API

![Alt text](screenshots/4.png?raw=true "Agents config")

#### Create a Cloud Function in Google Cloud 1/2
1. Go to GCP Console -> Cloud Functions -> Create Function
2. Select 2nd Gen Environment
3. Add Eventarc Trigger with Cloud Pub/Sub as the Event Provider and the scc-pubsub topic that you created earlier
4. Click Next

![Alt text](screenshots/5.png?raw=true "Cloud Function")

#### Create a Cloud Function in Google Cloud 2/2
1. Download the code from Github and put in in Cloud Function
2. Select Python3 as the Runtime
3. Create .env file with the following credentials that you noted from Azure earlier
4. Set the entry point as entry_point_function

**Please note that the Log Analytics custom table is created automatically in Azure, if it does not exist. This is the recommended way to deploy this connector.**

Also, as a best practice, you can save the Log Analytics Workspace ID and Key in GCP Secret Manager. This connector first tries to read the .env file, and if it does not find the values, it will try GCP Secret Manager using the PROJECT_ID mentioned in the .env file.
```
AZURE_LOG_ANALTYTICS_WORKSPACE_ID=YOUR_LOG_ANALYTICS_WORKSPACE_ID
AZURE_LOG_ANALYTICS_AUTHENTICATION_KEY=YOUR_PRIMARY_OR_SECONDARY_CLIENT_AUTHENTICATION_KEY
AZURE_LOG_ANALYTICS_CUSTOM_TABLE=YOUR_CUSTOM_LOG_TABLE_NAME

PROJECT_ID=YOUR_GCP_PROJECT_ID
```
5. Deploy the function

![Alt text](screenshots/6.png?raw=true "Cloud Function")

#### View SCC Alerts in Azure Sentinel
1. Trigger an SCC Alert in GCP Console (e.g. by opening a firewall port)
2. Go to Azure -> Log Analytics Workspace -> Logs
3. Create a New Query and write the custom table name appended by ‘_CL’ and hit run
4. You will see the SCC findings listed as in the screenshot

![Alt text](screenshots/7.png?raw=true "SCC Alerts in Sentinel")

### Conclusion
Congrats on successfully deploying this connector! If you have any feedback, or need any help in setting this up, please reach out to amiacs@gmail.com.
