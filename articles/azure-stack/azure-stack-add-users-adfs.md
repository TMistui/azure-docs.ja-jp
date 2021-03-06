---
title: Add users for Azure Stack ADFS | Microsoft Docs
description: Learn how to add users for ADFS deployments of Azure Stack
services: azure-stack
documentationcenter: 
author: HeathL17
manager: byronr
editor: 
ms.service: azure-stack
ms.workload: na
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: article
ms.date: 3/1/2017
ms.author: helaw
translationtype: Human Translation
ms.sourcegitcommit: c00512e52eefbb95987fd989b330b1b48c5a9d29
ms.openlocfilehash: 5f7febb10f776d6a314c98a2ec476725e7b07f6f
ms.lasthandoff: 03/01/2017


---
# <a name="add-users-in-the-azure-stack-poc"></a>Add users in the Azure Stack POC

To add more users in the POC deployment, you must add them to the Azure Stack POC AD using Microsoft Management Console from the Azure Stack host computer.
1.    On the Azure Stack host computer, open Microsoft Management Console.
2.    Click **File > Add or remove snap-in**.
3.    Select **Active Directory Users and Computers** > **AzureStack.local** > **Users**.
4.    Click **Action** > **New** > **User**.
5.    In the New Object – User window, provide and confirm a password
6.    To avoid requirements to change the password for the user you just created, uncheck **User must change password at next logon** and check **Password never expires**.
7.    Click **Next** to finalize the values and click Finish to create the user.



