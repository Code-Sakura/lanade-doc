# ゲームエンジンはどれを採用するべきか

## 候補
* Unity
* UnrealEngine
* Godot Engine 
* 自作エンジン

## 選定
UnrealEngine

## 選定理由
* 外部エクスポート、インポートの自由度がUnityと比較して高い
    * FBX / glTF / USD などの業界標準フォーマットを一次データとして扱っており、外部ツール主導の制作パイプラインに組み込める設計。
    外部ツールで制作されたコンテンツを前提でインポート機構が整備されていて、そのままゲームに取り込めるように設計。

        * glTF を公式にサポートしており、シーンやアセットを読み込んだり書き出す機能がある

        * USDファイルを保持したまま操作でき、そのままエクスポート可能
    　  
        * 参考:
            * https://dev.epicgames.com/documentation/en-us/unreal-engine/working-with-content-in-unreal-engine
            
            * https://dev.epicgames.com/documentation/en-us/unreal-engine/universal-scene-description-in-unreal-engine

    * ※Unityにおいて、FBX等の3Dモデルをサポートするが、内部でプロジェクト内用の独自フォーマットとしてAssetに変換されてしまう。
    つまり、内部表現が主であり、外部エクスポート前提ではない。

        参照link:
        * https://docs.unity3d.com/2020.1/Documentation/Manual/3D-formats.html

* 初期の時点で、高いグラフィックが保証されている

* C++で定義された型や設計思想を前提に、Blueprintが構築されている
    * Blueprintとは
        ```
            ノードベース（視覚的）でロジックを構成し、C++ とシームレスに統合可能な仕組み
        ```

        参考link:
        * https://dev.epicgames.com/documentation/en-us/unreal-engine/blueprints-visual-scripting-in-unreal-engine

* Dedicated Server 前提設計
    * Unityのように“まず外部SDKを入れないと始まらない”状況ではない
    * Dedicated Serverとは？
            ```
                クライアント/サーバー方式でゲーム状態を正確に扱う仕組み
            ```

        参考LINK:
        * https://dev.epicgames.com/documentation/en-us/unreal-engine/setting-up-dedicated-servers-in-unreal-engine

* gRPC / REST / 独自プロトコルとの相性が良い
    * Unreal Engine は公式に 低レベル TCP/UDP ソケット API（FUdpSocketBuilder / FTcpSocketBuilder）を提供していて、
    C++ ネイティブで任意のデータ形式による独自プロトコルの実装が可能。
        * TCPが保証するものは、順序保証、再送制御、コネクション管理のみで、データの意味、認証、圧縮、暗号化などは保証しないため、独自プロトコルが作成できると考えられる

    公式link:
    * https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/Networking

    * 補足
        * エンジンのネットワーク層は標準的なレプリケーション機構と低レベルの通信機構が明確に分離されているため、ゲーム状態同期とは別レイヤで独自の通信設計を組み込む自由度が高い設計が可能。

            * 通信そのものと、ゲームの同期を別レイヤとして設計している
                * 例：キャラ移動はレプリケーション、経済データは独自TCPプロトコル
                * なぜこれが強いのかというと、ゲームロジックを壊さず通信変更やDedicatede Serverと外部APIの併用が可能だから  
                    * 例 : Dedicated Serverはリアルタイムなゲーム世界を担当、外部APIは非リアルタイム、業務的、永続的な処理を担当する構成
        * イメージ
        ```
        [ Client (UE) ]
            |
            | ① ゲーム同期（Replication / RPC）
            v
        [ Dedicated Server (UE) ]
            |
            | ② 独自TCP / gRPC / REST
            v
        [ 外部API群 ]
        ├ 経済・通貨
        ├ 課金 / 決済
        ├ トークン / NFT
        ├ ユーザーデータ
        ├ 規制対応 / ログ
        ```

    公式link
        * https://dev.epicgames.com/documentation/en-us/unreal-engine/gameplay-framework-in-unreal-engine

    * Unityだと・・・
        * WebGLの制約があり、TCP/UD使用不可でWebSocket限定。
        参考link
            * https://docs.unity3d.com/Manual/webgl-networking.html

        * NetworkObject を中心とした同期モデルがTransport レイヤと強く結びついいて、通信と同期を分離した設計は前提
        →併用はできるらしい

        ※UnityでもTransportなどの低レベル通信パッケージは存在するが、ネットワークがエンジンの実行モデル(Unity Transport)と結びついて、独自プロトコル設計だと、エンジン都合を考慮する必要がある。

