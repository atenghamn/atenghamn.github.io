---
layout: post
title: "The worst way to get a reminder"
date: 2025-07-23 11:00:00 +0200
categories: C#, azure
---

Ah vaccation, the perfect time to build something completely useless and impractical! 

So the situation is that I recently bought a some new plants for indoor use and planted some seeds in my garden. All of this is great but whats the point of buying something if I can't build something to go along with it ;) 

I've previously built a small soil moisture sensor that I use to monitor the soil moisture for indoor plants. It works but now I needed something that I could leave outside and survive the weather. I could of course have but a ESP-32 in a waterproof box but I already know how to do that so whats the fun in that?

The most obvious solution and probably the best one would be to set an alarm on my phone or use a calendar app to remind me to water the plants. But again, where is the fun in that?

Instead I decided to use Azure Functions and a timer trigger to send me an email every day at 8:00 AM reminding me to water the plants. This is of course a very bad idea since I will get an email every day even if I don't need to water the plants. But hey, I had never used Azure Functions before so why not?


I created a new Azure Function project and added a timer trigger function that runs every day at 8:00 AM. The code looks something like this:

```csharp
public class TimeFunc(ILoggerFactory loggerFactory)
{
    private readonly ILogger _logger = loggerFactory.CreateLogger<WateringTimeFunc>();

    [Function("WateringTimeFunc")]
    public void Run([TimerTrigger("0 0 20 * * *")] TimerInfo myTimer)
    {
        _logger.LogInformation("C# Timer trigger function executed at: {executionTime}", DateTime.Now);
       
    }
} 
```

The next step was sending a reminder email (I now this is so stupid, there are a hundred ways to do this better...), I previously used SendGrid to send emails so I decided to go with that. BUT.... Appearantly SendGrid didn't have a free option (apart from a 60 day trial) so that went to shit. After some searching I found out that Azure has a service called Azure Communication Services that includes email capabilities and has a free tier. So I decided to give that a try.

I created a new Azure Communication Services resource in the Azure portal and obtained the connection string. Then I updated my Azure Function to use the Azure Communication Services SDK to send the email. The updated code looks like this:
```csharp
public class TimeFunc(ILoggerFactory loggerFactory)
{
    private readonly ILogger _logger = loggerFactory.CreateLogger<WateringTimeFunc>();

    [Function("TimeFunc")]
    public void Run([TimerTrigger("0 0 20 * * *")] TimerInfo myTimer)
    {
        _logger.LogInformation("C# Timer trigger function executed at: {executionTime}", DateTime.Now);
        var connectionString = Environment.GetEnvironmentVariable("EmailPrimary");
        if (string.IsNullOrEmpty(connectionString))
        {
            throw new InvalidOperationException("Cant access connection string.");
        }

        var senderAddress = Environment.GetEnvironmentVariable("SenderEmailAddress");
        var recipientAddress = Environment.GetEnvironmentVariable("RecipientEmailAddress");
        var emailClient = new EmailClient(connectionString);
        try
        {
            var emailSendOperation = await emailClient.SendAsync(
                wait: WaitUntil.Completed,
                senderAddress: senderAddress,
                recipientAddress: recipientAddress,
                subject: "Dags att vattna blommorna",
                htmlContent: "<html><body>Vattna blommorna!</body></html>");
            _logger.LogInformation("Email sent successfully with operation ID: {operationId}", emailSendOperation.Id);
        }
        catch (RequestFailedException ex)
        {
            _logger.LogError($"Email send operation failed with error code: {ex.ErrorCode}, message: {ex.Message}");
        }
    }
} 
``` 

Everything was smooth sailing from here, I deployed the function to Azure and set up the necessary environment variables for the connection string and email addresses. Now, every day at 8:00 AM, I receive an email reminding me to water my plants. I will probably be annoyed after a week or so and just turn it off, but hey, I learned something new and had some fun along the way!