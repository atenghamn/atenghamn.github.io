---
layout: post
title: "Building multi agent AI systems with Semantic Kernel"
date: 2024-09-15 11:50:00 +0200
categories: ai, c#, semantikernel, llm
---

I have been playing around with [Semantic Kernel](https://learn.microsoft.com/en-us/semantic-kernel/overview/), Microsofts sdk for building AI agents and more recently been testing out what you can do with mulitple agents. Before we start it may be suitable to have some disclaimers. If you have never heard of Semantic Kernel, a good place to start is with [Microsofts quick start](https://learn.microsoft.com/en-us/semantic-kernel/get-started/quick-start-guide?pivots=programming-language-csharp). This post will assume that you have some experience with semantic kernel and that you know your way around C#. To code along with this example you also need an openAI api key (and some credits). 

The second disclaimer is that some of the nuget packages I use in this post are in alpha or release-candidates so when you are reading this there may be newer releases with added functionality or breaking changes. With that said, lets jump in to how to create a multi agent system with Semantic Kernel.

If we are going to work with AI agents the first thing to do is to define what an AI agnent is in the Semantic Kernel context. Microsoft describes it pretty straigh forward as "Agents are software based entities that leverage AI models to do works for you. They are built to perform a wide range of tasks and called different names based on the jobs they do." in their [documentation](https://learn.microsoft.com/en-us/semantic-kernel/concepts/agents?pivots=programming-language-csharp). They go on to list different examples like chatbots and copilots. But what does the agent consist of? An agent in Semantic Kernel is made up of three building blocks: plugins, planners and persona. A plugin is essentially a function and planners are execution plans for these functions. If you haven't read my previous post about [creating planners in Semantic Kernel](https://atenghamn.github.io/ai,/c%23,/semantickernel/2024/08/12/creating-planners-in-semantic-kernel.html) it may be a good idea to do that if you haven't worked with planners, prompts or kernel functions before. Given an agent a persona sounds kind of creepy but it's more in the lines of context. The persona is the part that communicate with the end user. Think of the chatGPT interface where you can have a conversational style of interacting with the model. This is the persona. Here we give the agent a context of what role it has and what we wish to achive. 

Now that we now the lingo it's time for some coding! I've made a simple demo project where the main idea is to build a photography helper that can help us create photo trips depending on what we want to shoot. Maybe not the most original idea but I needed something for this demo so it will do for this purpose.


### Setting up Semantic Kernel
First thing first let's download the following nuget packages 

```bash
dotnet add package Microsoft.SemanticKernel.Connectors.OpenAI --version 1.19.0
dotnet add package Microsoft.SemanticKernel.Agents.Core --version 1.19.0-alpha
dotnet add package Microsoft.SemanticKernel.Agents.Abstractions --version 1.19.0-alpha
dotnet add package Microsoft.SemanticKernel.Agents.OpenAI --version 1.19.0-alpha

```
As you can see the packages for working with agents are still in alpha so please keep in mind that this is pretty unstable and likely to change, it's not something yoy want to take to production. For testing these packages at home, remeber to disable the warnings from Rozlyn with: 

```
#pragma warning disable SKEXP0110, SKEXP0001 

```

 Once we downloaded all the nuget packages and disabled the warning it's time to set up  Semantic Kernel. 
```csharp
var kernel = Kernel.CreateBuilder()
    .AddOpenAIChatCompletion(
        modelId: "gpt-4o",
        apiKey: "your-api-key-here"
    )
    .Build();
```

Try out the model you like, I went with 4o but maybe you can go for a cheaper model. 

#### Creating agents

Now it's time for us to create some agents. I decided to go with three agent (you can go with as many or as few you like just remember that more agents often leads to a higher cost since every agent integrates with the LLM). In this example we are going to create three agents:

- The location scout 
 will scout photography locations for us based on our requrements and give us recommendatinos on where to shoot and also create a plan of how to get there with public transportation. 

- The idea suggestor, the location scout will pass on the locations to the idea suggestor who will create ideas for photo shoots based on the locations given by the location scout.

- The manager, the idea suggestor will pass on the ideas and location to the manager who will review them and ensure our requirements are met. If they are met hte manager will approve this by just responding "approve". 

To create our agents we create a new agent and give them instructions and a kernel. In this example we use the same kernel for all of our agents but you can use diferent kernels for different agents if you want to. It's perfectly fine to use OpenAi for some and maybe AzureOpenAi or Llama for others. 

```csharp

// Create instructions for each of the agents
var locationScout = """
You are a photography location scout which will take the requirement and create a plan for suggesting photography locations in Skåne, Sweden.
Location scout understands the requirements and form the location documents with a plan on were to photograph and how to get there by public transportation from Lund
""";

var ideaSuggester = "You  are a idea suggester and your goal is to suggest ideas for photography shoots by taking into consideration all the requirements given by the location scout.";

var manager = """
You are the manager that will review photographic ideas given by the idea suggester and make sure all client requirements are completed.
Once all client requirements are completed you can approve the request by just responding "approve".
""";

// Create a new agent
ChatCompletionAgent managerAgent = 
new()
{
    Instructions = manager, // Give it the instruction from above
    Name = "ManagerAgent", // Give it a name, this can be whatever
    Kernel = kernel // Point to a kernel
};

ChatCompletionAgent locationScoutAgent =
    new()
    {
        Instructions = locationScout,
        Name = "LocationScout",
        Kernel = kernel
    };

ChatCompletionAgent ideaSuggesterAgent =
    new()
    {
        Instructions = ideaSuggester,
        Name = "IdeaSuggester",
        Kernel = kernel
    };

```

### Termination strategy 
If you look in the instructions for the manager the line below may seem a bit odd
``` 
"Once all client requirements are completed you can approve the request by just responding "approve".
```
This is the termination strategy it's purpose is to define a definiton of done for our agents. To create a termination strategy we create a new class that inherits from the [TerminationStrategy](https://learn.microsoft.com/en-us/dotnet/api/microsoft.semantickernel.agents.chat.terminationstrategy?view=semantic-kernel-dotnet) class.

```csharp

using System.Diagnostics.CodeAnalysis;
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.Agents;
using Microsoft.SemanticKernel.Agents.Chat;

namespace PhotographyHelpers;

[Experimental("SKEXP0110")] // Since it's in alpha this is an experimental feature so keep it out of production environments
public class ApprovalTerminationStrategy : TerminationStrategy
{
    protected override Task<bool> ShouldAgentTerminateAsync(
        Agent agent,
        IReadOnlyList<ChatMessageContent> history,
        CancellationToken ct
    ) => Task.FromResult(history[^1]
                             .Content?.Contains("approve", StringComparison.OrdinalIgnoreCase)
                         ?? false);
}

```

This nothing fancy, the method is inherited from the TerminiationStrategy class and it evalutes the input messages from our chat to see if it meets our definition of done (our completion criteria). In this perticual case it takes the last message (first from the end) and check for the word "approve" since that is what we are looking for. If it's found then true is returned. 

### Creating a group chat

Now it's time to link everything together and we do that by creating a [AgentGroupChat](https://learn.microsoft.com/en-us/dotnet/api/microsoft.semantickernel.agents.agentgroupchat?view=semantic-kernel-dotnet). The name is pretty selfexplanatory and what you do is you give your agents as input and sets the termination strategy.

```csharp
AgentGroupChat chat = new(managerAgent, locationScoutAgent, ideaSuggesterAgent)
{
    ExecutionSettings = new()
    {
        TerminationStrategy = new ApprovalTerminationStrategy()
        {
            Agents = [ managerAgent ],
            MaximumIterations = 6
        }
    }
};

```
Here we give it the termination strategy we created earlier and explicitly say that it's our managerAgent that can determine if the approval strategy has been fulfilled. I've set the maximum iterations to 6, you can set it to whatever you want and maybe 6 is a bit high in this case since it rarely needs all six attempts. The group chat holds our agents chat conversation and our conversation with the agents. All that's left now is to tie it all together and create a really simple user interface so that we can interact with our agents. 

### Creating the terminal interface 

Now we are going to create a simple command line interface so that we can interact with our agents and print the response from our agents. 

```csharp
Console.WriteLine("Specify what and when you want to photography");
while (true)
{
    var input = Console.ReadLine();
    chat.AddChatMessage(new ChatMessageContent(AuthorRole.User, input));
    
    await foreach (var content in chat.InvokeAsync())
    {
        Console.WriteLine($"# {content.Role} - {content.AuthorName ?? "*"}: {content.Content}");
    }
}
```

Here we start by presenting the user with a prompt to give the agents something to work with. Once we have a proimpt we add it as a chat message to the agents with the author role set to user. Our agents set of to work and we print each response by calling chat.InvokeAsync() and prinnting the response. Here you can alaborate freely to get the content that's relevant for you. 

Lets try it out by asking it to create a plan for a street photography shoot in Helsingborg for tomorrow! 
This is what we get: 

```powershell
Specify what and when you want to photography
I want to do some street photography tomorrow morning in Helsingborg

Assistant - ManagerAgent: That's a great idea! Could you provide any specific requirements or objectives you have in mind for your street photography session in Helsingborg? For instance, are there any particular subjects or themes you want to focus on, special locations you want to visit, or specific times you're aiming to capture? This will help in ensuring all your requirements are met.

Assistant - LocationScout: Excellent choice! Helsingborg offers diverse scenes for street photography, ranging from bustling city streets to tranquil harbor views. Here's a detailed plan for your street photography session tomorrow morning, including suggestions on where to photograph and how to get there by public transportation from Lund.

### Suggested Locations in Helsingborg

1. **Kärnan Tower and Surrounding Area**
    - **Description:** Begin your day here for a panoramic view of the city and peaceful early-morning scenes in the nearby park. The medieval tower and its surrounding gardens offer excellent vantage points and interesting human activity.
    - **Photography Subjects:** Historic architecture, early-morning light, people walking their dogs, joggers, and tourists.
2. **Stortorget (The Great Square)**
    - **Description:** This is the heart of the city with vibrant street life, cafes, and historic buildings. It's perfect for capturing local life and urban architecture.
    - **Photography Subjects:** Street performers, locals, and visitors interacting, architectural details.
3. **Kullagatan**
    - **Description:** One of Sweden's oldest pedestrian streets, filled with shops, cafes, and colorful facades. It's a fantastic spot for candid street photography.
    - **Photography Subjects:** Shoppers, street vendors, interesting shop windows, and pedestrians.
4. **Helsingborg Central Station Area**
    - **Description:** A modern hub with a mix of contemporary and classic design. It's busy in the mornings with commuters.
    - **Photography Subjects:** Commuters, travelers, reflections, and architectural contrasts.
5. **North Harbor (Norra Hamnen)**
    - **Description:** A modern waterfront area with boats, modern residential buildings, and walking paths. It's usually quieter in the morning with potential for serene and reflective compositions.
    - **Photography Subjects:** Boats, waterfront activities, reflections, and modern architecture.

### How to Get There by Public Transportation from Lund

1. **Lund to Helsingborg**
    - Take the Öresundståg train from **Lund Central Station** to **Helsingborg Central Station**. The journey takes about 40-45 minutes with frequent departures, especially in the mornings.
2. **Getting Around Helsingborg**
    - Most of the highlighted locations are within walking distance of each other and the Helsingborg Central Station, making it convenient to explore on foot.

### Detailed Itinerary

**06:00 AM - Departure from Lund**

- Arrive at Lund Central Station by 6:00 AM and catch the Öresundståg train to Helsingborg.

**07:00 AM - Arrive in Helsingborg**

- Arrive in Helsingborg Central Station by around 7:00 AM.

**07:15 AM - Kärnan Tower**

- Walk to Kärnan Tower, about a 10-minute walk from the Central Station.
- Spend around 45 minutes capturing the sunrise light and the serene morning atmosphere.

**08:15 AM - Stortorget**

- Head towards Stortorget, a 5-minute walk from Kärnan.
- Spend about an hour in Stortorget photographing the early bird locals, street performers, and architectural elements.

**09:15 AM - Kullagatan**

- Move to Kullagatan for some busy pedestrian street scenes.
- Spend around an hour photographing here.

**10:15 AM - Helsingborg Central Station Area**

- Circle back to the Central Station area to focus on commuter activities.
- Spend around 30-45 minutes here.

**11:00 AM - North Harbor (Norra Hamnen)**

- Walk to North Harbor (around 10-15 minutes from the Central Station).
- End your morning session capturing the tranquil scenes by the water and the unique modern architecture.

**12:00 PM - Conclusion**

- Option to grab lunch and review your photos, or head back to Lund by catching the Öresundståg train from Helsingborg Central Station.

### Tips for a Successful Street Photography Session

- **Early Start:** Early mornings provide soft light and fewer crowds.
- **Blend In:** Dress casually to avoid drawing too much attention.
- **Mind the Weather:** Check the forecast and dress accordingly.
- **Gear:** Bring a versatile lens (e.g., 35mm or 50mm) for varied shots.
- **Stay Safe:** Keep an eye on your surroundings and your gear.

Enjoy your street photography adventure in Helsingborg!

Assistant - IdeaSuggester: Sounds fantastic! Helsingborg has a lot to offer for a morning street photography session. Here's a well-rounded plan to capture the essence of the city:

### Suggested Locations in Helsingborg

1. **Kärnan Tower and Surrounding Area**
    - **Description:** Start with the iconic Kärnan Tower, a medieval tower offering panoramic views. The surrounding park is perfect for catching the early morning light.
    - **Photography Subjects:** Historic architecture, morning joggers, and the serene ambiance of the park.
2. **Stortorget (The Great Square)**
    - **Description:** A lively square in the heart of Helsingborg. It's a great place to capture the hustle and bustle of city life.
    - **Photography Subjects:** Early morning vendors, commuters, and cafes setting up for the day.
3. **Kullagatan**
    - **Description:** One of Sweden's oldest pedestrian streets. It's lined with shops, cafes, and colorful facades.
    - **Photography Subjects:** Shoppers, street vendors, interesting shop windows, and pedestrians.
4. **Sofiero Castle Gardens**
    - **Description:** A bit of a walk or a short bus ride, but the gardens are splendid in the morning light.
    - **Photography Subjects:** Garden landscapes, flowers in bloom, and the castle's architectural details.
5. **Helsingborg Central Station Area**
    - **Description:** Modern design mixed with classic elements, a busy hub especially during morning rush hours.
    - **Photography Subjects:** Travelers, architectural details, and the movement of trains.

### Detailed Itinerary

**06:00 AM - Departure from Lund**

- Arrive at Lund Central Station by 6:00 AM and catch an early Öresundståg train to Helsingborg.
- Travel Time: Approximately 40 minutes.

**07:00 AM - Arrive in Helsingborg**

- Arrive at Helsingborg Central Station.

**07:15 AM - Kärnan Tower**

- Walk to Kärnan Tower, roughly a 10-minute walk from the station.
- Spend around 45 minutes capturing the morning light and the tower's surroundings.

**08:00 AM - Stortorget**

- Head to Stortorget, about a 5-minute walk from Kärnan.
- Spend about an hour capturing the early city life and architectural splendor.

**09:00 AM - Kullagatan**

- Move to Kullagatan for some vibrant street scenes.
- Spend about an hour photographing here.

**10:00 AM - Helsingborg Central Station**

- Circle back to capture the bustling station area.
- Spend around 30-45 minutes.

**10:45 AM - North Harbor (Norra Hamnen)**

- Walk to North Harbor, approximately a 10-15 minute walk from the Central Station.
- Capture waterfront activities and modern architecture.

**11:30 AM - Sofiero Castle Gardens (Optional)**

- If you're up for it, take a short bus ride or drive to Sofiero Castle Gardens.
- Spend about an hour capturing the peaceful garden scenes.

**12:30 PM - Conclusion**

- Head back towards the central area for lunch and review your shots.
- Return to Lund by catching the Öresundståg train from Helsingborg Central Station.

### Tips for Success

- **Check the Weather:** Dress accordingly and be prepared for potential changes in conditions.
- **Blend In:** Dress in a way that helps you blend into the environment, avoiding attracting unwanted attention.
- **Keep Gear Light:** A versatile zoom lens and a lightweight tripod can be very handy.
- **Stay Safe:** Be aware of your surroundings, especially in busy areas.

Enjoy capturing the charm and vibrancy of Helsingborg!

Assistant - ManagerAgent: Your plan covers a great mix of locations that showcase Helsingborg's diverse scenes and atmospheres. All client requirements have been considered: starting early, capturing a blend of historic and modern settings, focusing on street life and architectural details, and incorporating a picturesque garden.

approve
```
As you can see the location scout start by giving us locations in Helsingborg and passing them on to the idea suggestor who gives us plans for what to photograph on the given locations. Finaly the manager looks at the plan and approves it. We have some tweaks we could do like telling the ideasuggestor not to repeat everything the location scout has said but instead just add to it. Our manager could instead respond with the whole plan before approving or we could introduce a fourth agent between the idea suggestor and manager that handels the packaging of the end user message so that everything we see would be the final plan. But that would require finetuning our loop with the Console.WriteLine so that it only printed the message from the agent that presents the finished plan.

### Summary 
So this was a quick introduction introduction on bulding multi agents systems in Semantic Kernel. I hope it can be a stepping stone and inspiration for you to do your own experiments with Semantic Kernel. Until next time, take care! 