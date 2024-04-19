---
layout: post
title:  "Defining the domain layer"
date:   2024-04-19 23:16:00 +0200
categories: arichitecture
---

As part of my journey to convert my service for tracking Reptile Metrics, I began by defining the domain layer. For those who are not yet acquainted with Clean Architecture, it bears many similarities to Domain-Driven Design (DDD). I recommend reading [this blogpost](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)  before proceeding further.

When defining the domain layer, I began by identifying my aggregates, entities, and value objects. The core idea behind my reptile tracking service is to allow reptile owners to log information about their animals’ eating habits and growth. While it might seem a bit excessive to have an app for this purpose, many reptiles are fascinating creatures with diverse dietary needs. They consume a variety of live animals, such as crickets, mealworms, pinkies, and banana flies. Ensuring a balanced diet for them is easier said than done. Moreover, if you have multiple reptiles, it quickly becomes challenging to remember who ate what and when. The reptile tracker service serves as my simple solution to address this issue.

The distinctions I made was

#### Entities:

- Reptile 
- Feeding 
- Shedding 
- Measurement
- Aggregates:


#### Value Objects:

- FoodType (like crickets, zoophobas etc...)

The aggregate root of my service was the reptile entity. Since everything in the service center around the reptile entity this is the aggregate root. 

An important aspect of creating domain entities is defining rich domain models. My old reptile entity looked like this: 

```
[Table("reptile", Schema = "dbo")]
public class Reptile
{
    [Column("id")]
    [Required]
    [Key]
    public int Id { get; set; }
    
    [Column("name")]
    [StringLength(60)]
    public string Name { get; set; }
    
    [Column("species")]
    [StringLength(60)]
    public string Species { get; set; }
    
    [Column("birthdate")]
    public DateTime Birthdate { get; set; }
    
    [Column("reptile_type")]
    [Required]
    public ReptileType ReptileType { get; set; }
    
    [NotMapped]
    public ICollection<Length> MeasurmentHistory { get; set; }
    
    [NotMapped]
    public ICollection<Weight> WeightHistory { get; set; }
    
    [NotMapped]
    public ICollection<FeedingEvent> FeedingHistory { get; set; }
    
    [NotMapped]
    public ICollection<SheddingEvent> SheddingHistory { get; set; }
    
    [ForeignKey("Account")]
    [Column("account_id")]
    public string AccountId { get; set; }
    
    [NotMapped]
    public Account.Model.Account Account { get; set; }
}

```
This main takeaway here is that it's closly tied with entity framework and [code first](https://www.infoworld.com/article/3712581/demystifying-the-code-first-approach-in-ef-core.html) if I was to change database provider I would also have to check so my mapping to entities worked as expected. 

My Reptile entity in my domain project looks similiar but have some differences.
```
public sealed class Reptile : Entity
{
    public Reptile() { }

    public Reptile(Guid id) : base(id)
    {
    }
    public Reptile(
        Guid id,
        Name name,
        Species species,
        DateTime birthdate,
        ReptileType reptileType,
        Guid user
        ) : base(id)
    {
        Name = name;
        Species = species;
        Birthdate = birthdate;
        ReptileType = reptileType;
        User = user;
    }

    public Name Name { get; private set; }
    public Species Species { get; private set; }
    public DateTime Birthdate { get; private set; }
    public ReptileType ReptileType { get; private set; }
    public ICollection<Guid> MeasurmentHistory { get; private set; }
    public ICollection<Guid> FeedingHistory { get; private set; }
    public ICollection<Guid> SheddingHistory { get; private set; }
    public Guid User { get; set; }

    public static Result<Reptile> Create(
        Name name,
        Species species,
        DateTime birthdate,
        ReptileType reptileType,
        Guid user
        )
    {
        var reptile = new Reptile(Guid.NewGuid(), name, species, birthdate, reptileType, user);
        reptile.RaiseDomainEvent(new ReptileCreatedDomainEvent(reptile.Id));
        return reptile;
    }
}
``` 

As you can see, there are some differences. If we start with what has been removed, the first thing to notice is that the annotations for Entity Framework have been removed. As a step in decoupling our entities from the database implementation, we will now leave the database implementation to the other layers.

Furthermore, we can see that our domain models are a lot richer. Gone are the raw string values like:
```
    public string Name { get; set; }
``` 
and instead we use 
```
    public Name Name { get; private set; }

```
where Name is a record that accepts a string value. 
```
    public record Name(string Value);
```

