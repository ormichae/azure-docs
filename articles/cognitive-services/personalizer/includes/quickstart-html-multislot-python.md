---
title: include file
description: include file
services: cognitive-services
manager: nitinme
ms.service: cognitive-services
ms.subservice: personalizer
ms.topic: include
ms.custom: cog-serv-seo-aug-2020
ms.date: 03/23/2021
---
[Reference documentation](/python/api/azure-cognitiveservices-personalizer/azure.cognitiveservices.personalizer) | [Library source code](https://github.com/Azure/azure-sdk-for-python/tree/master/sdk/cognitiveservices/azure-cognitiveservices-personalizer) | [Package (pypi)](https://pypi.org/project/azure-cognitiveservices-personalizer/) | [Samples](https://github.com/Azure-Samples/cognitive-services-quickstart-code/tree/master/python/Personalizer)

## Prerequisites

* Azure subscription - [Create one for free](https://azure.microsoft.com/free/cognitive-services)
* [Python 3.x](https://www.python.org/)
* Once you have your Azure subscription, <a href="https://ms.portal.azure.com/#create/Microsoft.CognitiveServicesPersonalizer"  title="Create a Personalizer resource"  target="_blank">create a Personalizer resource </a> in the Azure portal to get your key and endpoint. After it deploys, click **Go to resource**.
    * You will need the key and endpoint from the resource you create to connect your application to the Personalizer API. You'll paste your key and endpoint into the code below later in the quickstart.
    * You can use the free pricing tier (`F0`) to try the service, and upgrade later to a paid tier for production.

## Setting Up

[!INCLUDE [Upgrade Personalizer instance to Multi-Slot](upgrade-personalizer-to-multislot.md)]

[!INCLUDE [Change model frequency](change-model-frequency.md)]

### Create a new python application

Create a new Python file and create variables for your resource's endpoint and subscription key.

[!INCLUDE [Personalizer find resource info](find-azure-resource-info.md)]

```python
import datetime, json, os, time, uuid, requests

# The endpoint specific to your personalization service instance; 
# e.g. https://<your-resource-name>.cognitiveservices.azure.com
PERSONALIZATION_BASE_URL = "https://<REPLACE-WITH-YOUR-PERSONALIZER-ENDPOINT>.cognitiveservices.azure.com"
# The key specific to your personalization service instance; e.g. "0123456789abcdef0123456789ABCDEF"
RESOURCE_KEY = "<REPLACE-WITH-YOUR-PERSONALIZER-KEY>"
```

## Object model

[comment]: <> (TODO: change links to ccb docs and add personalizer client info once sdk is ready)

To ask for the single best item of the content for each slot, create a [rank_request], then send a post request to the [multislot/rank] endpoint (/dotnet/api/microsoft.azure.cognitiveservices.personalizer.personalizerclientextensions.rank). The response is then parsed into a [rank_response].

To send a reward score to Personalizer, create a [rewards](/dotnet/api/microsoft.azure.cognitiveservices.personalizer.models.rewardrequest), then send a pose request to [multislot/events/{eventId}/reward](/dotnet/api/microsoft.azure.cognitiveservices.personalizer.personalizerclientextensions.reward).

Determining the reward score, in this quickstart is trivial. In a production system, the determination of what impacts the [reward score](../concept-rewards.md) and by how much can be a complex process, that you may decide to change over time. This design decision should be one of the primary decisions in your Personalizer architecture.

## Code examples

These code snippets show you how to do the following tasks by sending HTTP requests for Python:

* [Create base URL's](#create-base-URL's)
* [Rank API](#request-the-best-action)
* [Reward API](#send-a-reward)

## Create Base URL's

In this section you'll do two things:
* Construct the Rank and Reward URL's
* Construct the rank/reward request headers

Construct the Rank / Reward URL's using the base url and the request headers using the resource key.

```python
RANK_URL = '{0}personalizer/v1.1-preview.1/multislot/rank'.format(PERSONALIZATION_BASE_URL)
REWARD_URL_BASE = '{0}personalizer/v1.1-preview.1/multislot/events/'.format(PERSONALIZATION_BASE_URL)
HEADERS = {
    'apim-subscription-id': RESOURCE_KEY,
    'Content-Type': 'application/json'
}
```

## Get content choices represented as actions

Actions represent the content choices from which you want Personalizer to select the best content item. Add the following methods to the script to represent the set of actions and their features. 

```python
def get_actions():
    return [
        {
            "id": "Red-Polo-Shirt-432",
            "features": [
                {
                    "onSale": "true",
                    "price": 20,
                    "category": "Clothing"
                }
            ]
        },
        {
            "id": "Tennis-Racket-133",
            "features": [
                {
                    "onSale": "false",
                    "price": 70,
                    "category": "Sports"
                }
            ]
        },
        {
            "id": "31-Inch-Monitor-771",
            "features": [
                {
                    "onSale": "true",
                    "price": 200,
                    "category": "Electronics"
                }
            ]
        },
        {
            "id": "XBox-Series X-117",
            "features": [
                {
                    "onSale": "false",
                    "price": 499,
                    "category": "Electronics"
                }
            ]
        }
    ]

```

## Get user preferences for context

Add the following methods to the script to get a user's input from the command line for the time of day and the type of device the user is on. These will be used as context features.

```python
def get_context_features():
    time_features = ["morning", "afternoon", "evening", "night"]
    time_pref = input("What time of day is it (enter number)? 1. morning 2. afternoon 3. evening 4. night\n")
    try:
        parsed_time = int(time_pref)
        if(parsed_time <=0 or parsed_time > len(time_features)):
            raise IndexError
        time_of_day = time_features[parsed_time-1]
    except (ValueError, IndexError):
        print("Entered value is invalid. Setting feature value to", time_features[0] + ".")
        time_of_day = time_features[0]

    device_features = ['mobile', 'tablet', 'desktop']
    device_pref = input("What type of device is the user on (enter number)? 1. mobile 2. tablet 3. desktop\n")
    try:
        parsed_device = int(device_pref)
        if(parsed_device <=0 or parsed_device > len(device_features)):
            raise IndexError
        device = device_features[parsed_device-1]
    except (ValueError, IndexError):
        print("Entered value is invalid. Setting feature value to", taste_features[0]+ ".")
        device = taste_features[0]

    return [
        {'time': time_of_day},
        {'device': device}
        ]
```

## Get slots

Slots make up the the page which the user will interact with. Personalizer will decide which action to display in each one of the defined slots. Actions can be excluded from specific slots, shown as `ExcludeActions`. `BaselineAction` is the default action for the slot which would have been displayed without the use of Personalizer.


[comment]: <> (Need to add the slot documentation links)
This quickstart has simple slot features. In production systems, determining and [evaluating](../concept-feature-evaluation.md) [slots and features](../concepts-features.md) can be a non-trivial matter.

```python
def get_slots():
    return [
        {
            "id": "BigHeroPosition",
            "features": [
                {
                    "size": "large",
                    "position": "left",
                }
            ],
            "excludedActions": ["31-Inch-Monitor-771"],
            "baselineAction": "Red-Polo-Shirt-432"
        },
        {
            "id": "SmallSidebar",
            "features": [
                {
                    "size": "small",
                    "position": "right",
                }
            ],
            "excludedActions": ["Tennis-Racket-133"],
            "baselineAction": "XBox-Series X-117"
        }
    ]
```

## Make HTTP requests

Send post requests to the Personalizer endpoint for multi-slot rank and reward calls.

```python
def send_rank(rank_request):
    response = requests.post(RANK_URL, data=json.dumps(rank_request), headers=HEADERS )
    return json.loads(response.text)
```

```python
def send_reward(reward_request, event_id):
    reward_url = '{0}{1}/reward'.format(REWARD_URL_BASE, event_id)
    res = requests.post(reward_url, data=json.dumps(reward_request), headers=HEADERS)
```

## Get feedback for personalizer decisions

Add the following method to the script. You will signal if Personalizer made a good decision for each slot through command line prompt.

```python
def get_reward_for_slot():
    answer = input('\nIs this correct? (y/n)\n').upper()
    if (answer == 'Y'):
        print('\nGreat! The application will send Personalizer a reward of 1 so it learns from this choice of action for this slot.\n')
        return 1
    elif (answer == 'N'):
        print('\nYou didn\'t like the recommended item.The application will send Personalizer a reward of 0 for this choice of action for this slot.\n')
        return 0
    print('\nEntered choice is invalid. Service assumes that you didn\'t like the recommended item.\n')
    return 0
```

## Create the learning loop

The Personalizer learning loop is a cycle of [Rank](#request-the-best-action) and [Reward](#send-a-reward) calls. In this quickstart, each Rank call, to personalize the content, is followed by a Reward call to tell Personalizer how well the service performed.

The following code loops through a cycle of asking the user their preferences through the command line, sending that information to Personalizer to select the best action for each slot, presenting the selection to the customer to choose from among the list, then sending a reward score to Personalizer signaling how well the service did in its selection.

```python
run_loop = True

while run_loop:

    eventId = str(uuid.uuid4())
    context = get_context_features()
    actions = get_actions()
    slots = get_slots()

    rank_request = {
        "eventId": eventId,
        "contextFeatures": context,
        "actions": actions,
        "slots": slots,
        "deferActivation": False
      }

    #Rank the actions for each slot
    rank_response = send_rank(rank_request)
    rewards = {"reward": []}

    for i in range(len(rank_response['slots'])):
        print('\nPersonalizer service decided you should display: {0} in slot {1}\n'.format(rank_response['slots'][i]['rewardActionId'], rank_response['slots'][i]['id']))

        slot_reward = {'slotId': rank_response['slots'][i]['id']}
        # User agrees or disagrees with Personalizer decision for slot
        slot_reward['value'] = get_reward_for_slot()
        rewards['reward'].append(slot_reward)

    # Send the rewards for the event
    send_reward(rewards, rank_response['eventId'])

    answer = input('\nPress q to break, any other key to continue:\n').upper()
    if (answer == 'Q'):
        run_loop = False
```

Take a closer look at the rank and reward calls in the following sections.

Add the following methods, which [get the content choices](#get-content-choices-represented-as-actions), [get user preferences for context](#get-user-preferences-for-context), [get the slots](#get-slots), [Make HTTP requests](#make-HTTP-requests), [Get reward for each slot](#get-feedback-for-personalizer-decisions) before running the code file:

* get_actions
* get_context_features
* get_slots
* send_rank
* send_reward
* get_reward_for_dsot

## Request the best action

To complete the Rank request, the program asks the user's preferences to create content choices. The request body contains the context features, actions and their features, slots and their features, and a unique event ID, to receive the response. The `send_rank` method needs the rank_equest to send the multi-slot rank request.

This quickstart has simple context features of time of day and user device. In production systems, determining and [evaluating](../concept-feature-evaluation.md) [actions and features](../concepts-features.md) can be a non-trivial matter.

```python
eventId = str(uuid.uuid4())
    context = get_context_features()
    actions = get_actions()
    slots = get_slots()

    rank_request = {
        "eventId": eventId,
        "contextFeatures": context,
        "actions": actions,
        "slots": slots,
        "deferActivation": False
      }

    #Rank the actions for each slot
    rank_response = send_rank(rank_request)
```

## Send a reward

To get the reward score to send in the Reward request, the program gets the user's selection for each slot through the command line, assigns a numeric value to the selection, then sends the unique event ID, slot ID, and the reward score for each slot as the numeric value to the Reward API. Note that a reward does not need to be defined for each slot.

This quickstart assigns a simple number as a reward score, either a zero or a 1. In production systems, determining when and what to send to the [Reward](../concept-rewards.md) call can be a non-trivial matter, depending on your specific needs.

```python
rewards = {"reward": []}

for i in range(len(rank_response['slots'])):
    print('\nPersonalizer service decided you should display: {0} in slot {1}\n'.format(rank_response['slots'][i]['rewardActionId'], rank_response['slots'][i]['id']))

    slot_reward = {'slotId': rank_response['slots'][i]['id']}
    # User agrees or disagrees with Personalizer decision for slot
    slot_reward['value'] = get_reward_for_slot()
    rewards['reward'].append(slot_reward)

# Send the rewards for the event
send_reward(rewards, rank_response['eventId'])
```

## Run the program

Run the application with the python from your application directory.

```console
python sample.py
```

![The quickstart program asks a couple of questions to gather user preferences, known as features, then provides the top action.](../media/csharp-quickstart-commandline-feedback-loop/multislot-quickstart-program-feedback-loop-example.png)


[comment]: <> (Need to add a link to the multi slot sample source code)
The [source code for this quickstart](https://github.com/Azure-Samples/cognitive-services-quickstart-code/tree/master/dotnet/Personalizer) is available.
