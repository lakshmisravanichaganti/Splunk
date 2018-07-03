# Splunk

##### SPLUNK DASHBOARD CREATION

1. Create a Dashboard/Clone an exisiting dashboard
2. Click on Edit dashboard
3. Click on Add Panel, give a panel name and a sample search string, click save
4. Now the panel is shown on dashboard
5. Click on Edit, to add the actual search string


 ##### SEARCH STRING

index refers to the orgin which the splunk search happens
cf_space - organisation space 
cf_app_name - optional, if specified, scans only the specified app

search command - searches for a string match to pull out all the logs

rex command is used to match a regex pattern on a field

Every log in splunk is considered as event. regex is written on the splunk event.
Splunk event consists of timeshtamp, msg, event_type say logMessage, etc.

```
index=xxx cf_space=xxx cf_app_name=xxx 
| search "" 
| rex field = msg "xxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

Note: Use regex101.com to test the regex

Example message:
```
INFO [SplunkLearningApp] -- CustomLogMessage {"statusCd":200,"statusMessage":"SUCCESS"}

msg "status\":\(?<status>[^\]+).*\"statusMessage\":\"(?<statusMessage>[^\"]+)"

eval finalStatus = case(statusCd=="200","SUCCESS", statusCd == "433", "PARTIAL_SUCCESS") 
```
? - matches zero or more occurences
[^\"] matches any pattern not of this specified pattern, here matches any string not having \" quote.

```stats count(fruits) AS TotalFruits,count(Varieties) AS TotalVarieties by Name, color```

Name    Color      TotalFruits    TotalVarieties
__    ______      ___________   _______________
Mango     yellow       12              3
Apple     green	        20              5

``` rename Name AS "Fruit Name", Color AS "Fruit Color"

sort  -Fruit Name, -_time 
```

**********TRIGGER ANOTHER PANEL SEARCH BASED ON INPUT SEARCH*********
Click on Edit dashboard, go to panel
Click on vertical dots for more actions, click on edit drilldown
In Drilldown editor, select Manage tokens on this dashboard

To use the selected row, first column value`
```
set variableName $click.value$
```

To use any specific column
set variableName $row.variableInSearch$
```
index=my_org 	 cf_space_name=my_space	
| search "CustomLogMessage" AND $Name|s$ AND $Color`|s$
| rex field=msg "vinList\":\[(?<vin_list>[^\]]+).*requestType\":\"(?<requestType>[^\"]+).*traceId\":\"(?<traceId>[^\"]+).*srcAppCd\":(?<srcAppCd>[^,]+).*createUser\":\"(?<createUser>[^\"]+).*functionDetailsList\":\[(?<functionDetailsList>[^\]]+)" 
```
```
index=my_org 	 cf_space_name=my_space	
| search "ReceivedDeploymentRequest" AND $sTraceId_Search|s$ AND $sTraceId|s$
| rex field=msg "vinList\":\[(?<vin_list>[^\]]+).*requestType\":\"(?<requestType>[^\"]+).*traceId\":\"(?<traceId>[^\"]+).*srcAppCd\":(?<srcAppCd>[^,]+).*createUser\":\"(?<createUser>[^\"]+).*functionDetailsList\":\[(?<functionDetailsList>[^\]]+)" 
| eval vins=split(vin_list,",")
| mvexpand vins 
| eval vinDtl=replace(vins,"\"","")  
| eval functionLists=split(functionDetailsList,"},") 
| mvexpand functionLists 
| rex field=functionLists "functionId\":\"(?<functionId>[^\"]+).*.*functionExpiryDate\":\"(?<functionExpiryDate>[^\"]+)"
| dedup traceId,vinDtl,functionId  
| rename vinDtl AS Vin, functionId AS "Function ID", functionExpiryDate AS "Function Expiration Date"
| table Vin,"Function ID","Function Expiration Date"
| rex field=msg "vinList\":\[(?<vin_list>[^\]]+).*requestType\":\"(?<requestType>[^\"]+).*traceId\":\"(?<traceId>[^\"]+).*srcAppCd\":(?<srcAppCd>[^,]+).*createUser\":\"(?<createUser>[^\"]+).*functionDetailsList\":\[(?<functionDetailsList>[^\]]+)" 
| eval vins=split(vin_list,",")
| mvexpand vins 
| eval vinDtl=replace(vins,"\"","")  
| eval functionLists=split(functionDetailsList,"},") 
| mvexpand functionLists 
| rex field=functionLists "functionId\":\"(?<functionId>[^\"]+).*.*functionExpiryDate\":\"(?<functionExpiryDate>[^\"]+)"
| dedup traceId,vinDtl,functionId  
| rename vinDtl AS Vin, functionId AS "Function ID", functionExpiryDate AS "Function Expiration Date"
| table Vin,"Function ID","Function Expiration Date"
```

##### SEARCH TYPES
1.search - searches for particular event type match
2.union - searches for two events
3.multisearch - searches for more than two events 
4.join - joins searches 


##### MULTISEARCH
```
| multisearch

