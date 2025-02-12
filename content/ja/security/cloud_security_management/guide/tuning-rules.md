---
aliases:
- /ja/security_platform/cloud_workload_security/guide/tuning-rules/
- /ja/security_platform/cloud_security_management/guide/tuning-rules/
title: CSM Threats セキュリティシグナルの微調整
---

## 概要 

Cloud Security Management Threats (CSM Threats) は、ワークロードレベルで発生する不審なアクティビティを監視しています。しかし、ユーザーの環境における特定の設定のために、良性のアクティビティが悪意のあるものとしてフラグ付けされるケースもあります。良性であると想定されるアクティビティがシグナルをトリガーしている場合、そのアクティビティのトリガーを抑制してノイズを抑制することができます。

このガイドでは、ベストプラクティスのための考察と、シグナル抑制を微調整するためのステップを説明します。

## 抑制戦略

良性パターンを抑制する前に、検出アクティビティの種類に基づいて、シグナルに共通する特性を特定します。属性の組み合わせが具体的であればあるほど、より正確な抑制が可能になります。

リスクマネジメントの観点からは、より少ない属性に基づく抑制は、実際の悪意あるアクティビティを除外してしまう可能性が高くなります。悪意のある行動を見逃すことなく、効果的な微調整を行うには、一般的な主要属性をアクティビティの種類別に分類した以下のリストを参考にするとよいでしょう。

### プロセスアクティビティ

共通キー:
- `@process.args`
- `@process.executable.name`
- `@process.group`
- `@process.args`
- `@process.envs`
- `@process.parent.comm`
- `@process.parent.args`
- `@process.parent.executable.path`
- `@process.executable.user`
- `@process.ancestors.executable.user`
- `@process.ancestors.executable.path`
- `@process.ancestors.executable.envs`

To determine if a process is legitimate, review its parent process in the process tree. The process ancestry tree traces a process back to its origin, providing context for its execution flow. This helps in understanding the sequence of events leading up to the current process.

通常、親プロセスと不要なプロセスの属性の両方に基づいて抑制すれば十分です。

組み合わせ例:
- `@process.args`
- `@process.executable.group`
- `@process.parent.executable.comm`
- `@process.parent.executable.args`
- `@process.user`

広い時間軸で抑制することにした場合、値が変わると抑制が効かなくなるので、一時的な値を引数に持つ処理の使用は避けてください。

例えば、再起動や実行時に特定のプログラムは一時ファイル (`/tmp`) を使用します。このような値に基づいて抑制を構築しても、同様のアクティビティが検出された場合には有効ではありません。

例えば、コンテナ上の特定のアクティビティからのすべてのシグナルのノイズを完全に抑制したいとします。コンテナをスピンアップするプロセスを開始するプロセスツリー内のフルコマンドを選択します。実行中、プロセスは、コンテナが存在する限り、存在するファイルにアクセスします。対象とする動作がワークロードロジックと結びついている場合、エフェメラルプロセスインスタンスに基づく抑制定義は、他のコンテナ上の同様のアクティビティを調整するのに有効でなくなります。

### ファイルアクティビティ

ワークロード、該当ファイル、ファイルにアクセスするプロセスに関する識別情報を反映した属性に基づいて、ファイルアクティビティ関連の抑制を絞り込むことができます。

共通キー:
- ワークロードタグ:
  - `kube_container_name`
  - `kube_service`
  - `host`
  - `env`
- プロセス:
  - `@process.args`
  - `@process.executable.path`
  - `@process.executable.user`
  - `@process.group`
  - `@process.args`
  - `@process.parent.comm`
  - `@process.parent.args`
  - `@process.parent.executable.path`
  - `@process.user`
- ファイル:
  - `@file.path` 
  - `@file.inode`
  - `@file.mode`

シグナルの検査中に実際の悪意のあるアクティビティを判断するには、プロセスがファイルにアクセスし、変更する際のコンテキストが予想通りであるかどうかを検証します。インフラストラクチャー全体でファイルに対する意図的な動作を抑制することを避けるために、上記の共通キーから関連するすべてのコンテキスト情報を収集する組み合わせを常に持つ必要があります。

