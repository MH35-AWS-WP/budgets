# AWS予算額テンプレート

AWSで予算額を設定し、予算額に近づいた、もしくは到達した際にアラートを発出するAWS CloudFormationテンプレートです。

## 使い方

formation.ymlをAWS CloudFormationでデプロイしてください

## パラメータ説明

* ApplicationArn - myApplicationのAWS Service CatalogのアプリケーションのARN
* ApplicationTag - myApplicationのAWS Resource Groupsのタグ
* TargetEmail - 通知先のメールアドレス
* BudgetAmount - 予算額を米ドルベースで指定。デフォルトは100で、100ドルです

## テンプレートの解説

* AppAssocは、アプリケーションとAWS CloudFormationスタックの紐づけを行います
* Budgetで、実際の予算を定義します
    * Budgetで予算を設定します
        * BudgetTypeには予算の種類を指定します。今回は額面予算ですのでCOSTを指定します
        * TimeUnitには、予算の期間単位を指定します。今回は月単位予算ですのでMONTHLYを指定します
        * 予算額の種類ごとに、残りの指定方法は変わります
            * 今回のように、固定額予算を指定する場合は、BudgetLimitを指定します。Amountで予算額を、Unitで単位を指定します。予算額をドル建てで指定するならば、UnitはUSDです。Amountはパラメータで受け取ったBudgetAmountをRef関数で取得しています
            * 12か月分の成長込みの予算額を指定する場合、PlannedBudgetLimitsを指定します。この場合、キーには月初のThe Epochからのタイムスタンプを、値にはBudgetLimitと同じ内容(AmountとUnit)を指定します。例えば2025年5月に指定するのであれば、1746057600が最初のキーになります
            * 自動調整予算を指定する場合は、AutoAdjustDataを指定します。ここに指定するキーは以下の通りです
                * AutoAdjustTypeには、自動調整の種類を指定します。過去の実績に基づく場合はHISTORICALを指定します
                * HistoricalOptionsには、どれだけの期間を対象とするかを指定します。TimeUnitごとに指定できる値の範囲は異なり、MONTHLYであれば12(つまり12か月)が上限となります。原則として1年(QUARTERLY(四半期)であれば4、ANNUALLYであれば1)が上限となります(リザーブドインスタンスのカバー率などの場合、DAILYを指定でき、この場合は60(60日)が上限となります)
        * BudgetNameには、予算名を指定します
        * CostFiltersでは、予算のカバーする範囲を指定可能です。今回は指定しません
        * CostTypesには、額面予算における、どの種類のコストをカバーするかを指定します
            * デフォルトではすべてのコストを含み、非償却・非ブレンドコストを用います
            * クレジット・割引・RI以外の購読費・繰り返しの費用・返金・サポート・税・前払い費用について含む・含まないを指定可能です
                * 税というのは、日本では消費税のことを指します
            * 今回は指定をしていません。したがって、実際に発生したすべてのコストを、非償却・非ブレンドコストで予算範囲としています
        * FilterExpressionには、予算のフィルタ式を指定します。今回は特に指定しません
        * TimePeriodには、予算の有効期限を指定します。この期限を過ぎると、予算は削除されます。今回は有効期限を定めない予算ですので、設定はありません
            * Startでは、予算の開始時期を指定します。デフォルトは予算を作成した時点の始期です。例えばMONTHLY予算であれば、当月1日午前0時となります
            * Endでは、予算の終了時期を指定します。デフォルトでは定めがありません
            * 指定はMM/DD/YY HH:MM UTCの形です
    * NotificationsWithSubscribersで通知の設定を行います。最大5個の通知設定が配列で指定可能です。各要素で以下の設定を行います
        * Notificationでは通知条件の設定を行います
            * ComparisonOperatorで比較演算子の設定を行います。通常、額面予算であればGREATER\_THANを指定します。これはコストが超過することが問題なためです。逆にRI使用率等であればLESS\_THANを指定するでしょう。これは逆に使用率が低いことがRIの無駄遣いであるためです。このほかにEQUAL\_TOも指定可能です。こちらもRIで使用することがあります。100%に到達してしまっている場合、意図していない100%であれば、RIでカバーしきれていないインスタンスが存在している可能性があるためです。今回は額面予算ですのでGREATER\_THANを指定しています
            * NotificationTypeでは、実コストと予測コストのどちらで通知するかを指定します。ACTUALであれば実コスト、FORECASTEDであれば予測コストです。今回は実コストベースなのでACTUALを指定しています
            * Thresholdでは閾値を数値で指定します。この数値の意味はThresholdTypeによって変わります。今回は閾値80と100を指定しています
            * ThresholdTypeでは閾値の意味を指定します。PERCENTAGEであれば予算に対するパーセントです。ABSOLUTE\_VALUEであれば絶対値です。今回はPERCENTAGEです。つまり80%と100%の時に通知を発出します
        * Subscribersでは、通知先を配列で指定します。各通知では最低1件の通知先が必要です。また、最大で10件のメールアドレスと1件のAmazon SNSトピックを指定可能です。各要素で以下の設定を行います
            * SubscriptionTypeでは、通知先の種類を指定します。メールアドレスであればEMAIL、Amazon SNSトピックであればSNSを指定します
            * Addressでは、通知先を指定します。メールアドレスもしくはAmazon SNSトピックのARNを指定します
    * ResourceTagsではこの予算に対するタグの設定を行います。タグは最大200個まで指定可能で、タグは配列で指定します。各要素において、Keyがタグの名前、Valueがタグの値となります
        * AppManagerCFNStackKeyタグは、Application Managerにおけるスタック単位の管理タグです。値としてスタック名を指定します
        * awsApplicationタグは、myApplicationにおけるリソース管理タグです
