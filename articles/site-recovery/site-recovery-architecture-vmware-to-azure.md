---
title: "Azure Site Recovery での Azure への VMware レプリケーションのしくみ | Microsoft Docs"
description: "この記事では、Azure Site Recovery サービスを使用してオンプレミスの VMware VM と物理サーバーを Azure にレプリケートするときに使用されるコンポーネントとアーキテクチャの概要を説明します"
services: site-recovery
documentationcenter: 
author: rayne-wiselman
manager: carmonm
editor: 
ms.assetid: c413efcd-d750-4b22-b34b-15bcaa03934a
ms.service: site-recovery
ms.workload: storage-backup-recovery
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: get-started-article
ms.date: 05/29/2017
ms.author: raynew
ROBOTS: NOINDEX, NOFOLLOW
redirect_url: vmware-walkthrough-architecture
ms.translationtype: Human Translation
ms.sourcegitcommit: db18dd24a1d10a836d07c3ab1925a8e59371051f
ms.openlocfilehash: 1ebf6e562c99c5113cc33bc8cb87416c663029c6
ms.contentlocale: ja-jp
ms.lasthandoff: 06/15/2017

---



# <a name="how-does-vmware-replication-to-azure-work-in-site-recovery"></a>Site Recovery での Azure への VMware レプリケーションのしくみ

この記事では、[Azure Site Recovery](site-recovery-overview.md) サービスを使用したオンプレミスの VMware 仮想マシンと Windows/Linux 物理サーバーの Azure へのレプリケートに関係するコンポーネントとプロセスについて説明します。

オンプレミスの物理サーバーを Azure にレプリケートするときも、VMware VM レプリケーションと同じコンポーネントとプロセスが使用されますが、次の違いがあります。

- 構成サーバーとして、VMware VM ではなく物理サーバーを使用することができます。
- フェールバック用にオンプレミスの VMware インフラストラクチャが必要になります。 物理マシンにはフェールバックできません。

