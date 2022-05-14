---
layout: post
title:  "Modelling database interaction with TLA+"
date:   2022-05-08 20:00:00 +0200
categories: jekyll update
---

In this post i will present how I have modeled the interaction with a `postgresql` database of a backend in Go. 
The backend has not been replicated yet, so it will consist of 3 single instances services. Even though the design of the 3 services is not optimal I will present it in this post to show how the interaction with a `SQL` database can be modeled in `TLA+`. How to model database transactions change according to the `isolation level` used, in this post i will show the `serializable` isolation level and i will give some advice on how to model the other main levels. 

The system is made of 3 services, it's aim is to sign and broadcast items on an external service. But errors may occur, also on the external services, so there are components that will try to restore items that failed and will try to reenquee them. 

The first process is `Msg_Sender`, it will get all processes in db with status `enqueue`, it will select a set of them (we can for simplicity take it randomly) and will try to sign and broadcast them, if all works correctly update the status of the processes in the db to `processed`.

The process `StalledProcessesFinder` get from the db all the processes in status `processing` or `recovering` and check if they are `timedOut` (for simplicity in the model we will select them randomnly), if so updates the processes in the db updating the status to `timeOut`.

The process `HandleTimedOutTransactions` get the `timedOut` processes form the db, update their status to `recovering`, search the process by id in the external services, if the process is found then the process will finalize it in the db otherwise it will try to reenquee it in the db (for semplicity i will model possibly an infinite number of reenquees) 


The main variables used are the following, for making the model checking quicker I used only one process but the model is the same for multiple processes. The variable `inputProcesses` represent the processes to be inserted as `enqueue` in the db, this is done by another process not modeled in the specification. It simply take the processes from the sequence and put them in the set representing the database.
```
Init ==
        /\ components = {"msg_sender", "stalledProcessFinder", "handleTimeOutTransactions"}
        /\ programCounter = [self \in components |-> "start"]        
        /\ componentState = [x \in components |-> "ready"]
        /\ inputProcesses = << [id |-> "0", status |-> "null", txHash |-> "null"] >>
        /\ database = {}
        /\ signedAndBroadcasted = {}
        /\ table_lock = [msg_sender |-> "null", stalledProcessFinder |-> "null", handleTimeOutTransactions |-> "null"]
```

So lets starts with `Msg_sender`.
```
dequeueProcesses_get == /\  componentState["msg_sender"] = "ready"
                        /\  programCounter["msg_sender"] = "start"
                        /\  \E x \in database : x["status"] = "enqueue"
                        /\  table_lock["msg_sender"] = "null" /\ table_lock["stalledProcessFinder"] = "null" /\ table_lock["handleTimeOutTransactions"] = "null"
                        /\  local_dequeueProcesses_processesGet' = {x \in database: x["status"]="enqueue"}
                        /\  programCounter' = [programCounter EXCEPT !["msg_sender"] = "gotProcesses"]
```

Here in msg sender we don't have explicitly defined transactions, but using the `GORM` orm we have by default a transaction for each `CRUD` operation. So we need to check that the `lock` in the table has not been already taken. In the spec the only multi-transition transaction take a write lock so for semplicity we only record write lock and we don't alter the semantic of the model. Because of the transaction we can model the read operation on db as a single atomic operation. 
Here we have read the processes with status enqueue.

Then we have to update the database updating the processes status to `processing`. For simulating a database update we simply remove the processes from the set database and then immediately add the processes updated.
```
dequeueProcesses_update ==  /\  componentState["msg_sender"] = "ready"
                            /\  programCounter["msg_sender"] = "gotProcesses"
                            /\  table_lock["msg_sender"] = "null" /\ table_lock["stalledProcessFinder"] = "null" /\ table_lock["handleTimeOutTransactions"] = "null"
                            /\  local_dequeueProcesses_selected' = CHOOSE x \in (SUBSET local_dequeueProcesses_processesGet) : Cardinality(x) > 0
                            /\  LET local_dequeueProcesses_processing == {[x EXCEPT !["status"] = "processing"]: x \in local_dequeueProcesses_selected'}
                                    local_dequeueProcesses_selected_ids == {x \in {"0", "1", "2", "3", "4"} : (\E y \in  local_dequeueProcesses_selected' : y["id"] = x)}                         
                                    local_dequeueProcess_database == database
                                IN  database' = (database \ ({x \in local_dequeueProcess_database : (\E y \in local_dequeueProcesses_selected_ids : x["id"] = y)})) \union local_dequeueProcesses_processing
                            /\  LET local_dequeueProcesses_processing == {[x EXCEPT !["status"] = "processing"]: x \in local_dequeueProcesses_selected'}
                                IN msg_sender_selected' =  local_dequeueProcesses_processing
                            /\  programCounter' = [programCounter EXCEPT !["msg_sender"] = "updated"]
```


