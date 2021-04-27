---
title:  "Automagically manage your Azure Service Bus topics with Farmer"
date:   2021-04-27 21:46:00 +0100
categories: fsharp 
tags:
    - infrastructure-as-code
classes: wide
toc: true
header: 
    overlay_image: /assets/images/sansebastian1.jpg 
    overlay_filter: rgba(0, 0, 0, 0.4)
published: true
---

In this blog post I will show how we reduced our 1000+ lines of Azure Service Bus arm-template configuration to just a few lines of F#. And then some...

## Background

Although I do value what arm-templates bring to the table with repeatable infrastructure deployments, it does not really deliver on the promise of infrastructure as code. Mostly because the arm-template json is not a real programming language but rather a pretty horrible language to do any actual coding in. 

In our current Azure Service Bus setup we are using quite a lot of topics and subscriptions. We have adopted a one-to-one relationship between our messages and topics. A topic can then have one or more subscribers (in our case an Azure Function). 
For example a `BlogPostCreated` message is sent to a topic with the same name and will be consumed by the subscribers `UpdateBlogPostSearchIndex` and `NotifyBlogSubscribers`.

The json for creating a service bus namespace with a single topic and two subscriptions in an arm-template is quite verbose, and comes to about 50 lines of json:

```json
{
    "apiVersion": "2017-04-01",
    "dependsOn": [],
    "location": "westeurope",
    "name": "my-service-bus",
    "sku": {
        "name": "Standard",
        "tier": "Standard"
    },
    "tags": {},
    "type": "Microsoft.ServiceBus/namespaces"
},
{
    "apiVersion": "2017-04-01",
    "dependsOn": [
        "[resourceId('Microsoft.ServiceBus/namespaces', 'my-service-bus')]"
    ],
    "name": "my-service-bus/BlogPostCreated",
    "properties": {
        "defaultMessageTimeToLive": "P14D",
        "duplicateDetectionHistoryTimeWindow": "PT10M",
        "enablePartitioning": true,
        "requiresDuplicateDetection": true
    },
    "type": "Microsoft.ServiceBus/namespaces/topics"
},
{
    "apiVersion": "2017-04-01",
    "dependsOn": [
        "[resourceId('Microsoft.ServiceBus/namespaces/topics', 'my-service-bus', 'BlogPostCreated')]"
    ],
    "name": "my-service-bus/BlogPostCreated/UpdateBlogPostSearchIndex",
    "properties": {
        "deadLetteringOnMessageExpiration": true,
        "duplicateDetectionHistoryTimeWindow": "PT10M",
        "lockDuration": "PT5M",
        "maxDeliveryCount": 10,
        "requiresDuplicateDetection": true
    },
    "resources": [],
    "type": "Microsoft.ServiceBus/namespaces/topics/subscriptions"
},
{
    "apiVersion": "2017-04-01",
    "dependsOn": [
        "[resourceId('Microsoft.ServiceBus/namespaces/topics', 'my-service-bus', 'BlogPostCreated')]"
    ],
    "name": "my-service-bus/BlogPostCreated/NotifyBlogSubscribers",
    "properties": {
        "deadLetteringOnMessageExpiration": true,
        "duplicateDetectionHistoryTimeWindow": "PT10M",
        "lockDuration": "PT5M",
        "maxDeliveryCount": 10,
        "requiresDuplicateDetection": true
    },
    "resources": [],
    "type": "Microsoft.ServiceBus/namespaces/topics/subscriptions"
}
```

## Enter Farmer

[Farmer](https://compositionalit.github.io/farmer/), according to its own description, is "a .NET domain-specific-language (DSL) for rapidly generating Azure Resource Manager (ARM) templates."

With Farmer's DSL we can re-write the json above to

```fsharp
serviceBus {
    name "my-service-bus"
    sku ServiceBus.Sku.Standard
    add_topics [
        topic {
            name topicName
            message_ttl (14<Days>)
            duplicate_detection_minutes 10
            enable_partition
            add_subscriptions [
                subscription {
                    name "UpdateBlogPostSearchIndex"
                    lock_duration_minutes 5
                    enable_dead_letter_on_message_expiration
                    max_delivery_count 10
                    duplicate_detection_minutes 10
                }
                subscription {
                    name "NotifyBlogSubscribers"
                    lock_duration_minutes 5
                    enable_dead_letter_on_message_expiration
                    max_delivery_count 10
                    duplicate_detection_minutes 10
                }
            ]
        }
    ]
}
```

Since we know there's going to be many more topics and subscriptions, let's extract that part to an array where we can add all our topics and subscriptions:

```fsharp
module Topics =
    let all = 
        [|
            {| Topic = "BlogPostCreated"
               Subscriptions = [| "UpdateBlogPostSearchIndex"; "NotifyBlogSubscribers" |] |}
            // ... and many more
        |]

serviceBus {
    name "my-service-bus"
    sku ServiceBus.Sku.Standard
    add_topics 
        [ for ts in Topics.all do
            topic {
                name ts.Topic
                message_ttl (14<Days>)
                duplicate_detection_minutes 10
                enable_partition
                add_subscriptions 
                    [ for subscriptionName in ts.Subscriptions do
                        subscription {
                            name subscriptionName
                            lock_duration_minutes 5
                            enable_dead_letter_on_message_expiration
                            max_delivery_count 10
                            duplicate_detection_minutes 10
                        }]
            }]
}
```

With the move from json to Farmer above we managed to reduce the amount of code for our full service bus configuration (~30 topics) with ~1000 lines. 

## Another problem of using json arm-templates

Both the infrastructure code (the json template) and our Azure Functions application need to know about the topic and subscription names. Additionally, the `ServiceBusTrigger` attribute we use for the azure function input binding, requires the topic and subscription name to be known at compile time:

```csharp
[Function("UpdateBlogPostSearchIndex")]
public async Task RunUpdateBlogPostSearchIndex(
    [ServiceBusTrigger("BlogPostCreated", "UpdateBlogPostSearchIndex", Connection = "<connection-string>")]Message message, FunctionContext executionContext)
    {
        // code, code, code...
    }
```

Previously we didn't really solve this in a nice way, and ended up having the topic and subscription names duplicated in the arm-template parameters and the application code in the Azure Function App. If you needed to change the name of a topic or subscription you had to remember to do so in both places.

## The benefits of using a 'real' programming language

When you realise that Farmer is just .NET code and that you can have your infrastructure code interop with your application code this opens up a whole new can of worms. So when our Farmer code needs to add all the topics and subscriptions it just need to find all the usages of the `ServiceBusTrigger` attribute, and that's not too hard to do with a bit of reflection:

```fsharp
open System.Reflection
open Microsoft.Azure.Functions.Worker

module Topics =
    let all =
        typeof<MyFunctionApp>.Assembly.GetTypes()
        |> Array.choose 
            (fun typ -> 
                typ.GetCustomAttribute<ServiceBusTriggerAttribute>() 
                |> Option.ofObj)
        |> Array.groupBy (fun attribute -> attribute.TopicName)
        |> Array.map 
            (fun (topic, grouping) -> 
                let subscriptions = grouping |> Array.map (fun attribute -> attribute.SubscriptionName)
                {| Topic = topic; Subscriptions = subscriptions |})
```

So now we have, not only reduced our arm-template configuration by 1000+ lines of code, we have also the added benefit of always having the infrastructure deployment code being in sync with the application code. If you make a change to a topic or subscription in the application code the Farmer deployment will find out about it automagically!
