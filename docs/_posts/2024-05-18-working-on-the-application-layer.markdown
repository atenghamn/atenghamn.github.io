---
layout: post
title:  "Working on the application layer"
date:   2024-05-18 20:49:00 +0200
categories: arichitecture
---

# Working on the Application layer
In the last post I gave an explanation of how I worked with the domain layer when migrating an webservice for logging reptilemetrics. 
This time we will have a look at how I converted the application layer. But before we do, it can be a good idea to have some general understanding of the difference between the domain layer and application layer.

## Boundries
In the domain layer we defined our domain (no shit...) that means that the domain layers is the place were we define all of our domain objects. For a more in depth description of the difference between entities, value objects and aggregates see the [previous blogpost](https://atenghamn.github.io/arichitecture/2024/04/19/defining-the-domain-layer.html). The application layer on the other hand is were we define our usecases. In this specific context it includes things like register a measurement of reptiles (weight and height), register a feeding or a shedding or fetching a user. The thign that might be a bit confusing at first is that we also define our events in our domain layer like

```
using ReptileTracker.Domain.Abstractions;
namespace ReptileTracker.Domain.Feeding.Events;
public sealed record FeedingCreatedDomainEvent(Guid FeedingId) : IDomainEvent;
```
that    is raised by the domain object
```
using ReptileTracker.Domain.Abstractions;
using ReptileTracker.Domain.Feeding.Events;

namespace ReptileTracker.Domain.Feeding;
public sealed class Feeding : Entity
{
    public Feeding(){}

    public Feeding(Guid id): base(id)
    {
    }

    public Feeding(
        Guid id, 
        Guid reptileId, 
        DateTime date, 
        Amount amount, 
        FoodType foodType, 
        Notes? notes
        ) : base(id)
    {
        ReptileId = reptileId;
        Date = date;
        Amount = amount;
        FoodType = foodType;
        Notes = notes;
    }

    public Guid ReptileId { get; private set; }
    public DateTime Date { get; private set; }
    public Amount Amount { get; private set; }
    public FoodType FoodType { get; private set; }
    public Notes? Notes { get; private set; }

    public static Result<Feeding> Create(Guid reptileId, DateTime date, Amount amount, FoodType foodType, Notes? notes)
    {
        if (date > DateTime.UtcNow)
        {
            return Result.Failure<Feeding>(FeedingErrors.CantSeeIntoTheFuture);
        }

        if (amount.value < 1)
        {
            return Result.Failure<Feeding>(FeedingErrors.AmountNotRight);
        }
        var feeding = new Feeding(Guid.NewGuid(), reptileId, date, amount, foodType, notes);
        feeding.RaiseDomainEvent(new FeedingCreatedDomainEvent(feeding.Id));
        
        return feeding;
    }
}

```

But lets see how we get there :) 

## Working on the application layer

Our business logic for handling the feeding looked like this before 
```

    public async Task<Result<FeedingEvent>> AddFeedingEvent(FeedingEvent feedingEvent, CancellationToken ct)
    {
        try
        {
            await feedingRepository.AddAsync(feedingEvent, ct);
            await feedingRepository.SaveAsync(ct);
            Log.Logger.Debug("Added new feeding event to reptile {ReptileId}", feedingEvent.ReptileId);
            return Result<FeedingEvent>.Success(feedingEvent);
        }
        catch (Exception ex)
        {
            Log.Logger.Error("Failed to add new feeding event to reptile {ReptileId}", feedingEvent.ReptileId);
            return Result<FeedingEvent>.Failure(FeedingErrors.CantSave);
        }
    }
```
It's a simple service that holds a dependency on the repository and uses a result pattern to return a result to the user. The whole flow beeing that a POST request is made on the feeding endpoint which in turn triggers a call to this endpoint that makes a call to the repository who makes a call to the database. Pretty straight forwardad. 

In the new implementation I chose to use the CQRS pattern instead. CQRS -Command Query Responsibility Segregation, is a pattern for seperating the commands (post/put request) and querys (get requests). 

To read more about CQRS [Microsofts documentation](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs) is a good place to start. 