Finally we have to sign and broadcast the processes:
```
submitNewTxs_signAndBroadcast ==    /\  componentState["msg_sender"] = "ready"
                                    /\  programCounter["msg_sender"] = "updated"
                                    /\  signedAndBroadcasted' = signedAndBroadcasted \union {[x EXCEPT !["txHash"] = "0x53f",
                                                                                                       !["status"] = "processed"]: x \in msg_sender_selected}                                    
                                    /\  programCounter' = [programCounter EXCEPT !["msg_sender"] = "signedAndBroadcasted"]
```
For this we simply put the process in the set signedAndBroadcasted.

Then we need to update the database with the status `processed`, we will update the database as above:
```
submitNewTxs_updateBroadcasted ==   /\  componentState["msg_sender"] = "ready"
                                    /\  programCounter["msg_sender"] = "signedAndBroadcasted"
                                    /\  table_lock["msg_sender"] = "null" /\ table_lock["stalledProcessFinder"] = "null" /\ table_lock["handleTimeOutTransactions"] = "null"
                                    /\  LET local_dequeueProcesses_selected_ids == {x \in {"0", "1", "2", "3", "4"} : (\E y \in  local_dequeueProcesses_selected : y["id"] = x)}                         
                                        IN  local_submitNewTxs_processes' = {x \in database: x["id"] \in local_dequeueProcesses_selected_ids}
                                    /\  local_submitNewTxs_updatedProcesses' = {[x EXCEPT !["status"] = "processed",
                                                                                          !["txHash"] = "0x53f"]: x \in local_dequeueProcesses_selected}
                                    /\  database' = (database \ local_submitNewTxs_processes') \union local_submitNewTxs_updatedProcesses'                                                     
                                    /\  programCounter' = [programCounter EXCEPT !["msg_sender"] = "completed"]
```

For what regards Msg_sender we have the last 3 actions that regard the completation of the procedure, that will clean the local variables and restore the program counter to the beginning of the procedure.

Finally an action to model the failures of the service, here wih a single action i model all possible failures of the service with reinitialization of all the local variables.
```
msg_sender_fail   ==  /\  componentState["msg_sender"] = "ready"  
                            /\  local_dequeueProcesses_processesGet' = {}
                            /\  local_dequeueProcesses_selected' = {}
                            /\  msg_sender_selected' = {}
                            /\  local_submitNewTxs_processes' = {}
                            /\  local_submitNewTxs_updatedProcesses' = {}
                            /\  programCounter' = [programCounter EXCEPT !["msg_sender"] = "start"]
                            /\  componentState' = [componentState EXCEPT !["msg_sender"] = "failed"]
```

And an action of restore that simply change the componentState to `ready`.

The process StalledProcessFinder can be modeled in tha same way as above, there are no particular differences. 

For what regards HandleTimedOutTransactions, there are some novelties with wtha said above.

First we have to model an explicitly defined transaction that span ver the entire procedure, we can doing it simply taking a write lock at the beginning of the procedure and releasing it at the end.
```
handleTimeoutTransactions_get ==    /\  componentState["handleTimeOutTransactions"] = "ready"
                                    /\  programCounter["handleTimeOutTransactions"] = "start"
                                    /\  \E x \in database : x["status"] = "timeout"
                                    /\  table_lock["msg_sender"] = "null" /\ table_lock["stalledProcessFinder"] = "null" /\ table_lock["handleTimeOutTransactions"] = "null"
                                    /\  table_lock' = [table_lock EXCEPT !["handleTimeOutTransactions"] = "write"]
                                    /\  local_handleTimeoutTransactions_getProcesses' = {x \in database: x["status"]="timeout"}
                                    /\  programCounter' = [programCounter EXCEPT !["handleTimeOutTransactions"] = "gotProcesses"]
```

In the following action we model the select of a process (processes here are processed one by one) and its search in the external distributed services.
```
handleTimeoutTransactions_selectAndsearchProcessById   ==   /\  componentState["handleTimeOutTransactions"] = "ready"
                                                            /\  programCounter["handleTimeOutTransactions"] = "processing"
                                                            /\  \E y \in local_handleTimeoutTransactions_recovering : TRUE
                                                            /\  local_handleTimeoutTransactions_process' = CHOOSE x \in local_handleTimeoutTransactions_recovering : TRUE
                                                            /\  local_handleTimeoutTransactions_recovering' = local_handleTimeoutTransactions_recovering \ {local_handleTimeoutTransactions_process'}
                                                            /\  local_handleTimeoutTransactions_foundProcessId' =   IF (\E z \in signedAndBroadcasted : z["id"] = local_handleTimeoutTransactions_process'["id"])
                                                                                                                    THEN TRUE
                                                                                                                    ELSE FALSE
                                                            /\  programCounter' = [programCounter EXCEPT !["handleTimeOutTransactions"] = "finalizeOrReenquee"]
```

