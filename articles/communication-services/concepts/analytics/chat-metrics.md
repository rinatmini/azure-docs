---
title: Chat metrics definitions for Azure Communication Service
titleSuffix: An Azure Communication Services concept document
description: This document covers definitions of chat metrics available in the Azure portal.
author: mkhribech
manager: timmitchell
services: azure-communication-services
ms.author: mkhribech
ms.date: 06/23/2023
ms.topic: conceptual
ms.service: azure-communication-services
ms.subservice: data
---
# Chat metrics overview

Azure Communication Services currently provides metrics for all ACS primitives. [Azure Metrics Explorer](../../../azure-monitor\essentials\metrics-getting-started.md) can be used to plot your own charts, investigate abnormalities in your metric values, and understand your API traffic by using the metrics data that Chat requests emit.

## Where to find metrics

Primitives in Azure Communication Services emit metrics for API requests. These metrics can be found in the Metrics tab under your Communication Services resource. You can also create permanent dashboards using the workbooks tab under your Communication Services resource.

## Metric definitions

All API request metrics contain three dimensions that you can use to filter your metrics data. These dimensions can be aggregated together using the `Count` aggregation type and support all standard Azure Aggregation time series including `Sum`, `Average`, `Min`, and `Max`.

More information on supported aggregation types and time series aggregations can be found [Advanced features of Azure Metrics Explorer](../../../azure-monitor/essentials/metrics-charts.md#aggregation).

- **Operation** - All operations or routes that can be called on the Azure Communication Services Chat gateway.
- **Status Code** - The status code response sent after the request.
- **StatusSubClass** - The status code series sent after the response. 

### Chat API request metric operations

The following operations are available on Chat API request metrics:

| Operation / Route    | Description                                                                                    |
| -------------------- | ---------------------------------------------------------------------------------------------- |
| GetChatMessage       | Gets a message by message ID. |
| ListChatMessages     | Gets a list of chat messages from a thread. |
| SendChatMessage      | Sends a chat message to a thread. |
| UpdateChatMessage    | Updates a chat message. |
| DeleteChatMessage    | Deletes a chat message. |
| GetChatThread        | Gets a chat thread. |
| ListChatThreads      | Gets the list of chat threads of a user. |
| UpdateChatThread     | Updates a chat thread's properties. |
| CreateChatThread     | Creates a chat thread. |
| DeleteChatThread     | Deletes a thread. |
| GetReadReceipts      | Gets read receipts for a thread. |
| SendReadReceipt      | Sends a read receipt event to a thread, on behalf of a user. |
| SendTypingIndicator           | Posts a typing event to a thread, on behalf of a user. |
| ListChatThreadParticipants    | Gets the members of a thread. |
| AddChatThreadParticipants     | Adds thread members to a thread. If members already exist, no change occurs. |
| RemoveChatThreadParticipant   | Remove a member from a thread. |

:::image type="content" source="../media/chat-metric.png" alt-text="Screenshot showing Chat API Request Metrics.":::

If a request is made to an operation that isn't recognized, you receive a "Bad Route" value response.
## Next steps

- Learn more about [Data Platform Metrics](../../../azure-monitor/essentials/data-platform-metrics.md).