We can also see that the Reptile entity inherits an abstract class that looks like this: 
```
public abstract class Entity
{
    private readonly List<IDomainEvent> _domainEvents = new();
    protected Entity(Guid id)
    {
        Id = id;
    }
    protected Entity() { }
    public Guid Id { get; init; }
    public IReadOnlyList<IDomainEvent> GetDomainEvents() => _domainEvents.ToList();
    public void ClearDomainEvents() => _domainEvents.Clear();
    protected void RaiseDomainEvent(IDomainEvent domainEvent) => _domainEvents.Add(domainEvent);
}
```
The Entity class is abstract (instead of an interface) so we cant implent methods and I guess most things are pretty self explanatory except for maybe the IDomainEvent part. 

So what the flying f#*k is the IDomainEvent?  The IDomainEvent is my abstraction for working with Mediatr events in the domain layer, just an interface that use Mediatrs INotification interface

```
using MediatR;

namespace ReptileTracker.Domain.Abstractions;
public interface IDomainEvent : INotification
{
}
```

If you haven't heard of [Mediatr](https://github.com/jbogard/MediatR) it's a mediator pattern implementation for .NET  that is easy to use with [CQRS](https://martinfowler.com/bliki/CQRS.html). 

The idea here is that we raise events (we do something  when something happens). For example:  event A occurs and it triggers event B. In our case we have a RaiseDomainEvent and for the Reptile use case we have a ReptileCreatedDomainEvent. If we look at this part: 

```
    public static Result<Reptile> Create(
        Name name,
        Species species,
        DateTime birthdate,
        ReptileType reptileType,
        Guid user
        )
    {
        var reptile = new Reptile(Guid.NewGuid(), name, species, birthdate, reptileType, user);
        reptile.RaiseDomainEvent(new ReptileCreatedDomainEvent(reptile.Id));
        return reptile;
    }
```

we can se that when we create a new reptile we also invoke the RaiseDomainEvent. The ReptileCreatedDomainEvent is nothing more than a sealed record for now:
```
using ReptileTracker.Domain.Abstractions;
namespace ReptileTracker.Domain.Reptile.Events;

public sealed record ReptileCreatedDomainEvent(Guid ReptileId) : IDomainEvent;
```

The important thing to notice here is that we trigger the event when calling the static create method. 

Another part that's worth mentioning is that we work with functional error handling and the [result pattern](https://www.milanjovanovic.tech/blog/functional-error-handling-in-dotnet-with-the-result-pattern). The idea is to have a functional approach and be as expressive as possible. A result class that accepts generic types let us wrap our Reptile entity in a Result. And a simple error interface let us create custom exceptions for each entity 

```
public record Error(string Code, string Name)
{
    public static Error None = new(string.Empty, string.Empty);
    public static Error NullValue = new("Error.NUllValue", "A null value was provided");
}
```

```
using ReptileTracker.Domain.Abstractions;
namespace ReptileTracker.Domain.Reptile;

    public static class ReptileErrors
    {
            public static readonly Error CantSave = new Error("Reptile.CantSave", "Cant add this reptile");
            public static readonly Error CantDelete = new Error("Reptile.CantDelete", "Cant delete this reptile event");
            public static readonly Error NotFound = new Error("Reptile.NotFound", "Reptile not found.");
            public static readonly Error CantUpdate = new Error("Reptile.CantUpdate", "Cant update reptile event");
            public static readonly Error NoReptileHistory = new Error("Reptile.NoHistory", "No reptile history found");
            public static readonly Error DidntFindReptiles = new Error("Reptile.DidntFindReptiles", "Didn't find any reptiles");
            public static readonly Error EventlistNotFound =
                new Error("Reptile.EventlistNotFound", "Cant find the event list");   
    }

```

I used the same pattern before migrating to clean architecture but I think it's worth mentioning anyhow. 

The last part to our domain layer is implementing a repository interface like:
```
namespace ReptileTracker.Domain.Reptile;

public interface IReptileRepository
{
    Task<Reptile?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default);
    void Add(Reptile reptile);

}
```
Hopefully you see that interfaces has a really important role to play here. We strive to keep everything as loosely coupled as poosible. As of now we can really go anyway implementing a MSSQL, Postgres och Azure database.

There's much more in the domain layer but it all follow the same pattern. Have a look at [this branch](https://github.com/atenghamn/ReptileTracker/tree/clean-domain-layer) in my repo to see the rest. 

If you have any feedback, question or just want to bring something up just reach out on github, twitter or email. You find the links in the footer.  

Take care :) 