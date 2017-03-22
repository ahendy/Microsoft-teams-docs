﻿# Starting a conversation

Bots in Microsoft Teams allow private conversations with a single user or a group conversation in a Teams channel.

## Initiating conversation

Microsoft Teams currently supports three methods for conversation:
* One-on-One Response - Users can interact in a private conversation with a bot by simply selecting the added bot in the chat history, or typing its name or Bot ID in the To: box on a new chat.
* In Channel Response - A bot can also be @mentioned in a channel if it has been added to the team.  Note that additional replies to a bot in a channel require @mentioning the bot - it will not respond to replies where it is not @mentioned.
* In Channel Conversation Creation - A bot in a channel may also initiate a new conversation in a channel.

Note that bots in private group chats are currently not supported.


## One-on-One conversations

### Receiving messages

For a 1:1 conversation with a bot, the regular [Activity](https://docs.botframework.com/en-us/core-concepts/reference/#activity) message schema will apply.

Of note:
* `channelId` - this value, which specifics with Bot Framework channel the message is coming from, will always be `msteams`
* `from.id` - This is a unique and encrypted id for that user for your bot, and is suitable as a key should your app wish to store user data.  Note, though, that this is unique for your bot and cannot be directly used outside your bot instance in any meaningful way to identify that user.

### Replying to message
In order to reply to an existing message, you simply need to call the `ReplyToActivity()` in [C#](https://docs.botframework.com/en-us/csharp/builder/sdkreference/routing.html#replying) or 'session.send' in [Node.JS](https://docs.botframework.com/en-us/node/builder/chat/session/#sending-messages).  The BotFramework SDK handles all the details.

If you choose to use the REST API, you can also call the [/conversations/{conversationId}/activities/{activityId}`](https://docs.botframework.com/en-us/restapi/connector/#/Conversations) endpoint. Please note:  if you are constructing the outgoing payload yourself, always use the `serviceUrl` received in the inbound messages as your outbound `serviceUrl`.

## One to Many - Bots in Channels

### Receiving messages

For a bot in a channel, in addition to the [regular message schema](https://docs.botframework.com/en-us/core-concepts/reference/#activity), your bot will also receive the following properties:

* `channelData` - see [below](#teams-specific-functionality).
* `conversationData.id` - this value is the reply chain ID, consisting of channel id plus the id of the first message in the reply chain. 
* `conversationData.isGroup` - this value will be set to `true` for bot messages in channels
* `entities` - this object may contain one or more Mention objects (see [below](#mentions))

Please note that channelData should be used as the definitive information for team and channel Ids, for your use in cacheing and utilizing as key local storage.

### Replying to message
Replying to a message in a channel is the same as in 1:1.  Note that replying to a message in a channel shows as a reply to the initiating reply chain - for bots in channels, the conversationId contains channel and the top level message id.  While the BotFramework takes care of the details, you may cache that conversationId for future replies to that conversation thread as needed.  

## Mentions

Bots have the ability to be retrieve and construct @mentions in a message, and to be triggered in channel have to be @mentioned themselves to receive a message.  The users in question, including the bot itself in channels, are passed in the `entities` object, with the following properties:

* **`type`** - the string "mention"
* **`mentioned.id`** - the GUID for the user, which is unique for your bot
* **`mentioned.name`** - the name of the user.  Note, at this time because we do not provide an API to return profile information, you can either obtain this value from a received message or from an external lookup, e.g. Azure ActiveDirectory
* **`text`** - the @mention text in the body of the message itself, which may or may not be the full name of that user due to Teams' ability to shorten (hit *backspace* when doing @mention name support).  Note that the text here, like in the body of the message, will be wrapped with the <at></at> tag.

#### Example Entities object

```json
"entities": [ 
    { 
        "type":"mention", 
        "mentioned":{ 
            "id":"29:08q2j2o3jc09au90eucae",
            "name":"Larry Jin" 
        }, 
        "text": "<at>@Larry Jin</at>"
    } 
] 
```

### Retrieving mentions


You can retrieve all mentions in the message by calling the `GetMentions()` function in the BotFramework C# SDK.  This should return an array of all the @mentions sent in the `entities` section of the schema.

Note that bots in a channel only respond if @mentioned and therefore the body of the text message will always include the @Bot name.  Ensure your message parsing excludes that.  For example:

```c#
Mention[] m = sourceMessage.GetMentions();
var messageText = sourceMessage.Text;

for (int i = 0;i < m.Length;i++)
{
    if (m[i].Mentioned.Id == sourceMessage.Recipient.Id)
    {
        //Bot is in the @mention list.  
        //The below example will strip the bot name out of the message, so you can parse it as if it wasn't included.  Note that the Text object will contain the full bot name, if applicable.
        if (m[i].Text != null)
            messageText = messageText.Replace(m[i].Text, "");
    }
}
```

### Constructing mentions

Your bot can @mention other users in messages posted into channels. To do this, your message must:
* Include <at>@username</at> in the message text. Note, at this time because we do not provide an API to return profile information, you can either obtain this value from a received message or from an external lookup.
* Include the `mention` object inside the `entities` collection

#### Schema example

```json
{ 
    "type": "message", 
    "text": "Hey <at>Larry Jin</at> check out this message", 
    "timestamp": "2016-10-29T00:51:05.9908157Z", 
    "serviceUrl": "https://skype.botframework.com", 
    "channelId": "msteams", 
    "from": { 
        "id": "28:9e52142b-5e5e-4d7b-bb3e- e82dcf620000", 
        "name": "SchemaTestBot" 
    }, 
    "conversation": { 
        "id": "19:aebd0ad4d6ab42c8b9ed19c251c2fc37@thread.skype;messageid=1481567603816" 
    }, 
    "recipient": { 
        "id": "8:orgid:6aebbad0-e5a5-424a-834a-20fb051f3c1a", 
        "name": "stlrgload100" 
    }, 
    "attachments": [ 
        { 
            "contentType": "image/png", 
            "contentUrl": "https://upload.wikimedia.org/wikipedia/en/a/a6/Bender_Rodriguez.png", 
            "name": "Bender_Rodriguez.png" 
        } 
    ], 
    "entities": [ 
        { 
            "type":"mention", 
            "mentioned":{ 
                "id":"29:08q2j2o3jc09au90eucae",
                "name":"Larry Jin" 
            }, 
            "text": "<at>@Larry Jin</at>"
        } 
    ], 
    "replyToId": "3UP4UTkzUk1zzeyW" 
}
```

## Channel Conversation Creation

Once your bot has been added to the team, it can also post into a channel to create a new reply chain.  With the BotBuilder SDK, call  `CreateConversation()` for [C#](https://docs.botframework.com/en-us/csharp/builder/sdkreference/routing.html#conversationmultiple) or utilize Proactive Messaging techniques (`bot.send()` and `bot.beginDialog()`) in [Node.JS](https://docs.botframework.com/en-us/node/builder/chat/UniversalBot/#proactive-messaging).  

Alternatively, you can issue a POST request to the [`/conversations/{conversationId}/activities/`]() resource.

Note: at this point, bots in Microsoft Teams cannot initiate 1:1 / direct or 1:many / group conversations.

### Example (C#)
```csharp
var channelData = new Dictionary<string, string>();
channelData["teamsChannelId"] = yourTeamsChannelID;
IMessageActivity newMessage = Activity.CreateMessageActivity();
newMessage.Type = ActivityTypes.Message;
newMessage.Text = "Hello channel.  This is a newly created reply chain.";
ConversationParameters conversationParams = new ConversationParameters(
    isGroup: true,
    bot: null,
    members: null,
    topicName: "Test Conversation",
    activity: (Activity)newMessage,
    channelData: channelData);
var result = await connector.Conversations.CreateConversationAsync(conversationParams);
```

## Teams-specific functionality

Bots in Teams have channel-specific data and functionality that you can leverage to differentiate your bot experience in Teams.  This includes such functionality as retrieving team and channel information, as well as [channel events](botevents.md) sent to your bot by Teams.

### Teams channel data

Teams-specific information is sent and received in the `channelData` object, which has been updated since Preview launch.

#### Example channelData object (channelCreated event)
```json
"channelData": {
    "channel": {
        "id": "19:693ecdb923ac4458a5c23661b505fc84@thread.skype",
        "name": "My New Channel"
    },
    "eventType": "channelCreated",
    "team": {
        "id": "19:693ecdb923ac4458a5c23661b505fc84@thread.skype"
    },
    "tenant": {
        "id": "72f988bf-86f1-41af-91ab-2d7cd011db47"
    }
}
```

`channelData` objects:
* `channel` - this object is passed only in channel contexts, when the bot is @mentioned or for events in channels in teams where the bot has been added.
    - `id` - the GUID for the channel.
    - `name` - the channel name, passed only in cases of [channel modification events](botevents.md). 
* `eventType` - Teams event type - passed only in cases of [channel modification events](botevents.md).
* `team` - this object is passed only in channel contexts, not 1:1.
    - `id` - the GUID for the channel.
    - Note: there is no name value being passed at this time
* `tenant.id` - the O365 tenant id on which Teams is running.  This is passed in all contexts.

>**Note:** the payload also may contain `channelData.teamsTeamId` and `channelData.teamsChannelId` properties for backwards compatibility.  These should be deprecated in favor of the above.

### Fetching the team roster
Your bot can query for the list of team members. With the BotBuilder SDK, call  `GetConversationMembersAsync()` for [C#](https://docs.botframework.com/en-us/csharp/builder/sdkreference/d7/d08/class_microsoft_1_1_bot_1_1_connector_1_1_conversations_extensions.html#a0a665865891d485956e52c64bce84d4b) to return a list of userIds for the `team.id` retrieved from the `channelData` of the inbound schema.

```csharp
ChannelAccount[] members = connector.Conversations.GetConversationMembers(sourceMessage.Conversation.Id);

replyMessage.Text = "These are the member userids returned by the GetConversationMembers() function";

for (int i = 0; i < members.Length; i++)
{
    replyMessage.Text += "<br />" + members[i].Id; //Not currently supported: members[i].Name;
}
```

Alternatively, you can issue a GET request to the [`/conversations/{teamId}/members/`](https://docs.botframework.com/en-us/restapi/connector/#!/Conversations/Conversations_GetConversationMembers) resource, using `teamId` in the `conversationId` field in the API call.

Currently, this only returns the userIds but we will be including profile information in the future.


## Full inbound Schema example (bot in a channel)
```json
{
    "type": "message",
    "id": "1485983408511",
    "timestamp": "2017-02-01T21:10:07.437Z",
    "serviceUrl": "https://smba.trafficmanager.net/amer-client-ss.msg/",
    "channelId": "msteams",
    "from": {
        "id": "29:1XJKJMvc5GBtc2JwZq0oj8tHZmzrQgFmB39ATiQWA85gQtHieVkKilBZ9XHoq9j7Zaqt7CZ-NJWi7me2kHTL3Bw",
        "name": "Richard Moe"
    },
    "conversation": {
        "id": "19:253b1f341670408fb6fe51050b6e6ceb@thread.skype;messageid=1485983194839"
    },
    "recipient": {
        "id": "28:c9e8c047-2a74-40a2-b28b-b262d5f5327c",
        "name": "Teams TestBot"
    },
    "textFormat": "plain",
    "text": "<at>Teams TestBot</at> Hello <at>Jingjing Xia</at>",
    "attachments": [
    {
        "contentType": "text/html",
        "content": "<div><span contenteditable=\"false\" itemscope=\"\" itemtype=\"http://schema.skype.com/Mention\" itemid=\"0\">Teams TestBot</span> Hello <span contenteditable=\"false\" itemscope=\"\" itemtype=\"http://schema.skype.com/Mention\" itemid=\"1\">Jingjing Xia</span></div>"
    }
    ],
    "entities": [
    {
        "type": "mention",
        "mentioned": {
            "id": "28:c9e8c047-2a74-40a2-b28b-b262d5f5327c",
            "name": "Teams TestBot"
        },
        "text": "<at>Teams TestBot</at>"
    },
    {
    "type": "mention",
    "mentioned": {
            "id": "29:1jnFbZYs0qXMLH-O4S9-sDLNc3NVEIMWWnC-q0tVdEa-8BRosfojI35QdNoB-yW8iutWLJzHUm_mqEZSSU8si0Q",
            "name": "Jingjing Xia"
        },
        "text": "<at>Jingjing Xia</at>"
    },
    {
        "type": "clientInfo",
        "locale": "en-US",
        "country": "US",
        "platform": "Windows"
    }
    ],
    "channelData": {
        "teamsChannelId": "19:253b1f352670408fb6fe51050b6e5ceb@thread.skype",
        "teamsTeamId": "19:712c61d0ef393e5fa681ba90ca943398@thread.skype",
        "channel": {
            "id": "19:253b1f352670408fb6fe51050b6e5ceb@thread.skype"
        },
        "team": {
            "id": "19:712c61d0ef393e5fa681ba90ca943398@thread.skype"
        },
        "onBehalfOf": "[]",
        "tenant": {
            "id": "72f988bf-86f1-41af-91ab-2d7cd011db47"
        }
    }
}
```




