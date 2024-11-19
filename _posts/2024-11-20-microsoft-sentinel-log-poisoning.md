---
title: "Log Poisoning in Microsoft Sentinel"
date: "2024-11-20"
categories: 
  - "cloud"
  - "security"
tags: 
  - "microsoft-sentinel"
  - "siem"
  - "soar"
  - "security"
  - "msrc"
coverImage: "new-ceng-logo-cropped.jpg"
---

I decided recently to dip my toe into security research in my spare time. This post is a writeup of some interesting goofiness that I found when playing around with Microsoft Sentinel Log Poisoning.

I raised the findings with [Microsoft Security Response Center (MSRC)](https://msrc.microsoft.com/), who after doing very little for a month, closed the case (`91889`) as low severity, which means I can finally write about it.

In writing this post, i realised that there is quite a bit of assumed learning, so i've tried to provide a relatively high level introduction, before exploring the Log Poisioning findings.

## What is Microsoft Sentinel?
Microsoft Sentinel is a Security Information and Event Management (SIEM) and a Security Orchestration, Automation and Response (SOAR) product all in one. It's not only capable of analysing all your logs, but you can also automate responses.

Microsoft Sentinel runs on Microsoft Azure, but its not just for Azure workloads. Microsoft tried to make this distinction clear in renaming the product from "Azure Sentinel" to "Microsoft Sentinel". You can integrate any kind of log into Microsoft Sentinel, from on-premise logs, to AWS workloads.

## How does Microsoft Sentinel work?
Behind the scenes, a cloud-scale database is provisoned named an [Azure Log Analytics workspace](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/log-analytics-workspace-overview). All of your logs need to be ingested into this database (more on that later). From there, you can query the data using Microsoft own query language - [Kusto Query Language](https://learn.microsoft.com/en-us/kusto/query)(KQL).

KQL and Log Analytics workspaces are fast. They are designed for querying large volumes of data in next to no time at all.

I dont want to downplay Microsoft Sentinel, but its essentially just a wrapper around this database. It streamlines the process for querying data on a schedule, helps to create incidents (and the whole management process around that), maps to security frameworks such as MITRE, and allows for automated response with integration into Azure Logic Apps (Microsoft Sentinel playbooks).

## Microsoft Sentinel - Analytic Rules & Incidents
Very quickly, I need to explain some of the components of Microsoft Sentinel, to help you understand the impact of this piece of research. 

One of the core components of Microsoft Sentinel is an [analytics rule](https://learn.microsoft.com/en-us/azure/sentinel/scheduled-rules-overview). These rules run on a schedule and execute KQL queries against your dataset.
![](../images/log-poisoning/analytics-rule-kql.png)

In the above screenshot, the analytic rule anmed "SSH - Potential Brute" force contains a pretty long KQL query, which will be ran periodically against the Log Analytics workspace.

Within Microsoft Sentinel, there is a Content Hub, which allows you to install a number of Microsoft Sentinel rules/queries into your deployment. For example the Security Threat Essentials solution is created and maintained by Microsoft, and includes 7 analytics rules.
![](../images/log-poisoning/content-hub-essentials.png)
There are tonnes more solutions in the content hub, allowing you to quickly install hundreds of queries and rapidly improve your coverage.

Outside the content hub, there is an the [official Microsoft Azure Sentinel repository](https://github.com/Azure/Azure-Sentinel/blob/master/Solutions/Syslog/Analytic%20Rules/ssh_potentialBruteForce.yaml), which contains a bunch of great rules from across the community, and as well as brilliant resources such as [kqlsearch.com](https://www.kqlsearch.com/) which collates queries from a lots of sources.

Once you have your analytics rules in place, they will run the query on a defined schedule. Inside the rule logic, you define the logic around:
- what to do if your query returns a result
- if you have a result, map the data to entities
- how many matches trigger an alert
- how many alerts trigger an incident

Finally, when the analytics rule is automatically ran on the defined schedule, if it meets your criteria, it'll create an incident.
![](../images/log-poisoning/incident-example.png)

Incidents can then be assigned to invididuals for triage/remediation, escalation, context linked to other incidents and all kinds of fancy things.

## What is Log Posioning?
Log Poisoning is a cyberattack technique where an attacker manipulates or injects malicious data into logs collected by a SIEM system. The goal is to disrupt, deceive, or manipulate how the SIEM processes and presents log data. This can mislead security analysts, obscure malicious activity, or even facilitate further attacks by exploiting trust in the SIEM system.

This can be bad for :
- Evasion of detection (bypassing detection rules) - An analytics query may be looking for sequential events, or if an analyst is trying to trace logs sequentially. 
- Manipulation of Alerts/Incidents - Malicious log entries can trigger false alarms, overwhelming analysts with noise and distracting them from genuine threats (a tactic known as "alert fatigue").
- Disruption of forensic fnvestigations - Poisoned logs can corrupt the integrity of the evidence, making it difficult to investigate breaches or reconstruct the attacker's actions.
- Trust Degradation - SIEM systems are central to an organization’s security operations. If logs are compromised, it undermines trust in the system and the ability to make data-driven decisions.
- Non-repudiation - depending on the situation, its possible for a log poisoning attack to either deceive or maniuplate logs directly and cover up a malicious actors tracks.
- Obfuscating malicious activities
- Implicate another party in the commission of a malicious act
- Exhausting SOC Resources
- Manipulating Compliance Audits

Finally, it can be bad for your wallet...

Microsoft Sentinel is a cloud solution - there arent any servers to manage here. As with a lot of cloud products & services, the costs can mount up as you scale. For Log Analytics databases, a successful log poisioning attack could easily result in a [denial of wallet](https://portswigger.net/daily-swig/denial-of-wallet-attacks-how-to-protect-against-costly-exploits-targeting-serverless-setups) attack. Current upload costs for a single Log Analytics workspace depend on your pricing strategy, but if your on Pay-As-You-Go, its as much as $5.38 per GB.

![](../images/log-poisoning/sentinel-pricing.png)

Lets be clear, uploading 10,000 GB of logs, would cost you $53,800 a day if you are on a Pay-As-You-Go tier!

With the log poisioning attack i'm about to show you, you may not even know that your logs are increasing!

## Microsoft Sentinel Log Ingestion
Before we dive into how the *potential* log poisoning works, you'll need to understand how Microsoft Sentinel log ingestion works.

You dont ingest logs into Microsoft Sentinel, but rather the Log Analytics workspace (database) behind it. Microsoft Sentinel is "just" a wrapper around the database. So the question become, how do you ingest logs into a Log Analytics workspace?

Azure Monitor is the service

This post is chiefly concerned with logs for compute, which requires three things:

### Azure Monitor Agent (AMA)
The agent colectrion adn sends logs to the Azure Monitor service.

The recommended way to install the AMA onto your virtual machines, is by creating a Data Collection Rule. But you can also use Azure Policy. 
The Microsoft Sentinel data connector, is essentially a wrapper around Azure Policy.

Any associated machines will need "permissions" to 


The AzureMonitorLinuxAgent is open source and is available [here](https://github.com/Azure/azure-linux-extensions/tree/master/AzureMonitorAgent).
The AzureMonitorWindowsAgent isnt on github, but once deployed on a Windows machine, it can easily be found on a deployed VM in the `C:\\\\\\` directory.

If you are collecting data for Syslog or Windows events via the Azure Monitor agent (which is very common), [you will need a Data Collection Endpoint](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/data-collection-endpoint-overview?tabs=portal#azure-monitor-agent-ama)

### Data Collection Endpoint (DCE)
An [Azure Monitor Data Collection Endpoint](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/data-collection-endpoint-overview?tabs=portal) defines an endpoint, for 


https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/data-collection-endpoint-overview?tabs=portal#azure-monitor-agent-ama


```
{
    "properties": {
        "description": "a data collection endpoint",
        "immutableId": "dce-a4aea7fb8c4b4053a10d07c2e0f7a5c2",
        "configurationAccess": {
            "endpoint": "https://my-dce-7xo3.uksouth-1.handler.control.monitor.azure.com"
        },
        "logsIngestion": {
            "endpoint": "https://my-dce-7xo3.uksouth-1.ingest.monitor.azure.com"
        },
        "metricsIngestion": {
            "endpoint": "https://my-dce-7xo3.uksouth-1.metrics.ingest.monitor.azure.com"
        },
        "networkAcls": {
            "publicNetworkAccess": "Enabled"
        },
        "provisioningState": "Succeeded"
    },
    "location": "uksouth",
    "tags": {},
    "id": "/subscriptions/fdc24141-9008-45d1-a749-fa990d42a015/resourceGroups/msft-sentinel/providers/Microsoft.Insights/dataCollectionEndpoints/my-dce",
    "name": "my-dce",
    "type": "Microsoft.Insights/dataCollectionEndpoints"
}
```

### Data Collection Rules (DCR)
[Azure Montor Data Collection Rules](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/data-collection-rule-overview) are the core component to data ingestion into your workspace.

They essentially define the incoming data stream definition, perform any data transformations are required, and forward onto a table in the Log Analytics workspace.

A DCR must send to an endpoint; which is either typically a the Log Analytics workspace or the Data Collection Endpoint.

Here is an example of a DCR which collects `Sylog` type data from a Linux Virtual machine, and forwards onto my Log Analytics workspace
```
{
    "properties": {
        "description": "",
        "immutableId": "dcr-62eeb31f81e64b8daf1393140d61069e",
        "dataSources": {
            "syslog": [
                {
                    "streams": [
                        "Microsoft-Syslog"
                    ],
                    "facilityNames": [
                        "auth",
                        "authpriv",
                        "cron",
                        "daemon",
                        "kern",
                        "lpr",
                        "mail",
                        "mark",
                        "news",
                        "syslog",
                        "user",
                        "uucp",
                        "local0",
                        "local1",
                        "local2",
                        "local3",
                        "local4",
                        "local5",
                        "local6",
                        "local7"
                    ],
                    "logLevels": [
                        "Debug",
                        "Info",
                        "Notice",
                        "Warning",
                        "Error",
                        "Critical",
                        "Alert",
                        "Emergency"
                    ],
                    "name": "syslog-live-logs"
                }
            ]
        },
        "destinations": {
            "logAnalytics": [
                {
                    "workspaceResourceId": "/subscriptions/xxxxxxxxxxxxxxxxxxxxx/resourceGroups/msft-sentinel/providers/Microsoft.OperationalInsights/workspaces/sentinel",
                    "workspaceId": "1827c797-4a4a-4a68-b6f3-xxxxxxxx",
                    "name": "syslog-live-logs"
                }
            ]
        },
        "dataFlows": [
            {
                "streams": [
                    "Microsoft-Syslog"
                ],
                "destinations": [
                    "syslog-live-logs"
                ]
            }
        ],
        "provisioningState": "Succeeded"
    },
    "location": "uksouth",
    "tags": {},
    "kind": "Linux",
    "id": "/subscriptions/xxxxxxxxxxxxxxxxxx/resourceGroups/research/providers/Microsoft.Insights/dataCollectionRules/SyslogLive",
    "name": "SyslogLive",
    "type": "Microsoft.Insights/dataCollectionRules"
}
```

The Windows equivelant is much simplier:
```
{
    "properties": {
        "description": "",
        "immutableId": "dcr-1316a9fba5154d658ba5a7ea1c3d2eef",
        "dataSources": {
            "windowsEventLogs": [
                {
                    "streams": [
                        "Microsoft-Event"
                    ],
                    "xPathQueries": [
                        "Application!*[System[(Level=1 or Level=2 or Level=3 or Level=4 or Level=0 or Level=5)]]",
                        "Security!*[System[(band(Keywords,13510798882111488))]]",
                        "System!*[System[(Level=1 or Level=2 or Level=3 or Level=4 or Level=0 or Level=5)]]"
                    ],
                    "name": "eventLogsDataSource"
                }
            ]
        },
        "destinations": {
            "logAnalytics": [
                {
                    "workspaceResourceId": "/subscriptions/xxxxxxxxxxxxxxxxxx/resourceGroups/msft-sentinel/providers/Microsoft.OperationalInsights/workspaces/sentinel",
                    "workspaceId": "1827c797-4a4a-4a68-b6f3-fb3e3cefd50a",
                    "name": "windows-live-logs"
                }
            ]
        },
        "dataFlows": [
            {
                "streams": [
                    "Microsoft-Event"
                ],
                "destinations": [
                    "windows-live-logs"
                ]
            }
        ],
        "provisioningState": "Succeeded"
    },
    "location": "uksouth",
    "tags": {},
    "id": "/subscriptions/xxxxxxxxxxxxxxxxxx/resourceGroups/research/providers/Microsoft.Insights/dataCollectionRules/WindowsLiveLogs",
    "name": "WindowsLiveLogs",
    "type": "Microsoft.Insights/dataCollectionRules"
}
```

These Data Collection Rules are configured to point to my DCE
![](../images/log-poisoning/dcr-resources.png)

You make have noticed that the DCE has an `immutableId` field, which is referenced in the DCRs.

Essentially, the architecture looks something like this:
< todo >






# Tables and Custom Tables
One final distinction you need to know, is that there is 

## Log Posioning in Microsoft Sentinel
Ok, so now we are in a position to explore what I wanted to talk about... Log Poisoning in Microsoft Sentinel.





## Mitigations
- You could keep an eye on your logs, by running a daily KQL query which has something like this:
```
Usage
| summarize TotalSizeInMB = sum(Quantity) by DataType
| extend TotalSizeInGB = TotalSizeInMB / 1024
| order by TotalSizeInGB desc
```

- This really isnt recommended, but in theory, you could update your analytic rules to parse the `_ResourceId` directly, rather than take the reported values at face value. 
Obviously, this depends on Microsoft fixing the issue where metadata fields can be spoofed, as I outlined earlier in the post.


### Timeline:
Submission number: `VULN-136697`
Case number: `91889`

- 11th October 2024 - Submitted to MSRC
- 15th October 2024 - Status changed to `Review/Repro`
- 27th October 2024 - I chased for an update
- 11th November 2024 - I chased for an update
- 12th November 2024 - Determined to be low severity, caes closed