この記事の末尾にあるコメント欄でご意見をお送りください。または [Azure Recovery Services フォーラム](https://social.msdn.microsoft.com/forums/azure/home?forum=hypervrecovmgr) に質問を投稿してください。


## <a name="architectural-components"></a>アーキテクチャ コンポーネント

VMware VM と物理サーバーの Azure へのレプリケートに関係するコンポーネントは多数あります。

**コンポーネント** | **場所** | **詳細**
--- | --- | ---
**Azure** | Azure では、Azure アカウント、Azure ストレージ アカウント、および Azure ネットワークが必要です。 | レプリケートされたデータはストレージ アカウントに格納され、オンプレミス サイトからのフェールオーバーが発生したときにそのレプリケートされたデータで Azure VM が作成されます。 Azure VM は、作成時に Azure 仮想ネットワークに接続します。
**構成サーバー** | デプロイするために必要なすべてのオンプレミス コンポーネントを実行する 1 台のオンプレミスの管理サーバー (VMware VM)。構成サーバー、プロセス サーバー、マスター ターゲット サーバーが含まれます。 | 構成サーバー コンポーネントは、オンプレミスと Azure の間の通信を調整し、データのレプリケーションを管理します。
 **プロセス サーバー**:  | 構成サーバーに既定でインストールされます。 | レプリケーション ゲートウェイとして機能します。 レプリケーション データを受信し、そのデータをキャッシュ、圧縮、暗号化によって最適化して、Azure Storage に送信します。<br/><br/> また、プロセス サーバーは保護されたマシンへのモビリティ サービスのプッシュ インストールを処理し、VMware VM の自動検出を実行します。<br/><br/> デプロイメントの拡大に合わせて、増大するレプリケーション トラフィックの処理を実行する独立した専用プロセス サーバーを追加できます。
 **マスター ターゲット サーバー** | 既定で、オンプレミスの構成サーバーにインストールされます。 | Azure からのフェールバック中にレプリケーション データを処理します。<br/><br/> フェールバックのトラフィックの量が多い場合は、フェールバック用に別のマスター ターゲット サーバーをデプロイすることができます。
**VMware サーバー** | VMware VM は vSphere ESXi サーバー上でホストされます。vCenter サーバーでホストを管理することをおすすめします。 | VMware サーバーは Recovery Services コンテナーに追加します。
**レプリケートされたマシン** | レプリケートする各 VMware VM にモビリティ サービスがインストールされます。 このサービスは、各マシンに手動でインストールすることも、プロセス サーバーからのプッシュ インストールでインストールすることもできます。

これらのコンポーネントをデプロイするための前提条件と要件については、[サポート マトリックス](site-recovery-support-matrix-to-azure.md)を参照してください。

**図 1: VMware から Azure へのコンポーネント**

![コンポーネント](./media/site-recovery-components/arch-enhanced.png)

## <a name="replication-process"></a>レプリケーション プロセス

1. Azure コンポーネント、Recovery Services コンテナーなどのデプロイをセットアップします。 このコンテナーでは、レプリケーションのソースとターゲットの指定、構成サーバーのセットアップ、VMware サーバーの追加、レプリケーション ポリシーの作成、モビリティ サービスのデプロイ、レプリケーションの有効化、テスト フェールオーバーの実行を行います。
2.  マシンがレプリケーション ポリシーに従ってレプリケートを開始し、データの初回コピーが Azure Storage にレプリケートされます。
4. Azure への差分変更のレプリケーションは、初期レプリケーションの終了後に開始されます。 マシンの追跡された変更は .hrl ファイルに保持されます。
    - レプリケートするマシンは、レプリケーション管理のために、受信ポート HTTPS 443 で構成サーバーと通信します。
    - レプリケートするマシンは、受信ポート HTTPS 9443 でレプリケーション データをプロセス サーバーに送信します (構成可能)。
    - 構成サーバーは、送信ポート HTTPS 443 経由で Azure によるレプリケーション管理を調整します。
    - プロセス サーバーは、ソース マシンからデータを受信し、そのデータを最適化して暗号化し、送信ポート 443 を介して Azure Storage に送信します。
    - マルチ VM 整合性を有効にすると、レプリケーション グループ内のマシンは、ポート 20004 を介して相互に通信します。 フェールオーバー時にクラッシュ整合性復旧ポイントとアプリ整合性復旧ポイントを共有するレプリケーション グループに複数のマシンをグループ化する場合にマルチ VM が使用されます。 これは、これらのマシンが同じワークロードを実行していて、一貫性を持たせる必要がある場合に役立ちます。
5. トラフィックは、インターネット経由で Azure Storage のパブリック エンドポイントにレプリケートされます。 また、Azure ExpressRoute の[パブリック ピアリング](../expressroute/expressroute-circuit-peerings.md#public-peering)を使用することもできます。 オンプレミス サイトから Azure へのサイト間 VPN を介したトラフィックのレプリケートはサポートされていません。

**図 2: VMware から Azure へのレプリケーション**

![拡張](./media/site-recovery-components/v2a-architecture-henry.png)

## <a name="failover-and-failback-process"></a>フェールオーバーとフェールバックのプロセス

1. テスト フェールオーバーが想定どおりに動作することを確認した後、必要に応じて、Azure への計画されていないフェールオーバーを実行することができます。 計画フェールオーバーはサポートされていません。
2. 単一のマシンをフェールオーバーするか、複数の VM をフェールオーバーするための[復旧計画](site-recovery-create-recovery-plans.md)を作成することができます。
3. フェールオーバーを実行すると、レプリカ VM が Azure に作成されます。 フェールオーバーをコミットして、レプリカの Azure VM からワークロードへのアクセスを開始します。
4. プライマリ オンプレミス サイトが再度使用可能になると、フェールバックできます。 フェールバック インフラストラクチャをセットアップし、セカンダリ サイトからプライマリへのマシンのレプリケートを開始して、セカンダリ サイトから計画外フェールオーバーを実行します。 このフェールオーバーをコミットすると、データがオンプレミスに戻るため、Azure へのレプリケーションをもう一度有効にする必要があります。 [詳細情報](site-recovery-failback-azure-to-vmware.md)

フェールバックの要件がいくつかあります。


- **Azure 上の一時的なプロセス サーバー**: フェールオーバー後に Azure からフェールバックする場合は、Azure からのレプリケーションを処理するために、プロセス サーバーとして構成された Azure VM をセットアップする必要があります。 この VM は、フェールバックの完了後に削除できます。
- **VPN 接続**: フェールバックするには、Azure ネットワークからオンプレミス サイトへの VPN 接続 (または Azure ExpressRoute) をセットアップする必要があります。
- **独立したオンプレミス マスター ターゲット サーバー**: オンプレミスのマスター ターゲット サーバーによって、フェールバックが処理されます。 マスター ターゲット サーバーは、既定では管理サーバーにインストールされますが、大量のトラフィックをフェールバックする場合は、この目的用にオンプレミスのマスター ターゲット サーバーをセットアップする必要があります。
- **フェールバック ポリシー**: オンプレミスにもう一度レプリケートするには、フェールバック ポリシーが必要です。 これは、レプリケーション ポリシーを作成したときに自動的に作成されます。
- **VMware インフラストラクチャ**: オンプレミスの VMware VM にフェールバックする必要があります。 これは、オンプレミスの物理サーバーを Azure にレプリケートする場合でも、オンプレミスの VMware インフラストラクチャを作成する必要があることを意味します。

**図 3: VMware/物理マシンのフェールバック**

![フェールバック](./media/site-recovery-components/enhanced-failback.png)


## <a name="next-steps"></a>次のステップ

[サポート マトリックス](site-recovery-support-matrix-to-azure.md)を見直す

