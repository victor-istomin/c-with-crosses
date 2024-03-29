---
title: "UE 5: Pitfalls of UActorComponent::PostEditChangeProperty"
date:  2023-07-13T10:42:55Z
summary: "Or why my component is TRASH? I mean, suddenly, its name was literally prefixed with 'TRASH_'"
tags: ["c++", "ue", "pitfalls"]
author: "Me"
draft: false
---

I noticed that the `PostEditProperty` behaves weirdly when implemented for my component. That's a debugging story I'd like to share, and a quick conclusion of how to override `UActorComponent::PostEditChangeProperty` correctly.

<details>
<summary>I want it short.
</summary>

An ActorComponent will update in a special way when spawned by a Blueprint. For example, when the component has been added to an Actor using BP editor. Just in case the reader wants to jump straight into the explanation, there is a [conclusion](#conclusion-how-does-the-editor-change-a-property-of-a-_blueprint-constructed_-component) section below. 

</details>


<div style="max-width: 500px;">

![Necromancer by Dawe Lowe Design](images/NECROmancerDLOWE.jpg "Necromancer by Dawe Lowe Design")
<div style="text-align: right"> 

Image Credit: [DAVE LOWE DESIGN](https://davelowedesign.com/)
</div>
</div>

_The Necromancer can make use of the Dead. You should not. At least, not in C++_
<br>

# The Problem

Let's assume there is a Scene Component that represents a stack of laser emitters. It is a reusable component for laster gates, player detectors, and maybe some evil trap. 
<video controls>
    <source src="images/gates-3000.webm" type="video/webm">
    <source src="images/gates-3000.mp4" type="video/mp4">
    Your browser does not support the video tag.
</video>

_-- This one_

The implementation is divided into C++ and Blueprint parts: 
* Base C++ class for the core logic: tick updates, game logic, etc.
* The derived Blueprint class holds UX-related entities: mesh, materials, effects, and color setup.

As the top-level class is a Blueprint, it's added to an actor by the Editor, making it a Blueprint-spawned component. It is a crucial detail.

Eventually, I wanted the component to reflect in-editor changes by updating its appearance. For instance, update the editor-only laser preview mesh when the Editor changes the default laser length property. 

The well-known way to do this is to override the `UActorComponent::PostEditChangeProperty` method. Having a decent C++ experience and several examples, this looks trivial, so I crafted the following code.

Hopefully, myself-from-the-future visited my PC and added an `ensure()` and some hints to guard us agains the sleep-deprived night and several hours of debugging.

{{< highlight cpp "linenos=table,hl_lines=5-6">}}
#if WITH_EDITOR
void UEmitterStackComponent::PostEditChangeProperty(FPropertyChangedEvent& event)
{
    ensure(IsValid(this));                       // Fine
    Super::PostEditChangeProperty(event);        // What could go wrong?
    ensure(IsValid(this));                       // Ouch!..
    InitilizeEmitters();                         // I believed it will work... :(
}
#endif
{{< /highlight >}}

This code triggers `ensure()` failure on line #6. Also, `Super::PostEditChangeProperty(event)` call prefixes the object's name with the "TRASH_" prefix, hinting about the wrong usage. Perhaps, the `UActorComponent::PostEditChangeProperty` replaces this component with a new one, marking the initial component as pending kill.

## The second attempt

Okay, since `UActorComponent::PostEditChangeProperty` re-create the component, let's make changes before `Super::PostEditChangeProperty(event);` 

It will work, won't it?
{{< highlight cpp >}}
void UEmitterStackComponent::PostEditChangeProperty(FPropertyChangedEvent& event)
{
    TestUproperty += 1;       // public: UPROPERTY() int TestProperty = 0;
    UE_LOG(LogTemp, Warning,
           TEXT("PostEditChangeProperty: EmitterStackComponent->Test: %d"),
           TestUproperty);

    ensure(IsValid(this));                // should be alive
    InitilizeEmitters();                  // please work
    Super::PostEditChangeProperty(event);
}
{{< /highlight >}}

I badly hoped it will work, and UE, like a genie, has granted my wish: it works! But in an unexpected way: for instance, `TestUproperty` is always 1, i.e. the `TestUproperty` is never propagated to the updated component. You know, making a wish, I should’ve ensured my requirements are unambiguous.

## Debugging the property change

Since guesswork did not help, let's grab epic 65 GiB of the engine debugging symbols using Launcher and dig into the UE source. 

Start the debugger, load the level, then add some breakpoints: 
 * Line #10: `Super::PostEditChangeProperty(event)` 
 * On the `UEmitterStackComponent::UEmitterStackComponent()` constructor, since we suspect a new component creation.

Now change one of UEmitterStackComponent’s properties in the editor and observe the breakpoint on the component constructor during the `UActorComponent::PostEditChangeProperty` that confirms re-creaton of the component. Let's take a look at the upside-down call stack, using a `Caller -> Callee` denotation for clarity:

{{< highlight cpp>}}
UEmitterStackComponent::PostEditChangeProperty   
-> USceneComponent::PostEditChangeProperty       
  -> UActorComponent::ConsolidatedPostEditChange 
     -> AActor::RerunConstructionScripts         
       -> AActor::ExecuteConstruction            
         -> // If this actor has a blueprint lineage, go ahead and 
            // run the construction scripts from least derived to most
            -> ...
               -> StaticDuplicateObjectEx        
                 -> ...
                    -> UEmitterStackComponent::UEmitterStackComponent()
{{< /highlight >}}

But why didn’t the `TestUproperty` update?

It turns out that UE C++ internals duplicated the <abbr title="Class Default Object">CDO</abbr> rather than the initial object and then set it up with edited properties. I think the reason is that it’s a standard way of BP component creation.

Further, two questions arise:
1. How does the Editor copy non-default property values into a new object?
2. Why doesn’t it copy the changed property during the `UEmitterStackComponent::PostEditChangeProperty` execution?

## The way Editor duplicates an object preserving edited properties

Using a data breakpoint (perhaps, I could describe this technique in a future article), we could find where some property has actually changed.

Long story short, `AActor::ExecuteConstruction()` deserialize the previously-cached data into a new instance of the duplicated object before running its construction script.

{{< highlight cpp "hl_lines=14">}}
// Call-stack (caller is below callee):
// > AActor::ExecuteConstruction                 Line 826
//   AActor::RerunConstructionScripts            Line 536
//   UActorComponent::ConsolidatedPostEditChange Line 963
//   USceneComponent::PostEditChangeProperty     Line 541

bool AActor::ExecuteConstruction(/*...*/, const FComponentInstanceDataCache* InstanceDataCache, /*...*/)
{
    // ...

    // If we passed in cached data, we apply it now, so that the UserConstructionScript can use the updated values
    if (InstanceDataCache)
    {
      InstanceDataCache->ApplyToActor(this, ECacheApplyPhase::PostSimpleConstructionScript);
    }
}
{{< /highlight >}}

Let’s find the code that gathers the `InstanceDataCache` and look at whether it's before or after our  `UEmitterStackComponent::PostEditChangeProperty` handler execution. `InstanceDataCache` tracks down to the `FActorTransactionAnnotation` structure that is created by several code places, so we'll take advantage of those 65 GiB of <abbr title="Program DataBase (debugging symbols)">PDBs</abbr> and place another breakpoint in the `FActorTransactionAnnotation::FActorTransactionAnnotation(const AActor* InActor, /*...*/)`. 

Change the property once more, and look at the insight: 
{{< highlight cpp "hl_lines=24 26">}}
// Call-stack (caller is below callee):
//   FActorTransactionAnnotation::FActorTransactionAnnotation Line 392    C++
// ...
//   SaveToTransactionBuffer        Line 2977
//   UObject::Modify                Line 1277
//   AActor::Modify                 Line 1655
// ... 
// > UActorComponent::Modify        Line 823
//   UActorComponent::PreEditChange Line 834
//   UObject::PreEditChange         Line 404

bool UActorComponent::Modify( bool bAlwaysMarkDirty/*=true*/ )
{
  AActor* MyOwner = GetOwner();
    
  // Components in transient actors should never mark the package as dirty
  bAlwaysMarkDirty = bAlwaysMarkDirty && (!MyOwner || !MyOwner->HasAnyFlags(RF_Transient));

  // If this is a construction script component we don't store them in the transaction buffer.  Instead, mark
  // the Actor as modified so that we store of the transaction annotation that has the component properties stashed
  if (MyOwner)
  {
    extern int32 GExperimentalAllowPerInstanceChildActorProperties;
    if (IsCreatedByConstructionScript() || (GExperimentalAllowPerInstanceChildActorProperties && MyOwner->IsChildActor()))
    {
      return MyOwner->Modify(bAlwaysMarkDirty);
    }
  }

  return Super::Modify(bAlwaysMarkDirty);
}
{{< /highlight >}}

Thus, the main problem is that the Editor snapshots our Actor before the editing starts, alters the edited property, and creates a new component using the snapshot. Other side-effect changes aren't "replicated" into it.

Let's summarize the findings. 

## Conclusion: how does the Editor change a property of a _blueprint-constructed_ component

`FPropertyValueImpl::ImportText` will do these steps in order:
1. `UActorComponent::PreEditChange` that will mark the parent object as dirty and save its state; 
2. `FPropertyValueImpl::ImportText` will update both the edited object __and its saved state__;
3. If editing CDO, propagate the changes to instances;
4. Make `FPropertyChangedEvent`, call `PostEditChangeProperty`. Regarding the `UActorComponent::PostEditChangeProperty`, it will do the following:

    4.1.  `AActor::ExecuteConstruction`: will create new BP-based components using CDO;
    
    4.2.  `AActor::ExecuteConstruction`: will restore newly created BP-based components field values using the cached data from step #2, including the property that has triggered the change.

5. Mark the initial component as pending kill, because it's no longer needed.  

Thus, the BP-created component will lose any changes the `MyComponent::PostEditChangeProperty` made. Also, `UActorComponent::PostEditChangeProperty` may mark it as a pending kill.

# Alternate initialization callbacks

Since `UActorComponent::PostEditChangeProperty` is tricky to use correctly for a BP-spawned component, a developer may consider refactoring the code to use other initialization callbacks instead.

Their execution order for the described case may help reasoning on which to use: 

{{< highlight cpp "hl_lines=4 10">}}
// 'source' means an old component that existed prior to the propertt change
// 'newComponent' means a new instance that will replace the 'source' after 
// BP construction scripts execution
source->PostEditChangeProperty(); // may mark 'source' as pending kill
ReRegister()->source->OnRegister(); 
newComponent->constructor;
newComponent->PostInitProperties();
newComponent->OnComponentCreated();
newComponent->OnRegister();
ApplyToActor(cache)->newComponent->properties restored;
newComponent->OnRegister(); // if `UActorComponent::bAllowReregistration` (by default)
newComponent->PostApplyToComponent();
{{< /highlight >}}

There are two functions called after properties restoring:  
 * UActorComponent::OnRegister - the pitfall is that it’s called three times with no clear indication of whether the properties are in an intermediate state. Thus 'OnRegister/OnUnregister' pair may be used to update the editor-only state, but the debugging may be intricate. 
 * UActorComponent::PostApplyToComponent() is never called during a normal initialization, even in the Editor, so its only purpose seems to be the described case. For me, it looks rather like a workaround than a solution. 
 * Other functions are called before the restoration of properties, thus the only use case for our case is to schedule an async update on the next frame. I think, it's an acceptable solution for the Editor functionality, but it's not ideal.  

# My way of handling property change by the Editor

* Treat the `Component::PostEditChangeProperty` like a read-only method that could only send some notification or trigger an async callback _on the owner actor_ to perform necessary updates on the next Editor tick for a new instance. 
* Use another change callback. I refactored my code to work fine with the `PostInitProperties` callback.
* Spawn the component using C++ by `CreateDefaultSubobject` to avoid the construction script re-run. Not the best idea, in my opinion, because I think the reusable component's method implementation should not depend on the means of construction. Eventually, misuse will occur.
  * At least, I'd place `ensureMsgf(IsValid(this), ...)` with a clear message at the end of the overridden `PostEditChangeProperty` to alert the developer once a Blueprint-created version occurs. 

## PostInitProperties alternative example

My final solution looks like this:

{{< highlight cpp>}}
void UEmitterStackComponent::PostInitProperties()
{
    Super::PostInitProperties();

    // postpone preload if laser is not immediately engaged
    if (LaserProperties.InitialState != FLaserEmitterState::Inactive)
        AsyncPreload();

    // Postpone emitters initialization because children components may 
    // still be uninitialized. In game world, initialization will 
    // occur on BeginPlay(), in editor it will be postponed for the next tick.
    // AActorComponent::PostEditChangeProperty() does not help, because it 
    // re-creates BP component instance thus making a new component object.
#if WITH_EDITOR
    if (UWorld* world = GetWorld(); 
        IsValid(world) && !world->IsGameWorld() && !IsTemplate())
    {
        // in editor, it's fine to postpone the initialization for the next tick, 
        // until all children emitters will be initialized for sure
        world->GetTimerManager().SetTimerForNextTick([this]{this->InitilizeEmitters();});
    }
#endif
}{{< /highlight >}}

Hope, this helps someone and sheds some light on the pitfalls of `PostEditChangeProperty` overriding for component classes.

# Reddit discussion

[Is here.](https://www.reddit.com/r/unrealengine/comments/14yzisc/ue_5_pitfalls_of/)

# Updates
 * Replaced deprecated `ensure(!IsPendingKill())` with `ensure(IsValid())` - my fault translating thoughts into the code.
 * Added a chapter on [initialization callbacks order](#alternate-initialization-callbacks).
