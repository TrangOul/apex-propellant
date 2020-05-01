# Apex Propellant

An _Elegant Object_ oriented alternative solution to Apex trigger handling. It's not rocket science, I promise.

```
       ^
      / \
     /___\
    |=   =|
    |  _  |
    | (_) |
    |  _  |
    | (_) |
   /|     |\   
  / |=   =| \  
 /  |##!##|  \ 
|  /|##!##|\  |
| / |##!##| \ |
|/   ^ | ^   \|
     ( | )
    ((   ))
   (( pro ))
   (( pel ))
   (( lan ))
    (( t ))
     (( ))
      ( )
       .
       .
```
**Deploy to SFDX Scratch Org:**
[![Deploy to SFDX](https://deploy-to-sfdx.com/dist/assets/images/DeployToSFDX.svg)](https://deploy-to-sfdx.com)

**Deploy to Salesforce Org:**
[![Deploy to Salesforce](https://raw.githubusercontent.com/afawcett/githubsfdeploy/master/deploy.png)](https://githubsfdeploy.herokuapp.com/?owner=berardo&repo=apex-propellant&ref=master)


## Overview

It's well known that triggers should be logicless, which means we should avoid coding directly within them. Instead, it's advisable to make calls to business classes not rarely called trigger handlers.

However, what's commonly overlooked is the dependency from the handler back to the Apex `Trigger` object.

The Apex Propellant library is inspired by a popular solution called [`TriggerHandler`](https://github.com/kevinohara80/sfdc-trigger-framework), a common generic class that can be extended to override methods like `beforeInsert()` or `afterUpdate()`, so that these methods are executed when triggers call something like:

```java
new MyBeautifulClassThatExtendsTriggerHandler().run();
```

However, while I agree this is an easy way to shift complex logics away from triggers, it doesn't bring much benefit as the handler still needs to deal with `Trigger.new`, `Trigger.old` and so on. This unwated dependency makes handler as not unit testable as triggers themselves.

On the other hand, **Apex Propellant** is a simple alternative targeting the same goal but also:

- Decoupling handlers from triggers and `System.Trigger`, allowing them to be tested in isolation
- Promoting an easy, truly object oriented API
- Promoting the usage of [immutable objects](https://en.wikipedia.org/wiki/Immutable_object)
- Giving you control over handler call repetitions and bypasses

## How it works

The intention is to make the analogy speak for itself.
You have a trigger and you aim to get your rocket code flying around the space every time your trigger is ... triggered.

That being said, as a real-world rocket, when your code takes off, it should not care about anything that was left off on the ground. All the data that you need to decide what to do and where to go should be there with you.

### Show me the code!

First things first, create a trigger and build your rocket:

```java
trigger AccountTrigger on Account(before insert) {
  OnBeforeRocket rocket = new MyAmazingRocket();
}
```

In order to reach the stars, you also need a Propellant, which is the fuel and artifact used to reach what's called the [_Escape Velocity_](https://en.wikipedia.org/wiki/Escape_velocity). Therefore, your very next step is to build a `Propellant` object then fire it.

```java
trigger AccountTrigger on Account(before insert) {
  OnBeforeRocket rocket = new MyAmazingRocket(); // 5, 4, 3, 2, 1 ...
  new Propellant(rocket).fireOff(); // Can you hear the sound? 🚀
}
```

### How should my rocket look like?

The bare minimum code you write to make the trigger above throw your rocket up in the space is like below:

```java
public class MyAmazingRocket extends OnInsertRocket {
  public void flyOnBefore() {
    System.debug('Yay, my 🚀 has gone through the ⛅️ on its way to the ✨');
  }
}
```

The `OnInsertRocket` is an abstract class that implements the two important interfaces `OnBeforeRocket` and `OnAfterRocket` but leave them alone for now.

The `flyOnBefore()` method is the only one called here because your trigger is `before insert` only. Propellant knows that and handles that for you, don't worry.

Would that mean I could have done `flyOnAfter()` and expect it to fly on `after trigger`?

Hey, you're smart! ... and you're right!

Alternatively, you could do this:

```java
public class MyAmazingRocket implements OnBeforeRocket {
  public void flyOnBefore() {
    System.debug('Yay, my 🚀 has gone through the ⛅️ on its way to the ✨');
  }
  public Boolean canTakeOff(TriggerOperation triggerWhen, Propellant propellant) {
    return true;
  }
}
```
Like I said before, even though the previous example introduced an abstract class, you ended up writing slightly less than now.

In order to fly on `before insert` trigger, your rocket only needs to implement `OnBeforeRocket`, which requires you to implement your proper flight (`flyOnBefore`) and another method that I'm gonna explain on the next section. Bear with me!

You might be asking (btw, you ask a lot, I like that 😍!), what about `after insert`?

Easy:
```java
public class MyAmazingRocket implements OnAfterRocket {
  public void flyOnAfter() {
    System.debug('Yay, my 🚀 is reaching the 🌚');
  }
  public Boolean canTakeOff(TriggerOperation triggerWhen, Propellant propellant) {
    return true;
  }
}
```

I know, I know ... you can have both:
```java
public class MyAmazingRocket implements OnAfterRocket {
  public void flyOnBefore() {
    System.debug('Yay, my 🚀 has gone through the ⛅️ on its way to the ✨');
  }
  public void flyOnAfter() {
    System.debug('Yay, my 🚀 is reaching the 🌚');
  }
  public Boolean canTakeOff(TriggerOperation triggerWhen, Propellant propellant) {
    return true;
  }
}
```
There's only one little thing to change when your rocket wants to take off twice in a single transaction (on before and on after triggers), you need to let `Propellant` know. Again, it's so easy:

```java
trigger AccountTrigger on Account(before insert, after insert) {
  OnBeforeRocket rocket = new MyAmazingRocket(); // 5, 4, 3, 2, 1 ...
  new Propellant(rocket, rocket).fireOff(); // Can you hear the sound? 🚀
}
```
If you didn't spot the difference, now we passed the same rocket instance twice.

I think you got the point. So let's move on to the Rocket hierarchy and finally expain the little `canTakeOff` kid.

## The Rocket Hierarchy

There's one generic interface called `Rocket` that exposes what every rocket needs to do. Fortunatelly it's just one single method:
```java
public interface Rocket {
  Boolean canTakeOff(TriggerOperation triggerWhen, Propellant propellant);
}

```
`canTakeOff` gives you the hability to decide whether your rocket will fly or not. The `Propellant` object always asks whether your rocket is ready or not and passes a `System.TriggerOperation` representing the trigger moment, and also its own state in case you need it (we'll see more on this further down when we discuss the rocket `Tank`).

On the trigger example above, `Propellant` would run something like this: `rocket.canTakeOff(TriggerOperation.BEFORE_INSERT, this);` and would move on only if this call returns `true`.

But what about the two methods (`flyOnBefore` and `flyOnAfter`)?
They come from respectively two subinterfaces of `Rocket`: `OnBeforeRocket` and `OnAfterRocket`.

![Rocket interface hierarchy](images/Rocket-Interfaces.png)

You see? That's where those methods come from.

As you can imagine, the class `OnInsertRocket`, used on the very first implementation example, has `canTakeOff` already implemented for you, giving the propeller green light to fire it off on `BEFORE_INSERT`. So that, if the trigger is for example `before insert, after insert, before update`, subclassing `OnInsertRocket`, would leave your rocket on the ground when the targeted record is being updated. Makes sense, right?

Conversely, when you directly implement the two interfaces, there's no Insert/Update/Delete/Undelete context involved. You're free to decide. Can you now see why `canTakeOff` receives a `TriggerOperation`?

### Abstract classes

There are four abstract classes that implement the two `Rocket` interfaces to allow rockets to fly on all trigger operations.

The image below presents those classes so feel free to extend them to create your own rockets.

![Rocket interface hierarchy](images/Rocket-Interfaces-Abstracts.png)

The table below summarises what operations are achieved by what classes.

| Operation  | TriggerOperations   | Rocket class       | Rocket Interface |
|------------|---------------------|--------------------|------------------|
| `insert`   | `BEFORE_INSERT`     | `OnInsertRocket`   | OnBeforeRocket   |
|            | `AFTER_INSERT`      |                    | OnAfterRocket    |
| `update`   | `BEFORE_UPDATE`     | `OnUpdateRocket`   | OnBeforeRocket   |
|            | `AFTER_UPDATE`      |                    | OnAfterRocket    |
| `delete`   | `BEFORE_DELETE`     | `OnDeleteRocket`   | OnBeforeRocket   |
|            | `AFTER_DELETE`      |                    | OnAfterRocket    |
| `undelete` | `AFTER_UNDELETE`    | `OnUndeleteRocket` | OnAfterRocket    |


Before and after operations are always part of the same transaction, hence making them behaviour of the same object most of the time makes total sense. However, there's nothing stopping you from having two separate classes and fire two completely distinct rockets.

When it comes to DML operations though, the likelyhood of having one single object handling operations across four distinct operations is arguably lower. I've witnessed fairly common applications of _TriggerHandler_ with a clear problem of separation of concerns as it all starts from a class where you are able to override methods for all DML operations, which is rarely applicable.

That being said, what's not rare is the fair intention to run the exact same behaviour across different DML operation, like on before insert and before update, for example. If this is your case, unfortunately you cannot extend two classes, so, in this case, the main interfaces are the only way to go.

## Trigger Payload

You might be missing the collections of saved or deleted records found in `Trigger.new`, `Trigger.old`, `Trigger.newMap`, and `Trigger.oldMap`. Well, when you directly implement the interfaces, it's totally up to you. But when you extend an abstract class, you can pass them up through its constructor, like below:

```java
public class MyAmazingRocket extends OnInsertRocket {
  public MyAmazingRocket(Set<SObject> newSet) {
    super(newSet);
  }
  public void flyOnBefore() {
    System.debug('Yay, my 🚀 has gone through the ⛅️ on its way to the ✨');
    System.debug('See my first record ID: ' + this.newSet.get(0).Id);
  }
}
```

Once you pass the argument through `super(newSet)`, you can retrive the information back using the internal attribute `newSet`.

* Using `super(Set<SObject>)` you have `this.newSet`
* Using `super(Set<SObject>, Set<SObject)` you have respectively `this.newSet` and `this.oldSet`
* Using `super(Map<ID, SObject>)` you have `this.newMap`
* Using `super(Map<ID, SObject>, Map<ID, SObject>)` you have respectively `this.newMap` and `this.oldMap`

Of course you need set everything up on your trigger:

```java
trigger AccountTrigger on Account(before insert) {
  OnBeforeRocket rocket = new MyAmazingRocket(Trigger.new);
}
```

Although it might seem I left this burden on your shoulders, as I could have accessed `Trigger.*` within the abstract classes, I can't stress enough how bad this practice would be. Think on a rocket taking off hooked to an anchor left on the ground.

New and old sets and maps belong to the trigger, and having an object with explicit dependency on `System.Trigger` makes it completely not unit testable as the only way to have something like `Trigger.new` populated it by saving a record.

**Remember:** Every time you need to store data before testing your code, you are **NOT** unit testing, you are actually running an integration test. Moreover, despite the importance of integration tests, you code should always be tested in isolation, which is the soul of unit tests.

