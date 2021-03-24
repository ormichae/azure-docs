---
title: include file
description: include file
services: cognitive-services
manager: nitinme
ms.service: cognitive-services
ms.subservice: personalizer
ms.topic: include
ms.custom: include file
ms.date: 03/23/2021
---
### Upgrade Personalizer instance to multislot

In the Azure portal, in the Personalizer resource on the **Model and learning settings** page, click **Export learning settings**. The **arguments** field in the downloaded json file will start with **--cb_explore_adf**. Change this to **--ccb_explore_adf** while leaving the rest of the file untouched. Save the file. 

![Learning settings before change](../media/settings/learning-settings-pre-upgrade.png)

![Learning settings after change](../media/settings/learning-settings-post-upgrade.png)

In the same tab in the portal, under **import learning settings** browse to find your recently modified json file and upload it. This will update your Personalizer instance to be a "Multi Slot" Personalizer and will now support multi-slot Rank and Reward calls.