組み合わせ例:
  - `@process.args`
  - `@process.executable.path`
  - `@process.user`
  - `@file.path`
  - `kube_service `
  - `host`
  - `kube_container_name`

### ネットワーク DNS ベースのアクティビティ

ネットワークアクティビティモニタリングは、DNS トラフィックをチェックし、サーバーのネットワークを危険にさらす可能性のある不審な行動を検出することを目的としています。特定の IP から DNS サーバーへのクエリをチェックしながら、プライベートネットワーク IP やクラウドネットワーク IP など、既知の IP アドレスからの良性アクセスに対してトリガーをかけることが可能です。

共通キー:
- プロセス:
  - `@process.args`
  - `@process.executable.group`
  - `@process.executable.path`
  - `@process.parent.executable.comm`
  - `@process.parent.executable.args`
  - `@process.user`
- ネットワーク/DNS 関連:
  - `@dns.question.name`
  - `@network.destination.ip/port`
  - `@network.ip/port`

ローカルアプリケーションが DNS 名を解決するために接続を行う場合、最初にチェックしたい特性は、DNS クエリと同様にルックアップを開始した IP のリストです。

組み合わせ例:
  - `@network.ip/port`
  - `@network.destination.ip/port`
  - `@dns.question.*`

### カーネルアクティビティ

カーネル関連のシグナルでは、ノイズは通常、ワークロードのロジックや特定のカーネルバージョンに関連する脆弱性から発生します。何を抑制するかを決定する前に、以下の属性を考慮してください。

共通キー:
- プロセス
  - `@process.args`
  - `@process.executable.group`
  - `@process.executable.path`
  - `@process.parent.executable.comm`
  - `@process.parent.executable.args`
  - `@process.user`
- ファイル
  - `@file.path `
  - `@file.inode`
  - `@file.mode`

このタイプのアクティビティに対する組み合わせの定義は、ファイルまたはプロセスのアクティビティに似ていますが、攻撃に使用されるシステムコールに関連するいくつかの特異性が追加されています。

例えば、Dirty Pipe の悪用は、特権昇格の脆弱性です。この攻撃を利用してローカルユーザーがシステム上で特権を拡大した場合、重大な事態になるため、ルートユーザーが期待するプロセスを実行することで発生するノイズを抑制することは理にかなっています。
- `@process.executable.user`
- `@process.executable.uid`

Additionally you might notice that signals are created even when some of your machines are running patched kernel versions (for example, Linux versions 5.16.11, 5.15.25, and 5.10 that are patched for Dirty Pipe vulnerability). In this case, add a workload level tag such as `host`, `kube_container_name`, or `kube_service` to the combination. However, when you use a workload level attribute or tag, be aware that it applies to a wide range of candidates which decreases your detection surface and coverage. To prevent that from happening, always combine a workload level tag with process or file based attributes to define a more granular suppression criteria.

## シグナルから抑制を加える

CSM Threats の検出ルールから報告された潜在的な脅威を調査する過程で、環境に特有の既知の良性の動作についてアラートを発するシグナルに遭遇することがあります。

Java プロセスユーティリティの悪用について考えてみましょう。攻撃者は、Java プロセスを実行するアプリケーションコードの脆弱性を意図的に狙います。この種の攻撃は、独自の Java シェルユーティリティを生成することにより、アプリケーションへの永続的なアクセスを伴います。

場合によっては、CSM Threats のルールは、例えば、セキュリティチームがアプリケーションの堅牢性を評価するためにペンテストセッションを実行しているような、予想されるアクティビティを検出することもあります。この場合、報告されたアラートの精度を評価し、ノイズを抑制することができます。

シグナルの詳細サイドパネルを開き、タブからタブに移動して、コマンドライン引数や環境変数キーなどの主要なプロセスメタデータを含むコンテキストを取得します。コンテナ化されたワークロードの場合、関連するイメージ、ポッド、Kubernetes クラスターなどの情報が含まれます。

