+++
title = "データの変換とルーティング"
weight = 40
pre = "<b>d. </b>"
+++

## 本章の概要
この章の終わりには、クラウドソリューションで次のようなことができるようになります

* デバイスから受信した未加工の音声データを、クラウド内に保存されたアプリケーションロジックで、部屋の在室状況に関するアクショナブルインテリジェンスに変換する
* 部屋の在室状況ステータスを Device Shadow からデバイスへ同期する
* AWS IoT Events にリソースを設定し、デバイスからのメッセージを受信する
* サーモスタットデバイスが出したメッセージを、AWS IoT Events に転送し、さらに処理する

## クラウドソリューションの設定方法
### 音声から部屋の在室状況を取得する
クラウドソリューションの最初の部分は、部屋が使用者によって利用されているかどうかを把握するインテリジェンスを追加することです。部屋の利用状況を推測するため、スマートサーモスタットデバイスが通知する音声レベルを使用します。サンプリングされた音声レベルが所定のしきい値を超過した場合、部屋は利用中とマークされます。しきい値未満の場合、利用中ではないとマークされます。このステータスを Device Shadow に保存し、それを使って変更をデバイスに同期できます。

なぜ、部屋の在室状況の分類に音声レベルと簡易しきい値バンドを使用するのか？ なぜ、モーションセンサーを使用しないのか。このケースでは、マイクが使用できるセンサーです。IoT ソリューションを設計する際には、常に最良の入力データを確保するための予算があるわけではありません。このアプローチでは、コスト削減とユースケースの達成をうまく両立できます。著者はこの方法がすべてのユースケースにおいて機能するものではないことを認識しています。例えば、耳が不自由で手話を使っている人たちのチームミーティングや、黙って書類に目を通すことから始まるチームミーティングでは機能しません。

AWS IoT Core の 2 つの機能を使って、部屋の在室状況の把握という、このユースケースを達成します。2 つの機能とは、トピックルールと Device Shadow です。トピックルールでは、トピックフィルターに届くメッセージの動作を定義できます。例えば、ペイロードをライブで変換し、ペイロードを新規送信先にルーティングするといったことがあります。Device Shadow は、通知された求めるデバイスの状態を同期する半構造化 JSON ドキュメントです。サーモスタットの Device Shadow に加えられた変更は、新規ペイロードとしてデバイスに送信されます。前回の章で、デバイスがシャドウの更新を受信するようすでに設定し、テストを行っています。

トピックルールと Device Shadow を組み合わせて、クラウドに保存されているアプリケーションロジックに基づき、機能的に Device Shadow に更新を加えられます。これにより、アプリケーションロジックを速やかに更新でき、新規コードをデバイスにプッシュする必要がなくなります。

この章の最初のマイルストーンは、IoT Core のトピックルールを作成することです。トピックルールはスマートサーモスタットがパブリッシュしたメッセージを受信し、サンプリングされた音声レベルを調査し、変化に応じて Device Shadow の部屋の在室状況ステータスを更新します。トピックルールは SQL クエリの条件ロジックを使って、新規ペイロードと IoT Core 再発行アクションを作成し、新規ペイロードを Device Shadow に送信します。