[search index=my_org cf_space_name=my_space ("VsdnUpgResponseHandler" 
         OR "VsdnIOTResponseHandler" 
         OR "VsdnCommandResponseHandler")
         AND "CvdaLog" AND $deployBatchId|s$
| rex field=msg "eventVinRecord\":\"(?<Vin>[^\"]+).*\"eventVsdnBatchId\":\"$deployBatchId|s$\".*\"eventStatusCd\":\"(?<StatusCd>[^\"]+).*\"eventStatusMsg\":\"(?<statusMsg>[^\"]+).*\"eventType\":\"(?<eventType>[^\"]+).*\"eventTransactionHeaderId\":\"(?<TransactionHeaderId>[^\"]+)"
| fields eventType,BatchId,statusMsg
] 

[search index=my_org cf_space_name=my_space "VsdnCorelatedAlertResponseHandler" AND "CvdaLog" AND $deployBatchId|s$ AND $deployVin|s$ AND $deployFunctionId|s$ 
| rex field=msg "eventVinRecord\":\"(?<Vin>[^\"]+).*\"eventVsdnBatchId\":\"$deployBatchId|s$\".*\"eventType\":\"(?<eventType>[^\"]+).*\"eventTransactionHeaderId\":\"(?<TransactionHeaderId>[^\"]+).*\"eventAdditionalInfo\":{.*\"avdStatus\":\[.*{\"functionId\":\"$deployFunctionId|s$\",\"dvdfunctionStatus\":(?<statusCd>[^,])"
| eval statusMsg = case(statusCd=="1","SUCCESS", statusCd == "0", "FAILED")]
| eval vin = $deployVin|s$
| eval functionId = $deployFunctionId|s$
| eval BatchId = $deployBatchId|s$
| dedup vin,functionId,eventType,BatchId,statusMsg
| table vin,functionId,eventType,BatchId,statusMsg,_time
|sort -_time
```
##### MVEXPAND
```
[search index=my_org cf_space_name=my_space "VsdnCorelatedAlertResponseHandler" AND "CvdaLog" AND $deployBatchId|s$ AND $deployVin|s$ AND $deployFunctionId|s$ 
| rex field=msg "eventVinRecord\":\"(?<Vin>[^\"]+).*\"eventVsdnBatchId\":\"$deployBatchId|s$\".*\"eventType\":\"(?<eventType>[^\"]+).*\"eventTransactionHeaderId\":\"(?<TransactionHeaderId>[^\"]+).*\"eventAdditionalInfo\":{.*\"avdStatus\":\[.*{\"functionId\":\"$deployFunctionId|s$\",\"dvdfunctionStatus\":(?<statusCd>[^,])"
| eval statusMsg = case(statusCd=="1","SUCCESS", statusCd == "0", "FAILED")]
| eval vin = $deployVin|s$
| eval functionId = $deployFunctionId|s$
| eval BatchId = $deployBatchId|s$
| dedup vin,functionId,eventType,BatchId,statusMsg
| table vin,functionId,eventType,BatchId,statusMsg,_time
|sort -_time
```

