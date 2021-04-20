---
title: "Azure Logic Apps for Application Gateway Reports"
date: 2021-04-20T13:59:18+12:00
draft: true
tags: ["azure","logic apps","kusto","application gateway","log analytics"]
---

Web Frontend's are probably one of the most used workloads in Cloud today.  Your ingress is basically your income for many companies, and technologies used such as Apache, NGINX, IIS mixed with your edge devices make up this stack.

In my current role we make heavy use of [Azure Application Gateways](https://docs.microsoft.com/en-us/azure/application-gateway/overview) which are L7 HTTP(S) Load Balancers for our web front end.  As the products we supply to clients are web based these are a perfect fit for our needs.  But with any web app comes web errors and if you are not logging these, you are missing them, but getting to these logs can be a challenge.

For AppGW you need to pump your diagnostic logs to a Log Analytics workspace, and then query them with [Kusto](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/concepts/).  This is fine for a quick check, but what if you want to get regular reports?  This was something I wanted to value add for our business recently and I found [Azure Logic Apps](https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-overview) could to give me the following each day:-

1. AppGW 300-499 Errors - Not great, you need to look at them
2. AppGW 500 Errors - Fix it now, you shouldn't have these
3. WAF (Web Application Firewall) Blocks - This is a good one, read on.
4. Store all of these on a SharePoint site which our WebDev Teams access.

First of all, make sure your AppGW is sending both Access and Diagnostic Logs to a Log Analytics Workspace.  Then go and setup a new Azure Logic App.  The region doesn't matter as you can query any LA Workspace.

Now when you have a logic App setup you can utilize the Designer to get started.  This gives you a GUI front end to drag and drop the app together.  Once you have everything setup, I like to export all of this to code and then its easy to reuse.

Lets start with just 1 report first so you can see the flow.  It will look like this.

- Recurrence is how often you want to run this Logic App.  Daily, weekly etc.
- Run query and list results - This is the bulk of the process, which is where your Kusto query runs
- Create CSV table - Export the results of the query above.
- Create File - Save the CSV file to SharePoint, but this could be email or anything really.

![single_la](/images/logicapp-blog-1.png)

### Run query and list results

Add an Action for **Azure Monitor Logs / Run query and list results**

When you set this up the first time, you need to point it to your subscription, resource group and the log analytics workspace you are sending AppGW logs too.  In the Query, input your Kusto, which is below

```kusto
// 500 to 599 Errors on Application Gateway Access Logs
AzureDiagnostics
| where ResourceType == "APPLICATIONGATEWAYS" and OperationName == "ApplicationGatewayAccess" and Resource == "YOUR-APPGW-NAME-HERE" and httpStatus_d between (500 .. 599)
| project TimeGenerated, Category, Resource, httpStatus_d, requestUri_s, host_s, ruleName_s, clientIP_s, httpMethod_s, userAgent_s
```

### Create CSV Table

Add an Action for **Data Operations / Create CSV Table**

When you select __Value__ you will see a Dynamic Content popup which is the results of the above query.  Add in **value** here and leave Columns as auto.

![single_csv](/images/logicapp-blog-3.png)

### Create File

Now you export the file, and for this example I am using SharePoint.  You could easily replace this email or any other location as you need.  You would just make sure the __File Content__ part at the end is the same.

So for me here I give it my SharePoint Address and Folder to store the file in.  In the file name you can include a dynamic date/time field which helps with organization.  When you click into the File Name field, just populate the expressions with ```formatDateTime(utcNow(), 'yyyy-MM-dd')``` and you are all set.

Lastly in the __File Content__ make sure to chose **Output** of the CSV Step.

![single_output](/images/logicapp-blog-4.png)

Congratulations, you are done!  Run your Logic App and you should get a nice CSV formatted file of your results

## Now Lets Get Advanced

Getting reports for HTTP error codes is certainly useful, but where this really shines is with WAF (Web Application Firewall) logs.  It's easy to get Matches or Blocks for these, but if you are only reporting Blocks then you are missing the Matches that triggered those.  That's key and one which I struggled to get reporting because you could have multiple matches stack up to reach a threshold before you get a block.  How do you report on them?

**transactionId_g** is the magical field you need to filter on here.  So lets get fancy and expand our Logic App out to what I set out to get at the start, to give me 3 daily reports, and its going to look like this...

![multi_logic_app](/images/logicapp-blog-5.png)

### WAF Blocks Query

In this step you run a Kusto query to get all WAF records that are **blocked**.  We will then use this data in the next step to filter with.  Add an Action for **Azure Monitor Logs / Run query and list results** with the below query

```
// Block AppGW Requests
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.NETWORK" and Category == "ApplicationGatewayFirewallLog" and Resource == "YOUR-APPGW-NAME-HERE" 
| where action_s contains "Blocked"
| project TimeGenerated, Category, Resource, transactionId_g, action_s, ruleSetType_s, ruleSetVersion_s, ruleId_s ,requestUri_s, hostname_s , clientIp_s, Message, details_message_s
```

### Initialize Variable

We are now setting up a blank variable array for the next step to store results in. Add an Action for **Variables / Initialize Variable**.  I called mine **BlockedData** but make sure you always pick **Array** as the type.

![multi_logic_app_var](/images/logicapp-blog-6.png)

### For Each

This is where the bulk to processing takes place so each step in summary is this:
1. Take the Output from the WAF Blocks Query
1. Run a Kusto Query to get all WAF records where it contains the **transactionId_g** from earlier.
1. Parse the results into a JSON
1. Loop through the parsed results and append them to the Array variable we created earlier.

That's a lot I know but once you have it working it make sense.

![multi_logic_app_foreach](/images/logicapp-blog-7.png)

#### Get WAF Records to Match Transaction ID

Add an Action for **Control / For Each** and in the output pick the __value__ from the previous step.  You then create an extra action inside the For Each look for **Azure Monitor Logs / Run query and list results**.  Inside that run the below Kusto.

```
// AppGW Blocks with matching transactional ID matches
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.NETWORK" and Category == "ApplicationGatewayFirewallLog" and Resource == "YOUR-APPGW-NAME-HERE" 
| where transactionId_g contains "@{items('For_each')?['transactionId_g']}"
| project TimeGenerated, Category, Resource, policyScopeName_s, hostname_s, transactionId_g, clientIp_s, action_s, ruleSetType_s, ruleSetVersion_s, ruleId_s ,requestUri_s, Message, details_message_s,details_file_s
```

Now inside that, where the `contains "@{items('For_each')?['transactionId_g']}"` part is, that is taking the dynamic content of the previous query where you got all the blocks, and getting all the WAF records for that.  This is going to give you matches and blocks!

#### Parse JSON

You now been to parse the results of the query above, which comes out in JSON so you are able to extract the results for the for each loop.  Otherwise you don't get all the data coming through.  I used this formatting for the query earlier.

```json
{
    "items": {
        "properties": {
            "Category": {
                "type": "string"
            },
            "Message": {
                "type": "string"
            },
            "Resource": {
                "type": "string"
            },
            "TimeGenerated": {
                "type": "string"
            },
            "action_s": {
                "type": "string"
            },
            "clientIp_s": {
                "type": "string"
            },
            "details_file_s": {
                "type": "string"
            },
            "details_message_s": {
                "type": "string"
            },
            "hostname_s": {
                "type": "string"
            },
            "policyScopeName_s": {
                "type": "string"
            },
            "requestUri_s": {
                "type": "string"
            },
            "ruleId_s": {
                "type": "string"
            },
            "ruleSetType_s": {
                "type": "string"
            },
            "ruleSetVersion_s": {
                "type": "string"
            },
            "transactionId_g": {
                "type": "string"
            }
        },
        "required": [
            "TimeGenerated",
            "Category",
            "Resource",
            "policyScopeName_s",
            "hostname_s",
            "transactionId_g",
            "clientIp_s",
            "action_s",
            "ruleSetType_s",
            "ruleSetVersion_s",
            "ruleId_s",
            "requestUri_s",
            "Message",
            "details_message_s",
            "details_file_s"
        ],
        "type": "object"
    },
    "type": "array"
}
```

#### Final For Each

Add an Action for **Control / For Each** and for the output you will pick the **Body** of the Parse JSON step.

Then add an action for **Variables / Append to Array Variable** and give it the name of your Array you created earlier.  The Value is going to be the **Current Item** of the For Each loop, which ensures the records being parsed for a particular transaction ID all come through correctly.  This is how you get Matches and Blocks.

#### Create CSV Table

Same as earlier with the simple steps, add an action for **Data Operations / Create CSV Table**.  This time for the From data you are going to select the **Array Variable** from earlier.

#### Create File

You have made it, export your file to wherever you want, just make sure that the File Content is coming from the **Output** of your final Create CSV Table Action.

You will now have a CSV Report which contains WAF Firewall Logs for every Block that occurs, along with all the matches that triggered it.  This was a huge value add for our company as we can now get daily reports into the hands of the web developers showing all the WAF blocks on the firewall and the direct actions which caused it so they can work on a fix.

This took me a long time to figure out initially so I hope by sharing I can save at least 1 person the time it took me to get it working.  Logic Apps really are a game changer for automating processes like this, so when I find other uses for them I'll be sure to share.

Andy