And here the end of the cycle:
```
handleTimeoutTransactions_endCycle   == /\  componentState["handleTimeOutTransactions"] = "ready"
                                        /\  programCounter["handleTimeOutTransactions"] = "processing"   
                                        /\  ~(\E y \in local_handleTimeoutTransactions_recovering : TRUE )   
                                        /\  programCounter' = [programCounter EXCEPT !["handleTimeOutTransactions"] = "complete"]
```

Finally the actions for finalize or reenquee the processes:
```
handleTimeoutTransactions_finalizeFound   ==    /\  componentState["handleTimeOutTransactions"] = "ready"
                                                /\  programCounter["handleTimeOutTransactions"] = "finalizeOrReenquee" 
                                                /\  local_handleTimeoutTransactions_foundProcessId = TRUE
                                                /\  local_handleTimeoutTransactions_process' = [local_handleTimeoutTransactions_process EXCEPT !["status"] = "processed", !["txHash"] = "0x53f"]
                                                /\  LET local_database_handleTimeoutTransactions == database
                                                    IN  database' = (database \ ({x \in local_database_handleTimeoutTransactions : x["id"] = local_handleTimeoutTransactions_process["id"]})) \union {local_handleTimeoutTransactions_process'}
                                                /\  programCounter' = [programCounter EXCEPT !["handleTimeOutTransactions"] = "processing"]
```
```                                                       
handleTimeoutTransactions_Reenqueue   ==    /\  componentState["handleTimeOutTransactions"] = "ready"
                                            /\  programCounter["handleTimeOutTransactions"] = "finalizeOrReenquee" 
                                            /\  local_handleTimeoutTransactions_foundProcessId = FALSE
                                            /\  local_handleTimeoutTransactions_process' = [local_handleTimeoutTransactions_process EXCEPT !["status"] = "enqueue"]
                                            /\  LET local_database_handleTimeoutTransactions == database
                                                IN  database' = (database \ ({x \in local_database_handleTimeoutTransactions : x["id"] = local_handleTimeoutTransactions_process["id"]})) \union {local_handleTimeoutTransactions_process'}
                                            /\  programCounter' = [programCounter EXCEPT !["handleTimeOutTransactions"] = "processing"]
```

And here our temporal property: `<>(\E x \in database : x["status"]="processed")`

Finally the spec definition has been done as following:
```
Spec == /\  Init /\ [][Next]_vars 
        /\  WF_vars(InsertProcessInDb)
        /\  WF_vars(dequeueProcesses_get \/ dequeueProcesses_update \/ submitNewTxs_signAndBroadcast \/ submitNewTxs_updateBroadcasted)
        /\  SF_vars(msg_sender_restore)  
        /\  WF_vars(stalledProcessFinder_get \/ stalledProcessFinder_update \/ stalledProcessFinder_complete)
        /\  SF_vars(stalledProcessFinder_restore)
        /\  WF_vars(handleTimeoutTransactions_get \/ handleTimeoutTransactions_updateAndReturn \/ handleTimeoutTransactions_selectAndsearchProcessById \/ handleTimeoutTransactions_endCycle \/ handleTimeoutTransactions_finalizeFound \/ handleTimeoutTransactions_complete \/ handleTimeoutTransactions_Reenqueue)
        /\  SF_vars(handleTimeoutTransactions_restore)
        /\  ~[]<>(componentState["msg_sender"] = "failed" \/ componentState["stalledProcessFinder"] = "failed" \/ componentState["handleTimeOutTransactions"] = "failed")
```     

We have used `strong fairness` only for the restore actions while we have `weak fairness` in the whole process for the 3 services. We have introduced also the fact that from a certain moment on there will be no errors. This for model that there will be an interval in time enough long for the system to process its data without errors. This works becouse our temporal property is of the simple form <>(A), it had been of the form <>[](A) or []<>(A) the assumption would have been too strong.

Here we modeled the interaction with the database with transaction of isolation level `serializable`. What if you want to model other isolation levels?

For `Uncommitted read` no transaction are created in the db, this means that also the single `CRUD` operation cannot be assumed atomic, this mean that a single action can read/write/update only a single row in the db.

For `Committed read` an idea would be to store the items in the db with a field committed an old value(snapshot) of the item, one a process try to read the item if the field committed is not set to true catch the old value. So the read is never blocking, while the write follows the well known rules for achieving a lock.

For `Repeteable read` we can model the scenario using the read and write lock on the single entries of the tables.

Finally for the level `Serializable`, as we have done above, we can use read and write locks on the entire tables of the db.

Even if the above models don't follow the implementation of the isolation level in the various dbs it doen's matter, the important aspect is that they follow the semantic of the isolation levels.

Hope you enjoied the article, if you want to suggest some improvement of the articles or have an exhange privately please contact me via emil or via my twitter profile. If you want to continue reading more posts on usage of TLA+ if you don't have read it already I suggest you [Hillel Wayne blog][hillel-blog]. 

[hillel-blog]: https://www.hillelwayne.com
