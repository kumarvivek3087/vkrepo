# Azure Cost anomaly detection 

This repo was forked from the [CSAAppServiceDeployments](https://github.com/microsoft/CSAAppServiceDeployments) repo. Initial idea and credits goes to @bwatts64.

## Table of Content  

* [Function App to Ingest Azure cost data into Log Analytics](#CostLA)
* [Deploy to Azure](#Deployment)
* [Post deployment steps (!important)](#PostDeployment)

# <a name="CostLA"></a>Function App to Ingest Azure cost data into Log Analytics  
Many Azure users (including myself) would like to more easily see Cost Anamolies that occur in their Azure environment. Currently, that functionality isn't available in Azure Cost Management (ACM) but if we were to ingest this data into Log Analytics, we could leverage the built-in anomaly detection in the product. The deployment below will create a Function app to ingest Azure cost management data into Log Analytics each day.

The architecture is made of the following resources:
- Azure Function App w/ System managed identity
- Azure Log Analytics
- Azure Application Insights
- Azure Storage Account
- Azure Monitor Workbook

<img src="./images/AzureCostAnomalies.jpg" alt="Environment"  Width="600">  

## <a name="Deployment"></a>Deploy to Azure  
**Make sure to read the post deployment steps!**

Bicep in used for the infrastructure as code. Any recent version of Azure PowerShell or Azure CLI can deploy a bicep file.

Be sure to be running latest version of [Bicep](https://github.com/Azure/bicep/releases/latest)

Here is an invocation example: 

```
az deployment group create -g poc-costanomalydetection -f .\Templates\main.bicep --parameters scopes=providers/Microsoft.Billing/billingAccounts/cf9d8426-6a39-43c3-9709-128a891ad2d5:5a0b4b8f-5fa6-49fc-8f4e-7743746d8f04_2019-05-31
```

###  <a name="PostDeployment"></a>Post Deployment Steps    
After deploying this solution you have to give the App Service system assigned managed identity "Billing account reader" or "Cost Management Reader" role at the scope or scopes you are quering. _The system assigned managed identity will have the same name as your function app._

If you want to load historical data into Log Analytics you can utilize the function named **PreLoadLogAnalytics**.  

1) Get the function url by opening the Function app and clicking on **Get Function URL**. Note that it may take a little bit to light up.  
2) Use a tool like PostMan or curl to send the request. Below is an example using curl.

**curl 'https://poccostingestionfadhr2puxbmgliy.azurewebsites.net/api/PreLoadLogAnalytics?code=ichangedthisstring'**

### Deploying Solution
Use the below link to deploy this solution to Azure. ***Note*** Make sure you follow the **Post Deployment Steps** after completing the deployment.

**Note** There are many parameters that you can supply, only one is required, `scopes`:  
1) deploymentPrefix: this will prefix the name of all the resources created.  
2) scopes: this defines the scope or scopes for the Cost Management API.
    - ex: providers/Microsoft.Billing/billingAccounts/cf9d8426-6a39-43c3-9709-128a891ad2d5:5a0b4b8f-5fa6-49fc-8f4e-7743746d8f04_2019-05-31
    - ex: subscriptions/5f1c1322-cebc-4ea3-8779-fac7d666e18f
    - ex: subscriptions/5f1c1322-cebc-4ea3-8779-fac7d666e18f, subscriptions/718c1322-cebc-4ea3-8779-fac7d666e18f  
  
  More informations on scopes can be found here: https://docs.microsoft.com/en-us/rest/api/cost-management/query/usage

[![Deploy](images/deploytoazure.svg?sanitize=true)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fslapointe%2Fazure-cost-anomalydetection%2Fmain%2FTemplates%2Fmain.json)


### Deployment Details  
Below are the details on the Bicep file used to deploy this solution. 

#### Log Analytics Workspace 
The data from Azure Cost API will be stored in this Log Analytics Workspace. So the first step is to deploy the workspace.  

#### Application Insights  
We want to be able to monitor the Azure Function using Application Insights. So before we deploy our Azure Function we deploy Application Insights

#### Storage Account    
Azure Function store data on a storage account. So before we deploy our Azure Function we deploy a Storage Account
 
#### Azure Function   
Now that we have all the pre-reqs deployed we can deploy the Azure Function.  The most important part of this deployment is configuring the Application Settings.  

#### Deploy Code from GitHub     
Having the Function app, we now need to actually deploy our functions. The code for this is sitting in this repo under the **AzureCost_to_LogAnalytics** folder.

#### Deploy the workbook
The last step is to actually deploy the workbook that will present the data from the discovered anomalies. The definition of the workbook is in the file **./Templates/CostWorkbook.json** and get injected by Bicep during the deployment.