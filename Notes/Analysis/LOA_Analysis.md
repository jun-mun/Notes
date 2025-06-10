# LOA Analysis

## 📌 Objective

NOTE: This analysis is done from the perspective of changing an employment status from MSS. We're hoping this will give us insight into a better solution with the Leave Integration project.

## Overview 
### Brief snapshot of the stack trace:

![Alt text](../images/LOA_analysis_debug_stack_trace.png "Stack Trace")

## 1. ActEmplConfirm

The flow starts with the user making a employment status change (in this case 'active -> LOA unpaid leave')

When the user continues,  **ActEmplConfirm#hraPlusPerform** method is called. This is where we call the EmploymentInfoBean#save method as shown below:

![Alt text](../images/ActEmplConfirm_save.png "ActEmplConfirm")

## 2. EmploymentInfoBean/susemp_process sproc

In EmploymentInfoBean#save method, EmployeeDAOImpl is called to sp_susemp_process.
The actual changes are being commited through this huge sproc... 

![Alt text](../images/Screenshot%202025-06-05%20171815.png?raw=true "Title")

Here is the stack when calling the susemp stored procedure:
![Stack when calling susemp sproc](../images/Screenshot%202025-06-06%20113336.png "")

I think this is where a detect_event record is added.

## 3. EventEngine
Then the record gets picked up by the **EventEngine**. Where we eventually call a method, EventEngine#processEventsWithActionsToProcess. 

![Where actions are generated](../images/getEventsWithActionsToProcess.png "")

As you can see on line 948, a DB call is made to gather a list of events with actions associated with them. This line calls thefollowing query:

```
get-events-with-actions-by-eeid

# Eventually calls the following sproc
call ues.dbo.GetEventsWithActionsByEeId(#value#)
```
[Link to Proc_GetEventsWithActionsByEeId.sql](https://github.com/AlightEngineering/CBA_cba-db-sources/blob/530de2c2a98b5e111cd52cd6aaf1a6a156494c5f/ues/Proc_GetEventsWithActionsByEeId.sql#L9)

<mark>^^ This might be the key to this whole thing</mark>

The LOA gets processed here![Alt text](../images/LOA_processing_method.png?raw=true "Title")

Most of the event processing occurs in EventEngine. 
## ✅ TODO
- check out the logic on how EventActions are generated.
- figure out what sp_susemp_process does