I used [Mediatr](https://github.com/jbogard/MediatR) to create the  command and command handler interfaces. I'm sure there's other packages that does the same thing but Mediatr is the only one I've used and I find it really easy and intuitive to work with.  

```
using MediatR;
using ReptileTracker.Domain.Abstractions;

namespace ReptileTracker.Application.Abstractions.Messaging;

public interface ICommand : IRequest<Result>, IBaseCommand
{
}

public interface ICommand<TResponse> : IRequest<Result<TResponse>>, IBaseCommand
{
}

public interface IBaseCommand
{
}
```

```

using ReptileTracker.Domain.Abstractions;
using MediatR;
using static ReptileTracker.Application.Abstractions.Messaging.ICommand;
namespace ReptileTracker.Application.Abstractions.Messaging;
    public interface ICommandHandler<TCommand> : IRequestHandler<TCommand, Result>
        where TCommand : ICommand
    {
    }

    public interface ICommandHandler<TCommand, TResponse> : IRequestHandler<TCommand, Result<TResponse>>
       where TCommand : ICommand<TResponse>
    {
    }


```
The easiest way of desribing a command is that it's the class that encapsulates the command (write) request. My feeding command looks like this: 

```
using ReptileTracker.Application.Abstractions.Messaging;

namespace ReptileTracker.Application.Feeding.RegisterFeeding;

public record FeedingRegisteredCommand(
    Guid FeedingId,
    Guid ReptileId,
    DateOnly Date, 
    int Amount,
    int FoodType,
    string? Notes
    ) : ICommand<Guid>;
```

The command handler is the request action so in this case it looks like: 

```
  using ReptileTracker.Application.Abstractions.Messaging;
using ReptileTracker.Application.Exceptions;
using ReptileTracker.Domain.Abstractions;
using ReptileTracker.Domain.Reptile;
using ReptileTracker.Domain.Feeding;

namespace ReptileTracker.Application.Feeding.RegisterFeeding;

public class FeedingRegisteredCommandHandler : ICommandHandler<FeedingRegisteredCommand, Guid>
{
    private readonly IFeedingRepository _feedingRepository;
    private readonly IReptileRepository _reptileRepository;
    private readonly IUnitOfWork _unitOfWork;

    public FeedingRegisteredCommandHandler(
        IFeedingRepository feedingRepository, 
        IReptileRepository reptileRepository,
        IUnitOfWork unitOfWork)
    {
        _feedingRepository = feedingRepository;
        _reptileRepository = reptileRepository;
        _unitOfWork = unitOfWork;
    }


    public async Task<Result<Guid>> Handle(FeedingRegisteredCommand request, CancellationToken cancellationToken)
    {
        var reptile = await _reptileRepository.GetByIdAsync(request.ReptileId, cancellationToken);
        if (reptile is null) return Result.Failure<Guid>(ReptileErrors.NotFound);


        try
        {
            var feeding = new Domain.Feeding.Feeding(
                request.FeedingId,
                request.ReptileId,
                request.Date.ToDateTime(TimeOnly.MinValue),
                new Amount(request.Amount),
                (FoodType)request.FoodType,
                new Notes(request.Notes ?? "")
            );
            _feedingRepository.Add(feeding);
            await _unitOfWork.SaveChangesAsync(cancellationToken);

            return feeding.Id;
        }
        catch(ConcurrencyException)
        {
            return Result.Failure<Guid>(FeedingErrors.CouldNotSave);
        }
    }
}
```

As you can see we create a new Feeding object and then save it to the repository and finalyy we return the id.

An extra feature that Mediatr offers is the ability to validare your requests by their AbstractValidator class. 

```
using System.Data;
using System.Security;
using FluentValidation;

namespace ReptileTracker.Application.Feeding.RegisterFeeding;

public class FeedingRegisteredCommandValidator : AbstractValidator<FeedingRegisteredCommand>
{
    public FeedingRegisteredCommandValidator()
    {
        RuleFor(x => x.FeedingId)
            .NotEmpty()
            .WithMessage("FeedingId is required");
        
        
        
        RuleFor(x => x.ReptileId)
            .NotEmpty()
            .WithMessage("ReptileId is required");
       
        RuleFor(x => x.Amount)
            .GreaterThanOrEqualTo(1)
            .NotEmpty()
            .WithMessage("Amount must have a positive value");

        RuleFor(x => x.FoodType)
            .NotEmpty()
            .GreaterThanOrEqualTo(0)
            .WithMessage("Set foodtype");

        RuleFor(x => x.Date)
            .NotEmpty()
            .WithMessage("Must set date");

    }
}

```

This is pretty straight forward using fluent validator and not a optional step to make it work. To make the validator validate we set it up in our pipeline using middleware 

```
using FluentValidation;
using MediatR;
using ReptileTracker.Application.Abstractions.Messaging;
using ReptileTracker.Application.Exceptions;

namespace ReptileTracker.Application.Abstractions.Behaviors;

public class ValidationBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IBaseCommand
{

    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
    {
        _validators = validators;
    }

    public async Task<TResponse> Handle(
        TRequest request, 
        RequestHandlerDelegate<TResponse> next, 
        CancellationToken cancellationToken
    )
    {
        if(!_validators.Any())
        {
            return await next();
        }   

        var context = new ValidationContext<TRequest>(request);

        var validationErrrors = _validators
            .Select(validator => validator.Validate(context))
            .Where(validationResult => validationResult.Errors.Any())
            .SelectMany(validationResult => validationResult.Errors)   
            .Select(validationFailure => new ValidationError(
                validationFailure.PropertyName, 
                validationFailure.ErrorMessage))
            .ToList();

        if (validationErrrors.Any())
        {
            throw new Exceptions.ValidationException(validationErrrors);   
        }

        return await next();

    }
}
```
With this setup we can "scan" the assembly that holds the DependencyInjection class at startup to find all implementations of the IValidator<T> interface

```
        services.AddValidatorsFromAssembly(typeof(DependencyInjection).Assembly);
```

## Summary
Defining our application layer is really about seperating our read and write operations and creating a requst pipeline thats easy to extend upon. The important thing to notice here is that all the dependencys our inward (towards the domain layer). In the next part we will wrap it up with look at the infrastructure and presentation layers to see how they call the application layer. 

Take care! 