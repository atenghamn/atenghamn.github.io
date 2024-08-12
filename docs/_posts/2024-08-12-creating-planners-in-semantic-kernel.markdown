---
layout: post
title: "Creating planners in Semantic Kernel"
date: 2024-08-12 10:00:00 +0200
categories: ai, c#, semantickernel
---

After seeing a lot of cool demos of semantic kernel during the developer days in Berlin earlier this spring I've been quite curious about how it works and how you can use it as an developer. Having read up a bit on it and played around with it I thought it would be nice to highlight some of the things I find really interesting with semantic kernel.

The first thing we should do is to answer the obvious question, what is semantic kernel? Semantic kernel is a framework from Microsoft and is described in their [documentation](https://learn.microsoft.com/en-us/semantic-kernel/) as: 
>Semantic Kernel is a lightweight, open-source development kit that lets you easily build AI agents and integrate the latest AI models into your C#, Python, or Java codebase. It serves as an efficient middleware that enables rapid delivery of enterprise-grade solutions.


## Making functions available to the planner
This post wont be a getting started guide to semantic kernel. Instead I will focus on planners. A planner is a pretty cool feature that essentially combines functions to answer prompts with semantic kernel. A bit like an facade or orchestrator, this makes it possible to let the users interact with the model in ways that you as a developer haven't anticapted. Lets look at a simple example to get a grasp of it. Before we continue you need to know that semantic kernel can take two different kind of functions; semantic functions that are more or less just straight prompts with a configuration file and native functions that are functions created by a developer, in my example they are written in C# but they could just as well be written in Pyhton or Java. 

The idea of this example program is to let users ask questions about asteroids coming towards earth on a daily basis and also ask questions about inhabiting space in the future. Yeah, I know it's a really strange use case but I just picked something that would make it easy to demonstrate the diffenrent functionalities :)  If we start by looking at our semantic function we can see that it's really simple: 

```
You're an expert on life in space but are explaining it to a venture capitalist looking for future investments. 
Given the question below, explain in detailed terms.  
Question:
{{ $input }}
```
I dont think it need any extra explanation except that it takes an input parameter that's our actual question. 

The native function on the other hand looks like this: 

```csharp
public class AstroidChecker
{
    [KernelFunction]
    [Description("Calls the NASA api to see if there are any hazardous asteroids near earth")]
    public async Task<string> GetForeCast()
    {
        var config = new ConfigurationBuilder()
            .AddUserSecrets<Program>()
            .Build();
        try
        {
            var apiKey = config["nasaApiKey"];
            var today = DateTime.Now.ToString("yyyy-MM-dd");
            using HttpClient client = new();
            var response =
                await client.GetFromJsonAsync<ForeCastData>(
                    $"https://api.nasa.gov/neo/rest/v1/feed?start_date={today}end_date={today}&api_key={apiKey}");
            var result = GetHazardousObjectSpeedsAsString(response); 
            return result;

        }
        catch (HttpRequestException ex)
        {
            Console.WriteLine($"Error fetching weather data: {ex.Message}");
            return "something unexpected happened";
        }
    }

    private static string GetHazardousObjectSpeedsAsString(ForeCastData data)
    {
        var result = new System.Text.StringBuilder();
        result.AppendLine("things coming towards earth today is: ");

        foreach (var dateEntry in data.NearEarthObjects)
        {
            foreach (var neo in dateEntry.Value)
            {
                if (!neo.IsPotentiallyHazardousAsteroid) continue;
                foreach (var closeApproach in neo.CloseApproachData)
                {
                    result.AppendLine($"Name: {neo.Name} which are coming at a speed of {closeApproach.RelativeVelocity.KilometersPerHour}km/h");
                }
            }
        }

        return result.ToString();
    } 
}
```

We got a class named AsteroidChecker that uses the NASA [api](https://api.nasa.gov/) to check for asteroid forecasts coming towards earth. The result is just printed in the console. The thing that makes this work with semantic kernel is the function decorator:
```csharp
    [KernelFunction]
    [Description("Calls the NASA api to see if there are any hazardous asteroids near earth")]
    public async Task<string> GetForeCast()
```

This tells semantic kernel that it's a kernel function so it's available to the semantic kernel and supply it with a short descrption on what the function does. Our GetForeCast() function is now ready to be used by semantic kernel. The next step is to set everything up! 

## Wiring it up
Now that we got our functions we need to make them available to semantic kernel. 

```csharp
 var builder = Kernel.CreateBuilder();
        builder.AddOpenAIChatCompletion(
            "gpt-3.5-turbo",
            openAiKeys.apiKey,
            openAiKeys.orgId,
            "gpt3");

         builder.AddOpenAIChatCompletion(
             "gpt-4",
             openAiKeys.apiKey,
             openAiKeys.orgId,
             "gpt4");
```
We start by creating a builder were we add our models. In this case I went with gpt-3.5 and gpt-4. They will execute in the order we put them in so in this case gpt-3.5 will be called first and if the kernel decicdes that it's to complex for gpt-3.5 it will instead use gpt-4. Note that we will also need to set this up in our semantic functions config file. 

Next step is to load our plugins in the kernel.
```csharp
     var pluginsDirectory =
            Path.GetFullPath(config["pluginsPath"]);
        builder.Plugins.AddFromPromptDirectory(pluginsDirectory);
        builder.Plugins.AddFromType<AstroidChecker>();
        var kernel = builder.Build();
```
Here you can see that we load two different kind of plugins. First we load the semantic plugins by specifing the semantic plugins directory. After that  we add our native function by using the AddFromType<T>() function. Finally we build our kernel so it's ready to be used. 

## Creating a planner 
The last step is to create a planner that can create a plan on how to execute our prompts and this part is were the magic happens. The really cool part about this is that we planner can combine our functions to handle prompts. So if we have a prompt that needs multiple functions to handle the prompt the planner handles this for us. The first step is to create a HandelbarsPlanner and supply it with options. As of now I think that the HandelBarsPlanner is a C# thing only and currently in preview so if you're using java or python I think you're currently stuck with the StepWisePlanner. In C# you have to disable the pragma SKEXP0060 to make it compile since it's in preview.
```csharp
#pragma warning disable SKEXP0060
``` 

When we create our HanndlebarsPlanner we supply it with options were we specify temperature and TopP (to set the level of creative freedom the kernel gets). We also set the max tokens (wich can get quite high in complex plans) and if we allow loops. Finally we create our planner with the options we specified. 

```csharp
        var plannerOptions = new HandlebarsPlannerOptions
        {
            ExecutionSettings = new OpenAIPromptExecutionSettings
                       {
                           Temperature = 0.5,
                           TopP = 0.3,
                           MaxTokens = 4000
                       },
            AllowLoops = true
        };
        var planner = new HandlebarsPlanner(plannerOptions);
```
To demonstrate how it works I created a question that utilizes both our functions. In one sentence we ask the kernel if there are any dangerous asteroids coming towards earth and when we can inhabit space. It's a really strange prompt but the goal is just to demonstrate how the planner works. When we created our plan we invoke it and print the answer to the console.
```csharp
        const string question = "is there any dangerous asteroids coming towards earth and when can we inhabit space? "; 
        var plan = planner.CreatePlanAsync(kernel, question).Result;
        var result = await plan.InvokeAsync(kernel, []);
        Console.WriteLine(result);
```

It now prints out 
```
Astroid Forecast: things coming towards earth today is: 
Name: (2024 JV33) which are coming at a speed of 39877.467220643km/h
Name: (2017 JE3) which are coming at a speed of 35433.7137855868km/h
Name: (2019 OY3) which are coming at a speed of 114459.6776338368km/h
Name: (2015 DB1) which are coming at a speed of 78251.3029765613km/h
Name: (2009 DH39) which are coming at a speed of 95977.0987384237km/h
Name: (2013 EV108) which are coming at a speed of 85037.1901093908km/h
Name: (2015 PM) which are coming at a speed of 59580.805597502km/h
Name: 620063 (2009 DH39) which are coming at a speed of 95977.1031943676km/h


As for inhabiting space, it's a complex question that depends on many factors including technological advancements, resources, and international cooperation. It's not something that can be predicted accurately at this time.
```
Remember that the HandelbarsPlanner is still in preview so the results you get may vary. It's a good idea to test your prompts a few times and change the temeperature and TopP to try and find a sweet spot between hallucination and correctness. I think the really cool part about this is that it manage to create plans that combine both native and semantic functions into one answer. With this you can create functionality that is way beond what you imagined when you wrote the individual functions. If you want to test your native functions that just as straight forward as a standard C# function. You can write tests and debug it. But it's damn near impossible to test your semantic functions since they are just prompts to the model of your choice and the result can very based on your planner options and the models inner workings.

That's all for now, I hope you got a taste for how you can work with planners in your applications. To read more about planners check [this site](https://handlebarsjs.com/). If you want to learn more about semantic kernel I recommend [Building AI Aplications with Microsoft Semantic Kernel](https://www.packtpub.com/en-us/product/building-ai-applications-with-microsoft-semantic-kernel-9781835463703) by Lucas A. Meyer. Remeber that semantic kernel is far from feature complete and things tend to change quite frequently. 

Take care! 
