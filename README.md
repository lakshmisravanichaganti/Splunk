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
[^\"] matches any pattern not of this specified pattern, here matches any string not haFruitg \" quote.

```stats count(fruits) AS TotalFruits,count(Varieties) AS TotalVarieties by Name, color```

Name    Color      TotalFruits    TotalVarieties
__    ______      ___________   _______________
Mango     yellow       12              3
Apple     green	        20              5

``` rename Name AS "Fruit Name", Color AS "Fruit Color"

sort  -Fruit Name, -_time 
```

#TRIGGER ANOTHER PANEL SEARCH BASED ON INPUT SEARCH
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
| rex field=msg "FruitList\":\[(?<Fruit_list>[^\]]+).*requestType\":\"(?<requestType>[^\"]+).*requestId\":\"(?<requestId>[^\"]+).*srcAppCd\":(?<srcAppCd>[^,]+).*createUser\":\"(?<createUser>[^\"]+).*varietiesDetailsList\":\[(?<varietiesDetailsList>[^\]]+)" 
```
```
index=my_org 	 cf_space_name=my_space	
| search "ReceivedmentRequest" AND $srequestId_Search|s$ AND $srequestId|s$
| rex field=msg "FruitList\":\[(?<Fruit_list>[^\]]+).*requestType\":\"(?<requestType>[^\"]+).*requestId\":\"(?<requestId>[^\"]+).*srcAppCd\":(?<srcAppCd>[^,]+).*createUser\":\"(?<createUser>[^\"]+).*varietiesDetailsList\":\[(?<varietiesDetailsList>[^\]]+)" 
| eval Fruits=split(Fruit_list,",")
| mvexpand Fruits 
| eval FruitDtl=replace(Fruits,"\"","")  
| eval TypeLists=split(varietiesDetailsList,"},") 
| mvexpand TypeLists 
| rex field=TypeLists "typeId\":\"(?<typeId>[^\"]+).*.*expiry\":\"(?<expiry>[^\"]+)"
| dedup requestId,FruitDtl,typeId  
| rename FruitDtl AS Fruit, typeId AS "Type ID", expiry AS "Type Expiration Date"
| table Fruit,"Type ID","Type Expiration Date"
| rex field=msg "FruitList\":\[(?<Fruit_list>[^\]]+).*requestType\":\"(?<requestType>[^\"]+).*requestId\":\"(?<requestId>[^\"]+).*srcAppCd\":(?<srcAppCd>[^,]+).*createUser\":\"(?<createUser>[^\"]+).*varietiesDetailsList\":\[(?<varietiesDetailsList>[^\]]+)" 
| eval Fruits=split(Fruit_list,",")
| mvexpand Fruits 
| eval FruitDtl=replace(Fruits,"\"","")  
| eval TypeLists=split(varietiesDetailsList,"},") 
| mvexpand TypeLists 
| rex field=TypeLists "typeId\":\"(?<typeId>[^\"]+).*.*expiry\":\"(?<expiry>[^\"]+)"
| dedup requestId,FruitDtl,typeId  
| rename FruitDtl AS Fruit, typeId AS "Type ID", expiry AS "Type Expiration Date"
| table Fruit,"Type ID","Type Expiration Date"
```

##### SEARCH TYPES
1.search - searches for particular event type match
2.union - searches for two events
3.multisearch - searches for more than two events 
4.join - joins searches 


##### MULTISEARCH
```
| multisearch

[search index=my_org cf_space_name=my_space ("message1" 
         OR "message2" 
         OR "message3")
         AND "CustomLogMessage" AND $BunchId|s$
| rex field=msg "eventFruitRecord\":\"(?<Fruit>[^\"]+).*\"eventFruitId\":\"$BunchId|s$\".*\"eventStatusCd\":\"(?<StatusCd>[^\"]+).*\"eventStatusMsg\":\"(?<statusMsg>[^\"]+).*\"eventType\":\"(?<eventType>[^\"]+).*\"eventTransactionHeaderId\":\"(?<TransactionHeaderId>[^\"]+)"
| fields eventType,BunchId,statusMsg
] 

[search index=my_org cf_space_name=my_space "message1" AND "CustomLogMessage" AND $BunchId|s$ AND $Fruit|s$ AND $typeId|s$ 
| rex field=msg "eventFruitRecord\":\"(?<Fruit>[^\"]+).*\"eventFruitId\":\"$BunchId|s$\".*\"eventType\":\"(?<eventType>[^\"]+).*\"eventAdditionalInfo\":{.*\"additionalStatus\":\[.*{\"typeId\":\"$typeId|s$\",\"typeStatus\":(?<statusCd>[^,])"
| eval statusMsg = case(statusCd=="1","SUCCESS", statusCd == "0", "FAILED")]
| eval Fruit = $Fruit|s$
| eval typeId = $typeId|s$
| eval BunchId = $BunchId|s$
| dedup Fruit,typeId,eventType,BunchId,statusMsg
| table Fruit,typeId,eventType,BunchId,statusMsg,_time
|sort -_time
```
##### MVEXPAND
```
[search index=my_org cf_space_name=my_space "VsdnCorelatedAlertResponseHandler" AND "CustomLogMessage" AND $BunchId|s$ AND $Fruit|s$ AND $typeId|s$ 
| rex field=msg "eventFruitRecord\":\"(?<Fruit>[^\"]+).*\"eventFruitId\":\"$BunchId|s$\".*\"eventType\":\"(?<eventType>[^\"]+).*\"eventAdditionalInfo\":{.*\"additionalStatus\":\[.*{\"typeId\":\"$typeId|s$\",\"typeStatus\":(?<statusCd>[^,])"
| eval statusMsg = case(statusCd=="1","SUCCESS", statusCd == "0", "FAILED")]
| eval Fruit = $Fruit|s$
| eval typeId = $typeId|s$
| eval BunchId = $BunchId|s$
| dedup Fruit,typeId,eventType,BunchId,statusMsg
| table Fruit,typeId,eventType,BunchId,statusMsg,_time
|sort -_time
```

