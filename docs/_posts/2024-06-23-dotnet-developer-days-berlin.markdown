---
layout: post
title:  ".NET Developer Days in Berlin"
date:   2024-06-23 16:30:00 +0200
categories: conference
---
I was fortunate enough to go to the [.Net Developer Days conference in Berlin](https://developerdays.eu/berlin/) between 17-18 of june. Since it was my first time going to any developer conference, I thought I’d share my impressions.

### Day one 
I arrived in Berlin early Monday morning and took a cab from Brandenburg Airport to the Estrel Hotel and Conference Center. It seemed huge, so I found out that it’s Germany’s largest hotel with 1,125 rooms, and the congress part of the building can hold up to 15,000 attendees.
![Estrel hotel](https://www.hotel-board.com/picture/estrel-berlin-hotel-1701983.jpg)

The first talk I attended was "Docker 101". Since I thought I lacked Docker skills, I saw this as a great opportunity to catch up. However, I found this talk to be a bit shallow and not very informative. This was my own fault since it was clearly advertised as an entry-level session, and I guess I know more about Docker than I thought :)

After the Docker talk, I attended Jiachen Jiang’s session from Microsoft on .NET Aspire and Azure Container Apps. It was a good talk, though it could have been a bit more hands-on. As it was, it served as a general introduction to the technology without delving into the nitty gritty of how it worked.

A talk I really enjoyed was given by Pieter Niejs on Semantic Kernel. Semantic Kernel is Microsoft’s SDK for building agents that can interact with AI models. This enables you to use LLMs from OpenAI or Hugging Face in your own codebase. The really cool thing about this is how easy it is to get started. My favorite feature is the ability to tell the model to use your own functions for certain scenarios if you annotate them with KernelFunction. See the example below from Microsoft’s documentation.

csharp

```
public class EmailPlugin
{
    [KernelFunction]
    [Description("Sends an email to a recipient.")]
    public async Task SendEmailAsync(
        Kernel kernel,
        [Description("Semicolon delimitated list of emails of the recipients")] string recipientEmails,
        string subject,
        string body
    )
    {
        // Add logic to send an email using the recipientEmails, subject, and body
        // For now, we'll just print out a success message to the console
        Console.WriteLine("Email sent!");
    }
}
```
I also attended a talk called "Modern C#: A Dive into the Community's Most Loved New Features" by Louëlla Creemers. She went over some of the new features and got me really excited to get involved with the Roslyn compiler project.

The absolute highlight of the day was the last talk: The Perils and Promise of Artificial Intelligence by Richard Campbell. Richard is one of the hosts of [.Net Rocks](https://www.dotnetrocks.com/) a podcast that I think you should check out if your'e a .Net developer. 

Richard Campbell told the history of AI development in a really entertaining way that made it easily accessible without losing depth. If you have the opportunity to hear him speak live, I highly recommend it as he’s a great speaker.

### Day two 
I started the second day with Digging for Gold: New in ASP.NET Core 8, a really pretty nice rundown of everything new in the latest release. An interesting thought here was that ASP.NET is feature complete while Blazor is still has a lot of room for innovation.

Another fun talk was Dino Espositos talk Human-readable Markup with ASP.NET Tag Helpers. It got me really hyped to do some more frontend work and actually check out Blazor. 

I started the second day with "Digging for Gold: New in ASP.NET Core 8," a nice rundown of everything new in the latest release. An interesting thought here was that ASP.NET is feature-complete, while Blazor still has a lot of room for innovation.

Another fun talk was Dino Esposito’s "Human-readable Markup with ASP.NET Tag Helpers." It got me really excited to do more frontend work and check out Blazor.

A final talk that I found really interesting was by Małgorzata Janeczek, titled "Screw Perfect, Aim for Good Enough." Her thesis was that we should stop striving for writing "perfect" code and instead focus on what matters. It may sound strange, but her point was that we should try to keep things as simple as possible to get the most value out of our time and reduce complexity. She definitely had some good points.

### Summary
This being my first developer conference, the obvious question is, "Would I like to go again?" The answer is a resounding YES! I got so much inspiration and energy from all the talks, and it was great talking to other developers and hearing what they were up to. I really enjoyed this conference and I really hope I get the oppertunity to go again next year! 

To check the comming .Net developer conferences click [here](https://developerdays.eu/)