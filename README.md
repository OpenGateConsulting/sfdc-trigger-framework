# SFDC trigger framework

[![npm version](https://badge.fury.io/js/sfdc-trigger-framework.svg)](https://badge.fury.io/js/sfdc-trigger-framework)
[![Maintainability](https://api.codeclimate.com/v1/badges/eeeae5a492e34feace99/maintainability)](https://codeclimate.com/github/kevinohara80/sfdc-trigger-framework/maintainability)

Fork of Kevin O'Hara's trigger framework to apply updates/fixes on a more frequent basis.

## Overview

Triggers should (IMO) be logicless. Putting logic into your triggers creates un-testable, difficult-to-maintain code. It's widely accepted that a best-practice is to move trigger logic into a handler class.

This trigger framework bundles a single **TriggerHandler** base class that you can inherit from in all of your trigger handlers. The base class includes context-specific methods that are automatically called when a trigger is executed.

The base class also provides a secondary role as a supervisor for Trigger execution. It acts like a watchdog, monitoring trigger activity and providing an api for controlling certain aspects of execution and control flow.

But the most important part of this framework is that it's minimal and simple to use. 

**Deploy to Salesforce Org:**
[![Deploy](https://raw.githubusercontent.com/afawcett/githubsfdeploy/master/deploy.png)](https://githubsfdeploy.herokuapp.com/?owner=OpenGateConsulting&repo=sfdc-trigger-framework&ref=master)

## Usage

To create a trigger handler, you simply need to create a class that inherits from **TriggerHandler.cls**. Here is an example for creating an Opportunity trigger handler.

```java
public class OpportunityTriggerHandler extends TriggerHandler {
```

In your trigger handler, to add logic to any of the trigger contexts, you only need to override them in your trigger handler. Here is how we would add logic to a `beforeUpdate` trigger.

```java
public class OpportunityTriggerHandler extends TriggerHandler {
  
  /* Optional Constructor - better performance */
  public OpportunityTriggerHandler(){
    super('OpportunityTriggerHandler');
  }
  
  public override void beforeUpdate() {
    for(Opportunity o : (List<Opportunity>) Trigger.new) {
      // do something
    }
  }

  // add overrides for other contexts

}
```

**Note:** When referencing the Trigger statics within a class, SObjects are returned versus SObject subclasses like Opportunity, Account, etc. This means that you must cast when you reference them in your trigger handler. You could do this in your constructor if you wanted. 

```java
public class OpportunityTriggerHandler extends TriggerHandler {

  private Map<Id, Opportunity> newOppMap;

  /* Optional Constructor - better performance */
  public OpportunityTriggerHandler(){
    super('OpportunityTriggerHandler');
    this.newOppMap = (Map<Id, Opportunity>) Trigger.newMap;
  }
  
  public override void afterUpdate() {
    //
  }

}
```

To use the trigger handler, you only need to construct an instance of your trigger handler within the trigger handler itself and call the `run()` method. Here is an example of the Opportunity trigger.

```java
trigger OpportunityTrigger on Opportunity (before insert, before update) {
  new OpportunityTriggerHandler().run();
}
```

## Cool Stuff

### Check Records That Have Been Processed

To prevent re-running trigger logic on the same record, you can call a static method to get record Ids that HAVE NOT been processed by your logic.

```java
public class OpportunityTriggerHandler extends TriggerHandler {
    private Map<Id, Opportunity> oldOpportunityMap;
    private Map<Id, Opportunity> newOpportunityMap;
    private List<Opportunity> newOpportunityList;
    private List<Opportunity> oldOpportunityList;
    
  /* Optional Constructor - better performance */
  public OpportunityTriggerHandler(){
    super('OpportunityTriggerHandler');
    this.oldOpportunityMap = (Map<Id, Opportunity>) Trigger.oldMap;
    this.newOpportunityMap = (Map<Id, Opportunity>) Trigger.newMap;
    this.oldOpportunityList = (List<Opportunity>) Trigger.old;
    this.newOpportunityList = (List<Opportunity>) Trigger.new;
  }
  
  public override void afterUpdate() {
    for (Id oppId : getIdsThatHaveNotBeenProcessed('opportunityAfterUpdate', newOpportunityMap.values())) {
        //this will now iterate over the ids that have not been processed
        //if this trigger is called again, the static map in the trigger handler 
        //will not return any ids already added from the previous call        
    }
  }

}
```

### Max Loop Count

To prevent recursion, you can set a max loop count for Trigger Handler. If this max is exceeded, and exception will be thrown. A great use case is when you want to ensure that your trigger runs once and only once within a single execution. Example:

```java
public class OpportunityTriggerHandler extends TriggerHandler {

  /* Optional Constructor - better performance */
  public OpportunityTriggerHandler(){
    super('OpportunityTriggerHandler');
    this.setMaxLoopCount(1);
  }
  
  public override void afterUpdate() {
    List<Opportunity> opps = [SELECT Id FROM Opportunity WHERE Id IN :Trigger.newMap.keySet()];
    update opps; // this will throw after this update
  }

}
```

### Bypass API

What if you want to tell other trigger handlers to halt execution? That's easy with the bypass api:

```java
public class OpportunityTriggerHandler extends TriggerHandler {
  
  /* Optional Constructor - better performance */
  public OpportunityTriggerHandler(){
    super('OpportunityTriggerHandler');
  }
  
  public override void afterUpdate() {
    List<Opportunity> opps = [SELECT Id, AccountId FROM Opportunity WHERE Id IN :Trigger.newMap.keySet()];
    
    Account acc = [SELECT Id, Name FROM Account WHERE Id = :opps.get(0).AccountId];

    TriggerHandler.bypass('AccountTriggerHandler');

    acc.Name = 'No Trigger';
    update acc; // won't invoke the AccountTriggerHandler

    TriggerHandler.clearBypass('AccountTriggerHandler');

    acc.Name = 'With Trigger';
    update acc; // will invoke the AccountTriggerHandler

  }

}
```

If you need to check if a handler is bypassed, use the `isBypassed` method:

```java
if (TriggerHandler.isBypassed('AccountTriggerHandler')) {
  // ... do something if the Account trigger handler is bypassed!
}
```

If you want to clear all bypasses for the transaction, simple use the `clearAllBypasses` method, as in:

```java
// ... done with bypasses!

TriggerHandler.clearAllBypasses();

// ... now handlers won't be ignored!
```

## Overridable Methods

Here are all of the methods that you can override. All of the context possibilities are supported.

* `beforeInsert()`
* `beforeUpdate()`
* `beforeDelete()`
* `afterInsert()`
* `afterUpdate()`
* `afterDelete()`
* `afterUndelete()`

## Sample Trigger and Trigger Handler

Use these as a starting point for your trigger. Remove any trigger contexts or override methods not needed.

### Trigger Example
```java
trigger OpportunityTrigger on Opportunity (before insert, before update, before delete, before undelete, after insert, after update, after delete, after undelete ) {
   new OpportunityTriggerHandler().run();
}
```
### Trigger Handler Example
```java
public class OpportunityTriggerHandler extends TriggerHandler {
  private Map<Id, Opportunity> oldOpportunityMap;
  private Map<Id, Opportunity> newOpportunityMap;
  private List<Opportunity> newOpportunityList;
  private List<Opportunity> oldOpportunityList;
    
  public OpportunityTriggerHandler(){
    super('OpportunityTriggerHandler');
    this.oldOpportunityMap = (Map<Id, Opportunity>) Trigger.oldMap;
    this.newOpportunityMap = (Map<Id, Opportunity>) Trigger.newMap;
    this.oldOpportunityList = (List<Opportunity>) Trigger.old;
    this.newOpportunityList = (List<Opportunity>) Trigger.new;
  }
  public override void beforeInsert() {
  }
  public override void beforeUpdate() {
  }
  public override void beforeDelete() {
  }
  public override void afterInsert() {
  }
  public override void afterUpdate() {
  }
  public override void afterDelete() {
  }
  public override void afterUndelete() {
  }

}
```