* C++で SDKを直接組み込める
    * Unreal側と同一レイヤーだから、SDKがOS APIを直接叩ける
    (Dedicate Serverにそのまま載せることができる)
        * Unreal Engine本体、作成したゲームコード、SDKはC++ネイティブコードでこれらをUnreal Build Toolが一つのバイナリとしてビルド、リンクするから、同一レイヤーと言える。
        
        * 同じメモリ空間、スレッドモデルで余計な変換、制約、遅延が発生しない

    参考link:
        * https://dev.epicgames.com/documentation/en-us/unreal-engine/integrating-third-party-libraries-into-unreal-engine


    ※ UnityのC#は低レイヤには間接的
        * C#SDLはUnityの実行モデル内で直接組み込む設計で、マネージドランタイムに依存するから


### 懸念点
* C#と比べるとメモリなどの管理の手間が増える
    * Unreal C++は完全な生C++ではない
        * new/delete を直接書かない
            * UObject / AActor / UPROPERTY / GC のルール理解が必要
        * UnrealEngine独自のGCがあるっぽい
* エンジン外（STL・独自クラス）に出ると難易度が跳ね上がる
    * イメージ
    ```
        [ Unreal Engine の世界 ]
        ├ UObject
        ├ Actor / Component
        ├ Blueprint
        ├ GC（自動メモリ管理）
        ├ FName / FString / TArray
        └ エディタ・可視化・デバッグ

        ──────────── 境界線 ────────────

        [ 純C++ / STL の世界 ]
        ├ std::vector
        ├ std::string
        ├ new / delete
        ├ RAII
        ├ドメイン層を切り出す(レイヤー分けすると)
        └ エンジンは一切面倒を見ない
    ```
    * 例：STLを使ったら・・・
        * Blueprintに表示されない
        * UPROPERTY()が付けられない
        * UnrealのGCが追跡できない
        * エディタで中身が見えない

* “綺麗なレイヤー分け”は・安全・可視性・生産性を失うことがある
    * Unrealは「アーキテクチャ」より「世界（World）」が中心
        * Webの場合の思想
            ```
                UI層
                ↓
                UseCase / Service層
                ↓
                Domain層（純粋ロジック）
                ↓
                Infra層

            ```
        * Unrealの場合の思想
            ```
                World
                    └ Actor
                        └ Component
                            └ UObject
                                └ Engineが全部管理

            ```
        * もしドメイン層を切り出してしまうと
            * エンジンが保証してくれない
            * 分けた分だけコードが増える(下記はアンチパターン)
                ```
                    Actor
                    ↓
                    UObject
                    ↓
                    ラッパークラス
                    ↓
                    純C++クラス
                    ↓
                    std::vector / std::string
                ```
                * 安全系、理想
                    ```
                        Actor
                        ↓
                        UObject / Component
                        ↓
                        Unrealの型だけ
                    ```
                    ↑に関して、
                    Actor / UObject / Component の内側（= エンジンに見せる領域）を、Unreal が理解できる仕組み（Reflection / GC / Serialize / Replicate / Editor）に乗せることができればいい
                    * どうして分けたいなら、状態を持たない関数だけは推奨とのこと

                    参考link
                    * https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-object-handling-in-unreal-engine
            * Unreakではデータ駆動が前提なため、もしやってしまうと、エディタから変えられない場合がある
            
            参考link
            * https://dev.epicgames.com/documentation/en-us/unreal-engine/array-containers-in-unreal-engine


* Unityと比較してビルド初速は厳しい
* ホットリロードが信用できないことがあるらしい
    * https://forums.unrealengine.com/t/how-blueprint-classes-created-from-c-classes-automatically-change-as-c-classes-change/504455より、古いDDLを掴むことがあるらしいc
* BlueprintとC++の責務分離が難しい
    * Blueprintのマージが難しい
    * git運用の難易度が高い
* エンジン内部概念（World / Actor / Component / Subsystem）が深いらしい

## その他エンジンのメリット、デメリット
### Unity
#### メリット
* C#の恩恵が受けれられる
    * ガーベジコレクション
    * and more
* 初期開発は早いと言われている
* Assetが豊富だと思われる
* 歴史が長い
* UnrealEngineと比較して、エディタが軽い
* 手軽に公開・販売できる「スマホ向けゲームアプリ」はUnityの方が作りやすい

#### デメリット
* ロジックがUnity依存
* 外部exportがUnity依存で不向き
* 長期・大規模開発では、設計を誤ると依存関係やパフォーマンスの管理コストが増大しやすい(←具体例を用いたい)
* Web3 SDKの選択肢は多いが、Unity専用実装になりがち

### Godot Engine 
#### メリット
* 完全オープンソース
* 軽量・高速
* エンジン改造が可能

#### デメリット
* 日本語のドキュメントが少ない
* 大規模3D・サンドボックス実績が少ない 
* Web3・経済シミュレーションの前例が少ない
* チーム開発・長期運用のノウハウ不足

### 自作エンジン
#### メリット
* 好き放題やれる

#### デメリット
* 工数がかかる
* エンジン開発が目的化する

### 参考URL
**【Unity vs UnrealEngine】の違いとは？2大ゲームエンジンの特徴を徹底比較！**

https://confidence-creator.jp/column/812/