{{< img src="/security/cws/guide/cws-tuning-rules.png" alt="シグナルに関連するイベント、ログ、その他のデータを表示するシグナルのサイドパネルです。" width="75%">}}

抑制条件を定義するには、任意の属性値をクリックし、**Never trigger signals for** を選択します。

この例では、これらの環境変数の使用に先立って、プロセスの祖先ツリー内で特権をエスカレートさせるアクションが実際に行われたかどうかを評価します。タグは、インフラストラクチャー内のどこでアクションが発生したかを示し、その重大度を低減するのに役立ちます。これらの情報があれば、これらの環境変数を継承しているプロセスに対して、ルールを調整することを決定できます。

ルールの調整を行う場合、シグナルの特定の属性を組み合わせることで、抑制の精度を向上させることができます。通常、抑制効果を高める以下の共通キーを使用するのが最適です。

- `@process.parent.comm`: シグナルを担当したプロセスが呼び出されたときのコンテキスト。このキーは、その実行が予期されたものであるかどうかを評価するのに役立ちます。通常、親プロセスはその実行をコンテキスト化するので、類似の良性動作を調整する良い候補となります。
- `@process.parent.path`: 同様に、親プロセスの対応するバイナリパスを追加すると、その場所を指定することで抑制が補完されます。
- `host`: 当該ホストが脆弱な環境、例えばステージング環境で動作していない場合、そこからイベントが発生するたびにシグナルがトリガーされるのを抑制することができます。
- `container.id`: ワークロードに関連する属性が混在している場合、抑制はより効率的になります。あるコンテナが良質のアクティビティ専用であることが分かっている場合、そのコンテナ名や ID を追加することでノイズを大幅に減らすことができます。
- `@user.id`: あるユーザーを既知のメンバーとして識別した場合、そのユーザーに関連するアクティビティを抑制することができます。

さらなる粒度のために、以下の属性は実行チェーンを再構築する際に過去のプロセスに関する情報を提供します。これらはプレフィックス `@process.ancestors.*` の下で見つけることができます。
- `file.name`
- `args`
- `file.path`

## ルールエディターから抑制を加える

シグナルは、セキュリティアラート内の関連するコンテキストを表面化させます。イベントデータは抑制フィルターに活用できますが、検出ルールが構築される観測可能性データはより良い調整候補を提供する可能性があります。

CSM Threats では、収集したカーネルイベントからランタイム Agent のログが生成されます。シグナルサイドパネルからコンテキストスイッチなしでログをプレビューすることができます。

1. 選択したシグナルの詳細サイドパネルで、[Events] タブをクリックします。
2. **View in Log Explorer** をクリックして、ログ管理に移動し、このシグナルを発生させるログの完全なリストを表示します。
   ログは多数存在するため、シグナルサイドパネルでは、これらのログとその共有属性を JSON 構造にまとめます。
3. Go back to the Events tab and scroll to the end of the panel. Expand the JSON dropdown to access all log attributes contained in runtime Agent events.
4. シグナルを抑制するキーと値のペアを、`@process.args`、`@process.group`、`@process.ancestors.comm`、または `@process.ancestors.args` などの共通のキーで特定することができるようになります。
5. ルールエディターでルールを開き、**Exclude benign activity with suppression queries** (抑制クエリを使用した良性アクティビティを除外する) で 役に立つと特定したキーと値のペアのリストを追加します。

例えば、`Java process spawned shell/utility` (Java プロセスが生成したシェル/ユーティリティ) というルールがあり、次のような属性の組み合わせで抑制したいとします。
- `@process.args:+x`
- `@process.executable.group:exec`
- `@process.ancestors.executable.comm:root`
- `@process.ancestors.executable.args:init`

**This rule will not generate signal if there is match** (このルールは、一致がある場合、シグナルを発生させません) にこれらのキー値を入力し、不要なシグナルを抑制します。

一方、正しい属性セットを識別して特定の条件でシグナルを発生させたい場合は、**Only generate a signal if there is a match** (一致がある場合、シグナルのみを発生させる) の組み合わせを指定します。