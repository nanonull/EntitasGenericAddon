# EntitasGenericAddon
Addon to [Entitas](https://github.com/sschmid/Entitas-CSharp) that allows using generic methods instead of code generator

## Goal
Make Entitas extensible by separate dll

## Main Concepts
  - Code Generator is optional
  - Type inference forces only valid type combinations during development and gives access only to needed methods
  - Interfaces serve as markers for type inference
      - `IScope` - base interface for context scope
      - `Scope<T>` - context scope of IComponent
      - `IComponent` - allows class to be managed by **Entitas**, gives access to `Get<T>` extension methods
      - `ICompData` - gives access to `Add`, `Remove`, `Replace`, `Has` requires `ICopyFrom<TSelf>`
      - `ICompFlag` - gives access to `Flag<T>`, `Is<T>`
      - `IUnique` - provides context `Add`, `Replace` etc methods for unique components or flags

  - Generic Events Feature
      - `IEvent_*<TScope, TComp>` - interface marker for `IComponent` classes
      - `IOn*<TScope, TComp>` - interface to implement by listener classes
      - `EventSystem_*<TScope, TComp>` - event system classes
 
  - Can be used together with regular Entitas components
  - Manual `EntityIndex` registration

## Installation
There are two ways of using EntitasGenericAddon:
  - with existing generated Contexts
    - For now it only adds new generic contexts, generated and generic context instances have different workflows. Improvements are welcome
    - Copy `Entitas.Generic`, `Entitas.Generic.Events` sources into same assembly as generated `Contexts` class
  - standalone without generator
    - Copy `Entitas.Generic`, `Entitas.Generic.Events` sources into `Assets` folder somewhere

## Usage

```csharp
public void Example()
{
    // initialization must be called before new Contexts( ) or accessing Contexts.sharedInstance
    Lookup_ScopeManager.RegisterAll( );

    var contexts = Contexts.sharedInstance;

    // if generator is not used in project
    // var contexts = new Contexts(  );
    // contexts.AddScopedContexts(  );

    var game = contexts.Get<Game>( );

    var entity = game.CreateEntity( );
    entity.Add( Cache<Position>.I.Set(3f, 10f) );
    entity.Replace( new Position( 20f, 1f ) );
    entity.Has<Position>( );
    entity.Remove<Position>( );

    entity.Flag<Moving>( true );
    entity.Is<Moving>( );
    entity.Flag<Moving>( false );

    game.Flag<GameActive>( true );  // provides unique component interfaces
    var gameActiveEntity = game.GetEntity<GameActive>( );
}
```

Scope
```csharp
public interface Game : IScope { }
```

Component
```csharp
public sealed class A : IComponent, ICompData, ICopyFrom<A>, Scope<Game>
{
    public int Value;

    public A Set(int value) // optional, allows using Cache<T>.I.Set()
    {
        Value = value;
        return this;
    }

    public void CopyFrom(A other)
    {
        Value = other.Value;
    }
}
```

FlagComponent
```csharp
public sealed class FlagA : IComponent, ICompFlag, Scope<Game> { }
```

Matcher
```csharp
Matcher<Entity<Game>>
    .AllOf(
        Matcher<Game, Position>.I,
        Matcher<Game, Velocity>.I )
    .AnyOf(
        Matcher<Game, Moving>.I )
    .NoneOf(
        Matcher<Game, Destroy>.I ) );
```

Events
```csharp
// Step 1. Add event markers to components
public sealed class FlagA : IComponent, ICompFlag, Scope<Game>
    , IEvent_Any<Game, FlagA> // <---
{ }
public sealed class B : IComponent, ICompData, ICopyFrom<B>, Scope<Game>
    , IEvent_SelfRemoved<Game, B>  // <---
{
    // some code
}

    
// Step 2. Add event systems to Systems. This step could be automated in future
    systems.Add( new EventSystem_SelfRemoved<Game, B>(  ) );
    systems.Add( new EventSystem_Any<Game, FlagA>(  ) );


// Step 3. Inherit and implement event interface
public class Some
    : MonoBehaviour
    , IOnAny<Game, FlagA>
    , IOnSelfRemoved<Game, B>
{
private void OnAny( FlagA component, Entity<Game> entity, Contexts contexts )
{
    // some code
}

private void OnSelfRemoved( B component, Entity<Game> entity, Contexts contexts )
{
    // some code
}


// Step 4. Add listeners to entity
private void OnEnable()
{
    var contexts = Contexts.sharedInstance;
    var entity = contexts.Get<Game>( ).CreateEntity( );

    entity.Add_OnSelfRemoved( this );

    entity.Add_OnAny( this ); // subscribe
    entity.Remove_OnAny( this ); // unsubscribe

    // writing types explicitly is required when implicit inference is impossible
    entity.Add_OnAny<Game, FlagA>( this );
    entity.Remove_OnAny<Game, FlagA>(  ); // removes listener component
}
}

```

EntityIndex
```csharp
// Step 1(Optional). Create const string key for accessing entity index
public static class EntIndex
{
    public const string B = "B";
}


// Step 2. Add EntityIndex during initialization stage
var context = contexts.Get<Game>( );
context.AddEntityIndex( EntIndex.B
    , context.GetGroup( Matcher<Game, B>.I )
    , ( e, c ) => ( (B)c ).Value );


// Step 3. Get entities at runtime
var entities = context.GetEntities( EntIndex.B, 23 );
```
## FAQ
**Q: What `Cache<T>.I` does?**

A: `Cache<T>.I`creates and reuses static copy of component for passing values to Entitas component through manually created `Component.Set` method. `I` is shortened `Instance`. Check [CacheT.cs](./Entitas.Generic/Lookup/CacheT.cs) for implementation.

**Q: I have better implementation of some interface/feature**

A: Improvements are great! Please write your suggestion in Issues section
