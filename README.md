# Universal Bot action model

## Context

Adaptive Cards are platform-agnostic snippets of UI, authored using a lightweight JSON format, that apps and services can share. Adaptive Cards not only adapt to the look-and-feel of the host, but also provide rich interaction capabilities. For more information about Adaptive Cards, please visit [adaptivecards.io](http://adaptivecards.io). 

As Adaptive Cards grew in popularity, different hosts started supporting different action models and this led to fragmentation. To solve this problem, the Teams, Outlook and Adaptive Cards teams worked on creating a new universal Bot action model compatible across hosts. This effort lead to the following:
- The generalization of Bots and the Bot Framework as the way to implement Adaptive Card-based scenarios for both Teams (Bots) and Outlook (Actionable Message)
- `Action.Execute` as a replacement for both `Action.Submit` (currently used by Bots) and `Action.Http` (currently used by Actionable Messages)
- Popular features only supported by Actionable Messages made available to Bots, namely:
  - The ability for a card to be refreshed at the time it is displayed
  - The ability for `Action.Execute` actions to return an updated card to be immediately displayed in the client

For more information about Actionable Messages in Outlook, please refer to the [Actionable Message documentation](https://docs.microsoft.com/en-us/outlook/actionable-messages/send-via-email)

Before `Action.Execute` |  With `Action.Execute`
:-------------------------:|:-------------------------:
![Untitled.png](Untitled.png) | ![Slide28.jpg](Slide28.jpg)
![Untitled%201.png](Untitled%201.png) | ![Slide24.jpg](Slide24.jpg)

Source: [Adaptive Cards @ Microsoft Build 2020](https://youtu.be/hEBhwB72Qn4?t=1393)

The rest of this document focuses on documenting the universal Bot action model in the context of Teams & Outlook. If you are already using Adaptive cards on Teams with Bot, you can use  the same Bot with a few changes to support `Action.Execute`. If you are using Actionable Messages on Outlook, you will need to develop a Bot that supports `Action.Execute`. Currently the support on Outlook clients for Universal Bot action model is under active development.

## Schema

### Action.Execute

When authoring Adaptive Cards, use `Action.Execute` in place of both `Action.Submit` &  `Action.Http`. The schema for `Action.Execute` is quite similar to that of `Action.Submit`.

**Example JSON**
```json
{
  "$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
  "type": "AdaptiveCard",
  "version": "2.0",
  "body": [
    {
      "type": "TextBlock",
      "text": "Present a form and submit it back to the originator"
    },
    {
      "type": "Input.Text",
      "id": "firstName",
      "placeholder": "What is your first name?"
    },
    {
      "type": "Input.Text",
      "id": "lastName",
      "placeholder": "What is your last name?"
    },
    {
      "type": "ActionSet",
      "actions": [
        {
          "type": "Action.Execute",
          "title": "Submit",
          "verb": "personalDetailsFormSubmit",
          "fallback": "Action.Submit"
        }
      ]
    }
  ]
}
```

**Properties**

| Property | Type | Required | Description 
| -------- | ---- | -------- | ----------- 
| **type** | `"Action.Execute"` | Yes | Must be `"Action.Execute"`. |
| **verb** | string | No | A convenience string that can be used by developer to identify the action |
| **data** | `string`, `object` | No | Initial data that input fields will be combined with. These are essentially ‘hidden’ properties. |
| **title** | `string` | No | Label for button or link that represents this action. |
| **iconUrl** | `uri` | No | Optional icon to be shown on the action in conjunction with the title. Supports data URI in Adaptive Cards version 1.2+ |
| **style** | `ActionStyle` | No | Controls the style of an Action, which influences how the action is displayed, spoken, etc. |
| **fallback** | `<action object>`, `"drop"` | No | Describes what to do when Action.Execute is unsupported by the client displaying the card. |
| **requires** | `Dictionary<string>` | No | A series of key/value pairs indicating features that the item requires with corresponding minimum version. When a feature is missing or of insufficient version, fallback is triggered. |


### Refresh mechanism

Alongside `Action.Execute`, a new refresh mechanism is now supported, making it possible to create Adaptive Cards that automatically update at the time they are displayed. This ensures that users always see up to date data. A typical refresh use case is an approval request: once approved, it is best that users are not presented with a card prompting them to approve when it's already been done, but instead provides information on the time the request was approved and by whom.

To allow an Adaptiver Card to automatically refresh, define its `refresh` property, which embeds an `action` of type `Action.Execute` as well as a `userIds` array.

#### IMPORTANT
If the `userIds` list property isn't included in the `refresh` section of the card, the card will NOT be automatically refresh on display. Instead, a button will be presented to the user to allow them to manually refresh. The reason for this is Channels in Teams can include a large number of members; if many members are all viewing the channel at the same time, and unconditional automatic refresh would results in many concurrent calls to the Bot, which would not scale. To alleviate the potential scale problem, the `userIds` property should always be included to identify which users should get an automatic refresh, with a maximum of **5** user IDs currently being allowed.

Note thae the `userIds` property is ignored in Outlook, and the `refresh` property is always automatically honored. There is no scale issue in Outlook because users will typically view the card at different times.

| Property | Type | Required | Description 
| -------- | ---- | -------- | ----------- 
| **action** | `"Action.Execute"` | Yes | Must be an action instance of type `"Action.Execute"`. |
| **userIds** | `Array<string>` | Yes | An array of `MRI`s of users for whom Auto Refresh must be enabled |

**Sample JSON**
```JSON
{
  "$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
  "type": "AdaptiveCard",
  "originator":"c9b4352b-a76b-43b9-88ff-80edddaa243b",
  "version": "2.0",
  "refresh": {
    "action": {
      "type": "Action.Execute",
      "title": "Submit",
      "verb": "personalDetailsCardRefresh"
    },
    "userIds": []
  },
  "body": [
    {
      "type": "TextBlock",
      "text": "Present a form and submit it back to the originator"
    },
    {
      "type": "Input.Text",
      "id": "firstName",
      "placeholder": "What is your first name?"
    },
    {
      "type": "Input.Text",
      "id": "lastName",
      "placeholder": "What is your last name?"
    },
    { 
      "type": "ActionSet",
      "actions": [
        {
          "type": "Action.Execute",
          "title": "Submit",
          "verb": "personalDetailsFormSubmit",
          "fallback": "Action.Submit"
        }
      ]
    }
  ]
}
```

#### IMPORTANT - ORIGINATOR Field

The `originator` is an unique identifier(GUID) used to identify the partner. This is generated when a partner subscribes to outlook as a channel. This is an important field that is checked on outlook clients before the adaptive cards are rendered. This field is ignored on teams.

### `adaptiveCard/action` Invoke activity

When an `Action.Execute` is executed in the client (whether it's the refresh action or an action explicitly taken by a user by clicking a button), a new type of Invoke activity - `adaptiveCard/action` - is made to your Bot. A typical `adaptiveCard/action` Invoke activity request will look like the following:

#### Request format

```
{ 
  "type": "invoke",
  "name": "adaptiveCard/action",

  // ... other properties omitted for brevity

  "value": { 
    "action": { 
      "type": "Action.Execute", 
      "id": "abc", 
      "verb": "def",
      "data": { ... } 
    },
    "trigger": "automatic | manual" 
  }
} 
```

| Field | Description |
| -------- | ----------- |
| **value.action** | A copy of the action as defined in the Adaptive Card. Like with `Action.Submit`, the `data` property of the action includes the values of the various inputs in the card, if there are any |
| **value.trigger** | Indicates if the action was triggered explicitly (by the user clicking a button) or implicitly (via automatic refresh) |

#### Response format

```JSON
{ 
    "statusCode": <number (200 – 599)>, 
    "type": "<string>", 
    "value": "<object>"
} 
```

| Field | Description |
| -------- | ----------- |
| **statusCode** | This field is a number ranging from 200-599 that mirrors HttpStatusCode values and is meant to be a sub-status for the result of the bot processing the Invoke. A missing, null, or undefined value for statusCode implies a 200 (Success).   |
| **type** | A set of well-known string constants that describe the expected shape of the value property  |
| **value** | An object that is specific to the type of response body  |

**Status codes**
If the Bot processed the request (i.e. if the Bot's code was involved at all in processing the request), the HTTP response's status code will be equal to 200. Otherwise, it will be in either the `4xx` or `5xx` range. An HTTP status code of 200 however does not necessarily mean that the Bot successfully processed the request. A client application should always look at the statucCode property in the response's body in addition to the HTTP response status code.

The below table lists the various statucCode that can be found in the response body and their meaning. Note that in the absence of the `statusCode` property, it is assumed to be 200.

| statusCode | Description |
| ----------| ----------- |
| 200 | The Bot successfully processed the request. The `type` and `value` contain the actual result |
| TODO: Other codes | TODO: Description for other codes |

The below table lists the values the `type` property might have. Each `type` value identifies the type of data in the `value` property:
| type |
| ---- |
| application/vnd.microsoft.activity.message | The `value` property contains a string message. |
| application/vnd.microsoft.card.adaptive | The `value` property contains a serialized Adaptive Card payload meant to be a replacement to the Adaptive Card that was at the origin of the action. |

## Summary: how to leverage the universal Bot action model

1. Use `Action.Execute` instead of `Action.Submit`. To update an existing scenario on teams, replace all instances of `Action.Submit` with `Action.Execute`. For upgrading an existing scenario on Outlook please refer the backward compatibility section below.
2. For cards to surface on outlook add the `originator` field. Refer the Sample JSON above.  
3. Add a `refresh` clause to your Adaptive Card if you want to leverage the automatic refresh mechanism or if your scenario requires contextual views. Be sure to specify the `userIds` property to identify which users (maximum 5) will get automatic updates. 
4. Handle `adaptiveCard/action` Invoke requests in your Bot
5. Whenever your Bot needs to return a new card as a result of processing an `Action.Execute`, you can use the Invoke request's context to generate cards that are specifically crafted for a given user. Make sure the response conforms to the response schema defined above.

## Backward compatibility - Outlook
Actionable messages on outlook supports `Action.Http` (which is a REST end point) whereas Universal Bot action model supports `Action.Execute`(which is a bot end point). Partners that want to leverage the features of Universal Bot action model need to implement a bot and subscribe to `Outlook Actionable Messages` as a channel. Work is in progress to allow for seamless migration for existing Actionable message partners to upgrade to Universal Bot action model.

## Backward compatibility - Teams
In order for your cards to be backward compatible and work for users on older versions of Teams, your `Action.Execute` actions should include a `fallback` property defined as an `Action.Submit`. Your Bot should be coded in such a way that it can process both `Action.Execute` and `Action.Submit`. Note that it is not possible for your Bot to return a new card when processing an `Action.Submit`, so fallback behavior via `Action.Submit` will provide a degraded experience for the end user.

> ### Important Note
> Some older Teams clients do not support fallback property when not wrapped in an `ActionSet`. In order to not break on such clients, it is **strongly recommended** that you wrap _all_ your `Action.Execute` in `ActionSet`. See example below on how to wrap `Action.Execute` in `ActionSet`.
 
```JSON
{
  "$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
  "type": "AdaptiveCard",
  "version": "1.0",
  "body": [
    {
      "type": "TextBlock",
      "text": "Present a form and submit it back to the originator"
    },
    {
      "type": "Input.Text",
      "id": "firstName",
      "placeholder": "What is your first name?"
    },
    {
      "type": "Input.Text",
      "id": "lastName",
      "placeholder": "What is your last name?"
    },
    {
      "type": "ActionSet",
      "actions": [
        {
          "type": "Action.Execute",
          "title": "Submit",
          "verb": "personalDetailsFormSubmit",
          "fallback": "Action.Submit"
        }
      ]
    }
  ]
}
```

## References
- [Adaptive Cards @ Microsoft Build 2020](https://youtu.be/hEBhwB72Qn4?t=1393)
- [Adaptive Cards @ Ignite 2020](https://techcommunity.microsoft.com/t5/video-hub/elevate-user-experiences-with-teams-and-adaptive-cards/m-p/1689460)



































