---
title: "Azure Time Series Insights 環境へのイベントの送信 | Microsoft Docs"
description: "このチュートリアルでは、Time Series Insights 環境にイベントをプッシュする方法について説明します"
keywords: 
services: time-series-insights
documentationcenter: 
author: venkatgct
manager: almineev
editor: cgronlun
ms.assetid: 
ms.service: time-series-insights
ms.devlang: na
ms.topic: get-started-article
ms.tgt_pltfrm: na
ms.workload: big-data
ms.date: 04/21/2017
ms.author: venkatja
ms.translationtype: Human Translation
ms.sourcegitcommit: ef1e603ea7759af76db595d95171cdbe1c995598
ms.openlocfilehash: d7c01e18355b66670c9ab7d964f5cdb7ba72bb8f
ms.contentlocale: ja-jp
ms.lasthandoff: 06/16/2017

---
# <a name="send-events-to-a-time-series-insights-environment-via-event-hub"></a>イベント ハブ経由で Time Series Insights 環境にイベントを送信する

このチュートリアルでは、イベント ハブを作成および構成し、サンプル アプリケーションを実行してイベントをプッシュする方法について説明します。 既に JSON 形式のイベントを含む既存のイベント ハブがある場合は、このチュートリアルをスキップし、[Time Series エクスプローラー](https://insights.timeseries.azure.com)で環境を表示してください。

## <a name="configure-an-event-hub"></a>イベント ハブを構成する
1. イベント ハブを作成するには、イベント ハブに関する[ドキュメント](https://docs.microsoft.com/azure/event-hubs/event-hubs-create)の手順に従います。

2. Time Series Insights のイベント ソースで排他的に使用されるコンシューマー グループを作成していることを確認します。

  > [!IMPORTANT]
  > このコンシューマー グループがその他のサービス (Stream Analytics ジョブや別の Time Series Insights 環境など) で使用されていないことを確認してください。 コンシューマー グループが他のサービスで使用されている場合、この環境および他のサービスの読み取り操作は悪影響を受けます。 "$Default" をコンシューマー グループとして使用している場合、他の閲覧者によって再利用される可能性があります。

  ![イベント ハブ コンシューマー グループの選択](media/send-events/consumer-group.png)

3. イベント ハブに、以下のサンプルでイベントの送信に使用される "MySendPolicy" を作成します。

  ![[共有アクセス ポリシー] を選択して [追加] ボタンをクリックする](media/send-events/shared-access-policy.png)  

  ![[新しい共有アクセス ポリシーの追加]](media/send-events/shared-access-policy-2.png)  

## <a name="create-time-series-insights-event-source"></a>Time Series Insights のイベント ソースを作成する
1. イベント ソースを作成していない場合は、[こちら](time-series-insights-add-event-source.md)に示された指示に従って、イベント ソースを作成します。

2. タイムスタンプ プロパティ名として "deviceTimestamp" を指定します。このプロパティは、以下のサンプルで実際のタイムスタンプとして使用されます。 タイムスタンプ プロパティ名は大文字と小文字が区別されます。また、JSON としてイベント ハブに送信される際、値は __yyyy-MM-ddTHH:mm:ss.FFFFFFFK__ 形式にする必要があります。 このプロパティがイベントに存在しない場合は、イベントがイベント ハブにエンキューされた時刻が使用されます。

  ![イベント ソースの作成](media/send-events/event-source-1.png)

## <a name="run-sample-code-to-push-events"></a>サンプル コードを実行してイベントをプッシュする
1. イベント ハブ ポリシー "MySendPolicy" に移動し、ポリシー キーを含む接続文字列をコピーします。

  ![MySendPolicy の接続文字列のコピー](media/send-events/sample-code-connection-string.png)

2. 3 つの各デバイスにつき 600 件のイベントを送信する次のコードを実行します。 `eventHubConnectionString` は、実際の接続文字列で更新してください。

```csharp
using System;
using System.Collections.Generic;
using System.Globalization;
using System.IO;
using Microsoft.ServiceBus.Messaging;

namespace Microsoft.Rdx.DataGenerator
{
    internal class Program
    {
        private static void Main(string[] args)
        {
            var from = new DateTime(2017, 4, 20, 15, 0, 0, DateTimeKind.Utc);
            Random r = new Random();
            const int numberOfEvents = 600;

            var deviceIds = new[] { "device1", "device2", "device3" };

            var events = new List<string>();
            for (int i = 0; i < numberOfEvents; ++i)
            {
                for (int device = 0; device < deviceIds.Length; ++device)
                {
                    // Generate event and serialize as JSON object:
                    // { "deviceTimestamp": "utc timestamp", "deviceId": "guid", "value": 123.456 }
                    events.Add(
                        String.Format(
                            CultureInfo.InvariantCulture,
                            @"{{ ""deviceTimestamp"": ""{0}"", ""deviceId"": ""{1}"", ""value"": {2} }}",
                            (from + TimeSpan.FromSeconds(i * 30)).ToString("o"),
                            deviceIds[device],
                            r.NextDouble()));
                }
            }

            // Create event hub connection.
            var eventHubConnectionString = @"Endpoint=sb://...";
            var eventHubClient = EventHubClient.CreateFromConnectionString(eventHubConnectionString);

            using (var ms = new MemoryStream())
            using (var sw = new StreamWriter(ms))
            {
                // Wrap events into JSON array:
                sw.Write("[");
                for (int i = 0; i < events.Count; ++i)
                {
                    if (i > 0)
                    {
                        sw.Write(',');
                    }
                    sw.Write(events[i]);
                }
                sw.Write("]");

                sw.Flush();
                ms.Position = 0;

                // Send JSON to event hub.
                EventData eventData = new EventData(ms);
                eventHubClient.Send(eventData);
            }
        }
    }
}

```
## <a name="supported-json-shapes"></a>サポートされている JSON 構造
### <a name="sample-1"></a>サンプル 1

#### <a name="input"></a>入力

単純な JSON オブジェクト。

```json
{
    "deviceId":"device1",
    "deviceTimestamp":"2016-01-08T01:08:00Z"
}
```
#### <a name="output---1-event"></a>出力 - 1 件のイベント

|deviceId|deviceTimestamp|
|--------|---------------|
|device1|2016-01-08T01:08:00Z|

### <a name="sample-2"></a>サンプル 2

#### <a name="input"></a>入力
2 つの JSON オブジェクトを含む JSON 配列。 各 JSON オブジェクトはイベントに変換されます。
```json
[
    {
        "deviceId":"device1",
        "deviceTimestamp":"2016-01-08T01:08:00Z"
    },
    {
        "deviceId":"device2",
        "deviceTimestamp":"2016-01-17T01:17:00Z"
    }
]
```
#### <a name="output---2-events"></a>出力 - 2 件のイベント

|deviceId|deviceTimestamp|
|--------|---------------|
|device1|2016-01-08T01:08:00Z|
|device2|2016-01-08T01:17:00Z|
### <a name="sample-3"></a>サンプル 3

#### <a name="input"></a>入力

2 つの JSON オブジェクトを含む入れ子になった JSON 配列を含む JSON オブジェクト。
```json
{
    "location":"WestUs",
    "events":[
        {
            "deviceId":"device1",
            "deviceTimestamp":"2016-01-08T01:08:00Z"
        },
        {
            "deviceId":"device2",
            "deviceTimestamp":"2016-01-17T01:17:00Z"
        }
    ]
}

```
#### <a name="output---2-events"></a>出力 - 2 件のイベント
"location" プロパティは各イベントにコピーされます。

|location|events.deviceId|events.deviceTimestamp|
|--------|---------------|----------------------|
|WestUs|device1|2016-01-08T01:08:00Z|
|WestUs|device2|2016-01-08T01:17:00Z|

### <a name="sample-4"></a>サンプル 4

#### <a name="input"></a>入力

2 つの JSON オブジェクトを含む入れ子になった JSON 配列を含む JSON オブジェクト。 この入力は、グローバル プロパティが複雑な JSON オブジェクトによって表現できることを示しています。

```json
{
    "location":"WestUs",
    "manufacturerInfo":{
        "name":"manufacturer1",
        "location":"EastUs"
    },
    "events":[
        {
            "deviceId":"device1",
            "deviceTimestamp":"2016-01-08T01:08:00Z",
            "deviceData":{
                "type":"pressure",
                "units":"psi",
                "value":108.09
            }
        },
        {
            "deviceId":"device2",
            "deviceTimestamp":"2016-01-17T01:17:00Z",
            "deviceData":{
                "type":"vibration",
                "units":"abs G",
                "value":217.09
            }
        }
    ]
}
```
#### <a name="output---2-events"></a>出力 - 2 件のイベント

|location|manufacturerInfo.name|manufacturerInfo.location|events.deviceId|events.deviceTimestamp|events.deviceData.type|events.deviceData.units|events.deviceData.value|
|---|---|---|---|---|---|---|---|
|WestUs|manufacturer1|EastUs|device1|2016-01-08T01:08:00Z|pressure|psi|108.09|
|WestUs|manufacturer1|EastUs|device2|2016-01-08T01:17:00Z|vibration|abs G|217.09|

## <a name="next-steps"></a>次のステップ

* [Time Series Insights ポータル](https://insights.timeseries.azure.com)で環境を表示する

