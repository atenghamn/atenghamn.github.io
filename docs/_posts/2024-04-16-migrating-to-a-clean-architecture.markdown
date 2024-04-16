---
layout: post
title:  "Migrating to clean architecture"
date:   2024-04-16 22:14:00 +0200
categories: arichitecture
---

This spring the company that I work for offered me the oppertunity to enroll in a course of my choice (as long as it was relevant to my work). I thought about everything from an Azure exam-preperation course to something in machine learning. But after thinking about in what direction I want to move as a junior developer who graduated a year back I thought that arcitechure and design pattern is a field that I really want to get into. Beeing a fan of Milan Jovanovics [newsletter](https://www.milanjovanovic.tech/).  I decided to give his course in clean architecture a go. 

Having completed the foundations of the course and having created an example application I thought it was time to really test my knowledge. I didn't want to create just another dummy application without any purpose more than beeing a POC for the thing I'm learning (I have a collection of them allready...). Instead I wanted to build something that I could really test with usage over time and someting that I could keep building on as new ideas emerged. 

Luckily I had just the perfect project for that! My reptile tracker service that is a simple service for logging reptile [metrics](https://github.com/atenghamn/ReptileTracker) 
And yeah... I realize that this sound pretty strange If you're not an reptile owner but the core idea of the service is to be able to log what and when your reptiles eat, when they shedd and their measurements. This can then be presented in some UI layer or just straight in the database which is my current setup since I havn't really had the time to finish the UI.

So my plan moving forward is to document the migration of the service here. I will probably write about migrating each layer here and then sum it up when everything is done. 

If you never heard about clean architecture I guess here is a good place to [start](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) 

What I hope to achive with this migration is:
- Reducing dependencies to make future changes easier
- Isolating the business logic making it more testable and maintainable

So yeah... here we go! 