1. [AWS IoT Core マネジメントコンソール](https://us-west-2.console.aws.amazon.com/iot/home?region=us-west-2#/)に移動し、[Act]、[ルール]、[作成] の順に選択します。
2. ルールに名前を付け、説明を入力します。この資料の今後のステップでは、このルール名を`thermostatRule`とします。
3. 以下のクエリを使って、**<<CLIENT_ID>>** をデバイスのクライアント ID/シリアルナンバーに置き換えます。

```SQL
SELECT CASE state.reported.sound > 10 WHEN true THEN true ELSE false END AS state.desired.roomOccupancy FROM '$aws/things/<<CLIENT_ID>>/shadow/update/accepted' WHERE state.reported.sound <> Null
```

4. [アクションの追加] を選択します
5. [AWS IoTのトピックにメッセージを再パブリッシュする]、[アクションの設定] の順に選択します
6. [トピック] には、`$$aws/things/<<CLIENT_ID>>/shadow/update` を使用します。**<<CLIENT_ID>>** がデバイスのクライアント ID/シリアルナンバーに置き換えられていることを確認します
7. [Choose or create a role to grant AWS IoT access to perform this action (このアクションの実行のために AWS IoT アクセスを付与するロールを選択または作成)] で、[ロールの作成] を選択すると、ポップアップが開き新しい IAM ロールに名前をつけます。その後、[ロールの作成] をクリックします
8. [アクションの追加] を選択して、アクションの設定を終了し、ルール作成フォームに戻ります。

このルールを細かく見ていき、各部分について説明します。SELECT 句は CASE ステートメントを使用して、簡易しきい値バンドを設定します。デバイスにより通知された音声レベルが 10 (0～255 段階) を超える場合、その部屋を利用中とみなします。この 10 を変更して、周囲の雑音の観測レベルに基づき、ソリューションの利用中の部屋のしきい値を設定できます。

CASE ステートメントの出力は、ペイロードキーの `state.desired.roomOccupancy` に AS キーワードとともに保存されます。つまり、`{"state": {"desired": {"roomOccupancy": false}}}` のような新しいペイロードを作成し、このペイロードをアクションに送信しています。

FROM 句はこのルールが新規メッセージを受信するトピックフィルターを示しています。このケースでは、Device Shadow サービスが新しい変更を受け入れた場合にアクションを実行する必要があるため、トピックフィルター `$aws/things/<<CLIENT_ID>>/shadow/update/accepted` を使用します。`$aws/things/<<CLIENT_ID>>/shadow/update` でデバイスが出したメッセージを傍受して、Device Shadow サービスに渡る前に変更する方法はありません。ルールエンジンと Device Shadow サービスは、同時に発行されたメッセージのコピーを受信しているためです。この動作はメッセージが発行された際の同じトピックの他の 2 人の受信者の場合と同様であり、それぞれがコピーを受信します。このパターンの欠点は、デバイスが通知した各shadowについて、**roomOccupancy** を計算する2つ目のshadowの更新がパブリッシュされ、事実上shadowでのトラフィックが倍になることです。(これを最適化する方法の 1つは、新規の値が以前の値と同じであるならば、**roomOccupancy** からの新しいshadowメッセージを発行しないことです。そのようにするには WHERE 句を修正します。この役に立つ解決策は、最適化よりもシンプルさを優先しています)。

WHERE 句はこのステートメントがtrueになった場合のみ、ルールのアクションを実行するといった、条件ステートメントを定義します。こちらではそれを使って無限ループを回避しています。条件ステートメント `state.reported.sound <> Null` を含めることによって、音声値が含まれる (厳密に言うとゼロではない) シャドウの更新に対してのみ、このルールが実行されるよう設定しています。サーモスタットはシャドウの更新で `state.reported.sound` 値を通知するため、このルールはサーモスタットがメッセージをパブリッシュすると実行されます。ルールが独自のシャドウの更新をパブリッシュした場合、ルールは再度実行されません。新しいシャドウの更新ペイロードには `state.desired.roomOccupancy` キーのみ含まれ、`state.reported.sound` の値は含まれないためです。

このルールのアクションは「republish」です。つまり、ルールクエリの出力を指定されたトピックにおける新規メッセージとしてパブリッシュします。このアクションで使用する発行トピックは、`$$aws/things/<<CLIENT_ID>>/shadow/update` で、新しいメッセージを Device Shadow に送信します。これはサーモスタットが C-SDK の Device Shadow インターフェイス経由でパブリッシュするものと同じトピックです。

(補足情報) Device Shadow トピックのその他の使用では *$* 記号は 1 つであるにもかかわらず、この場合は 2 つ使用している理由 ルールエンジンアクションは、任意で置換テンプレートをサポートするためです。置換テンプレートによって、実行時に評価される式を定義できます。置換テンプレートは `${ YOUR_EXPRESSION_HERE }` 表記を使用するため、Device Shadow トピックの `$aws` プレフィックスと競合します。正しい Device Shadow トピックを再発行アクションで使用するには、最初の $ 記号を避ける必要があるため、`$$aws/things/<<CLIENT_ID>>/shadow/update` のようになります。

このルールを IoT Core に保存した後、部屋の在室状況ステータスがスマートサーモスタットをシリアルモニター(`pio run --environment core2foraws --target monitor`)で参照することで、更新されているかの確認を行うことができます。

### HVAC のコマンドを決定する準備を行う
この章の次のマイルストーンは、現在の室温と部屋の在室状況に基づき、新しい HVAC 状態 (暖房、冷房、維持など) を指示するために必要なクラウドインフラストラクチャを準備することです。メッセージを受信するよう IoT Events サービスをプロビジョニングします。その後、2 つ目の IoT Core ルールを作成し、IoT Events と連携します。これにより、IoT Core から IoT Events へのデータフローが作られます。これは HVAC の状態を指示する探知器モデルの作成に移る前に必要となります。

IoT Events には、入力および探知器モデルという 2 つのリソースタイプがあります。入力はインバウンドメッセージを探知器モデルにマッピングするための事前定義済みスキーマです。探知器モデルは有限ステートマシンで、1 つ以上の入力からのメッセージを処理し、モデルの状態を変更すべきかどうかを判断します。

次のステップに従い、IoT Events に入力リソースを作成します。

1. [IoT Events マネジメントコンソール](https://us-west-2.console.aws.amazon.com/iotevents/home?region=us-west-2) に移動します。左側のメニューを展開し、[入力]、[Create input (入力を作成)] の順に選択します。
2. 入力に `thermostat` という名前を付け、説明を記入します。このモジュールの今後のステップでは、`thermostat` という名前を使います。
3. JSON ファイルをアップロードして、スキーマを定義する必要があります。次の内容で新しいファイルをコンピュータに作成し、`input.json`のようなファイル名を付けます。

```JSON
{
  "current": {
    "state": {
      "reported": {
        "sound": 10,
        "temperature": 35,
        "roomOccupancy": false,
        "hvacStatus": "HEATING"
      }
    },
    "version": 13
  },
  "timestamp": 1606282489
}
```

4. [Upload file (ファイルをアップロード)] を選択し、作成した新規ファイルを選びます。ファイルから読み取られたスキーマのプレビューを確認する必要があります。
5. [作成] を選択し、入力の作成を終了します。

これで入力リソースが IoT Events に作成されました。IoT Core に戻り、HVAC コントロールアプリケーションの作成に使用するこのリソースに、Device Shadow の更新を転送する新しいルールを作成できます。次の章でコントロールアプリケーションを作成します。

1. AWS IoT Core マネジメントコンソールに戻り、[Act]、[ルール]、[作成] の順に選択します。
2. ルールに名前を付け、説明を入力します。
3. 次のクエリのステートメントを使って、**<<CLIENT_ID>>** をデバイスのクライアント ID/シリアルナンバーに置き換えます。

```SQL
SELECT current.state as current.state, current.version as current.version, timestamp FROM '$aws/things/<<CLIENT_ID>>/shadow/update/documents'
```

4. [アクションの追加] を選択します。
5. [Send a message to an IoT Events Input (IoT Events 入力にメッセージを送信)]、[アクションの設定] の順に選択します。
6. [Input name (入力名)] で、IoT Events コンソールの先ほどのステップで作成した入力リソース名を見つけます。`thermostat` という名前になっているはずです。
7. [ロール] で [ロールの作成] を選択し、ロールに名前を付けます。その後 [ロールの作成] を選択し、IoT Core が IoT Events にメッセージを送信する許可を与える新しい IAM ロールを確定します。
8. [アクションの追加] を選択し、新しいルールアクションを確定します。これでルール作成フォームに戻ります。
9. [Create rule (ルールの作成)] を選択し、ルールの作成を終了します。

このルールは以前のものと比べてとてもシンプルです。 ルールは、スマートサーモスタットの Device Shadow が更新されると常に、JSON ドキュメント全体を受信し、それを新しい IoT Events 入力に転送するよう設定されています。入力は、Device Shadow ドキュメントからいくつかのフィールドのみ解析するよう設定されており、不必要なものは破棄します。

## 検証ステップ
次の章に進む前に、ソリューションが想定どおりに設定されているかを検証できます

1. サーモスタットデバイスはさまざまな雑音レベルを検知するため、デバイスがルールから roomOccupancy の最新ステータスを受信していることを確認する必要があります。音楽を流して 10 秒間音を立て、10 秒間静かにするということを交互に行い、シリアルモニター(`pio run --environment core2foraws --target monitor`) で状態変化を確認します。

想定どおりに機能している場合は、[クラウドアプリケーション](/jp/smart-thermostat/cloud-application.html)に進みましょう。

---
{{% button href="https://github.com/m5stack/Core2-for-AWS-IoT-EduKit/issues" icon="fas fa-bug" %}}Report bugs{{% /button %}} {{% button href="https://github.com/aws-samples/aws-iot-edukit-tutorials/discussions" icon="far fa-question-circle" %}}Community support{{% /button %}}