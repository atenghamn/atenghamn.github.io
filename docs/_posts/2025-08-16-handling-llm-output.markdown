---
layout: post
title: "Handling Guids and nesting from LLM output"
date: 2025-08-16 16:00:00 +0100
categories: C#, LLM
---

I've recently run into a bit off weird issues when trying to save my structured LLM outputs using C#. The issue comes when interacting with a LLM using Microsoft.Extensions.AI 

```csharp
  var response = await chatClient.GetResponseAsync<MealPlan>(chatHistory);
```

In my particular use case a MealPlan is a model that holds recipes and are tied to a user. The recipe in it's turn has a list of ingredients. 
Some off the issues i had was that the LLM (gpt-5-mini in my case) had a hard time with the nesting so I ended up with a graph tree that Entity Framework couldn't save.

First thing I realized that I screwed up was that I allowed the LLM to create GUIDs which resulted in me getting faulty GUIDs for some properties. 
So I hade to let Entity Framework be in charge of that and use decorations to make shure it was database generated and ignored when parsing on all my Id properties that were affected.


```csharp
    [JsonIgnore]
    [Required]
    [Key]
    [DatabaseGenerated(DatabaseGeneratedOption.Identity)]
    public  Guid Id { get; init; }
```

The second issue I had was with the relations where I would sometimes get meal plans without recipes and other times I would get loops of mealplans with recipes that had the mealplan inside them that had the recipe inside and so on resulting in a huge json payload (glad I hade the [NotMapped] decorator on affected entities in Entity Framework). And as first step I tried to be more precise in the prompt rules 

```csharp
  new(ChatRole.System, """
                                  You are a friendly assistant that plans meals for a family. You give them:

                                                                     1. The recipes that they can cook
                                                                     2. The ingredients that they can use

                                                                     Your goal is to always bring a mix of easy to cook meals, that are affordable and also nutritious. 

                                                                     Output a single JSON object that matches the MealPlan schema expected by the app.

                                                                     Rules:
                                                                     - Do NOT include any properties named "id" or ending with "Id" (they will be generated/assigned server-side).
                                                                     - Do NOT include navigation objects (e.g., do not embed Recipe inside Ingredient).
                                                                     - Make sure that recipes and ingredients are created and unique.
                                  
                                 """),
```
This made the output a lot better and I got it working pretty sweet. 
But I would still say that fixing issues like his can be pretty cumbersome since the LLM outputs are non determinitic by nature which makes it a lot harder
to predict the outcome. With that said beeing able to cast the output is a huge help. 
