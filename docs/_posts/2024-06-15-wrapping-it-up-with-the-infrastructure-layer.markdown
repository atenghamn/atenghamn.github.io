---
layout: post
title:  "Wrapping it up with the infrastructure layer"
date:   2024-06-15 13:25:00 +0200
categories: arichitecture
---

Now the time has come to wrap up the refactoring journey where we took an existing service and refactored it into an clean architechture design. 

If you want to read the previous posts you can find them here:
[Overview](https://atenghamn.github.io/arichitecture/2024/04/16/migrating-to-a-clean-architecture.html)

[Domain layer](https://atenghamn.github.io/arichitecture/2024/04/19/defining-the-domain-layer.html)

[Application layer](https://atenghamn.github.io/arichitecture/2024/05/18/working-on-the-application-layer.html)

## What is the infrastructure layer? 
To understand the role of the infrastructure we need to have an understanding of the domain layer and infrastructure layer. If you dont, please read the previuos articles. 

The fast forward version is that the domain layer is where we store our domain entities and keep our business logic. In other words, this is the heart and brains of our application. The application layer is what handles our communication between our domain and infrastructure layer. This is our transport layer that transports messages to and from our domain layer, usually with CQRS. 

That leave us with todays subject: the infrastructure layer. The infrastructure layer is the layer that's responisible for communicating with the outside world and handle all of our techinical matters. To narrow it down a bit I think it's a good idea to look at some of these features tha the infrastructure layer can handle. We are going to look on how we can handle database connenction and dependency injections in our infrastructure layer. There are of course many other use cases for an infrastructure layer like caching,  authentication and authorization but to narrow it down a bit we're going to stick with these three. 

## Database connections
In my service I use postgres database that runs in a docker container. In my infrastructure layer I use IDbConnection to create a NpgsqlConnection (nuget package for connectoin with postgres databases) like this 

```
internal sealed class SqlConnectionFactory : ISqlConnectionFactory
{
    private readonly string _connectionString;

    public SqlConnectionFactory(string connectionString)
    {
        _connectionString = connectionString;
    }
    public IDbConnection CreateConnection()
    {
        var connection = new NpgsqlConnection(_connectionString);
        connection.Open();

        return connection;
    }
}
```
As we will see later we can call htis factory and supply it with a connection string to so it can connect to our database.  Our infrastructure layer also holds our migrations (I use Entity Framework Core) and all repositorys. 

```
internal sealed class MeasurementRepository : Repository<Measurement>, IMeasurementRepository
{
    public MeasurementRepository(ApplicationDbContext dbContext) : base(dbContext)
    {
    }
}
```

These repositorys implements the interfaces from the domainlayer. 

Finally we have the Application DbContext.

```
using MediatR;
using Microsoft.EntityFrameworkCore;
using ReptileTracker.Application.Exceptions;
using ReptileTracker.Domain.Abstractions;

namespace ReptileTracker.Infrastructure;

public sealed class ApplicationDbContext : DbContext, IUnitOfWork
{
    private readonly IPublisher _publisher;
    public ApplicationDbContext(DbContextOptions options, IPublisher publisher) : base(options)
    {
        _publisher = publisher;
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(ApplicationDbContext).Assembly);
        base.OnModelCreating(modelBuilder);
    }   

    public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        try
        {
            var result = await base.SaveChangesAsync(cancellationToken);
            await PublishDomainEventsAsync(cancellationToken);
            return result;
        }
        catch (DbUpdateConcurrencyException ex)
        {
            throw new ConcurrencyException("A concurrency error occurred while saving the data", ex);
        }
    }

    private async Task PublishDomainEventsAsync(CancellationToken cancellationToken)
    {
        var domainEvents = ChangeTracker
            .Entries<Entity>()
            .Select(e => e.Entity)
            .SelectMany(x =>
            {
                var domainEvents = x.GetDomainEvents();
                x.ClearDomainEvents();
                return domainEvents;
            })
            .ToList();

        foreach (var domainEvent in domainEvents)
        {
            await _publisher.Publish(domainEvent);
        }
    }
}
```
Theres a lot of stuff going on here but lets try to break it down.

The first thing thats happening is that we have a construtor where we inject Mediatrs IPublisher (to publish in a CQRS pattern). 

The strangest thing in here might be: 

```
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(ApplicationDbContext).Assembly);
        base.OnModelCreating(modelBuilder);
    }   
```
 
 This is a method for scanning the assembly:
 ```
  modelBuilder.ApplyConfigurationsFromAssembly(typeof(ApplicationDbContext).Assembly);
 ```
and applying configurations on all entities so you dont have to call them manualy. 

These configurations can look somehting like this: 
```
    internal sealed class SheddingConfiguration : IEntityTypeConfiguration<Shedding>
    {
        public void Configure(EntityTypeBuilder<Shedding> builder)
        {
            builder.ToTable("shedding");
            builder.HasKey(x => x.Id);
            builder.Property(x => x.Date).IsRequired();
            builder.Property(x => x.Notes)
                .HasMaxLength(2000)
                .HasConversion(x => x.Value, Value => new Notes(Value));
            builder.HasOne<Reptile>()
                .WithMany()
                .HasForeignKey(x => x.ReptileId);
        }
    }
```
Seeing it now I'm a bit split if I really like it. I'm not shure if I think it looks clean or to abstract. Well well, it will stay for now... 

Finally we have the save changes method that our command handlers call when receiving a command 

```
public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        try
        {
            var result = await base.SaveChangesAsync(cancellationToken);
            await PublishDomainEventsAsync(cancellationToken);
            return result;
        }
        catch (DbUpdateConcurrencyException ex)
        {
            throw new ConcurrencyException("A concurrency error occurred while saving the data", ex);
        }
    }

    private async Task PublishDomainEventsAsync(CancellationToken cancellationToken)
    {
        var domainEvents = ChangeTracker
            .Entries<Entity>()
            .Select(e => e.Entity)
            .SelectMany(x =>
            {
                var domainEvents = x.GetDomainEvents();
                x.ClearDomainEvents();
                return domainEvents;
            })
            .ToList();

        foreach (var domainEvent in domainEvents)
        {
            await _publisher.Publish(domainEvent);
        }
    }
``` 
This is basicaly functionality for publishing a command and saving it to the database.

Thats pretty much how we work with databases in the infrastructure layer. As you can see it's a pretty lean setup that should be easy to migrate or scale.  

## Dependency Injection
We also handle our dependency injection need in this layer (so that we can call these classes from our Program.cs file). For this need  I've created a DependencyInjection.cs file that looks like this:

```
namespace ReptileTracker.Infrastructure;

public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(this IServiceCollection services, IConfiguration configuration)
    {
        services.AddTransient<IDateTimeProvider, DateTimeProvider>();

        AddPersistance(services, configuration);

        return services;
    }

    private static void AddPersistance(IServiceCollection services, IConfiguration configuration)
    {
        var connectionString = configuration.GetConnectionString("Database") ??
                               throw new ArgumentNullException(nameof(configuration));

        services.AddDbContext<ApplicationDbContext>(options =>
        {
            options.UseNpgsql(connectionString).UseSnakeCaseNamingConvention(); // Change naming conventions
            // to snake_case so that it matches the entities
        });

        services.AddScoped<IUserRepository, UserRepository>();
        services.AddScoped<IFeedingRepository, FeedingRepository>();
        services.AddScoped<IMeasurementRepository, MeasurementRepository>();
        services.AddScoped<IReptileRepository, ReptileRepository>();
        services.AddScoped<ISheddingRepository, SheddingRepository>();
        services.AddScoped<IUnitOfWork>(sp => sp.GetRequiredService<ApplicationDbContext>());
        services.AddSingleton<ISqlConnectionFactory>(_ =>
            new SqlConnectionFactory(connectionString));
        SqlMapper.AddTypeHandler(new DateOnlyTypeHandler());
    }
}
```
The real file hase some authentication and authorization stuff too but I removed that for this post since I left out anything about authentication and authorizaion. 

Here we set up our dependency injections and also our connections strings and db context. This is typicaly thing you would find in a Program.cs file but here we let our infrastructure layer handle it. 

## What about controllers, UI and alla other stuff? 
This has been a really light-weight rundown on the migrations process towards a clean architecthure. I showed you how to split up your application into domain, application and infrastructure layers. What I havn't showed is where to put controllers, a UI, other frameworks, docker setup and so on. But all these things are essentialy outside of our core appllication there for I skip that part for now. 

## Summary 
It's been a super fun project, migrating my service and I'm really glad I did. It left my with a much leaner and more loosely coupled application. Is clean architechture right for you? Thats for you to decide but know you seen my take on it, hope you find something fun to do with it too :) 
