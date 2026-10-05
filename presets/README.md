# 入荷検査プリセット

各機種の JSON は、接続設定、読取リクエスト、検査項目をまとめた `schemaVersion: 2` の定義ファイルです。UG65 / UG56 は各 70 項目、UR32S / UR32Sリモルータは各 1 項目を定義しています。2 スペースのインデントで整形し、1 項目ずつ編集できます。

`index.json` の `files` に機種別ファイル名を列挙します。ファイル名は `id.json` と一致させてください。公開配信元は `sunds-support-ryu/ug-kitting-downloads` の `main/presets/` です。アプリ起動時に GitHub から読み込み、取得できない場合は同梱版を使用します。「JSON を読み込む」で取り込んだファイルは、本機の localStorage に保存され、同一 ID の GitHub 版より優先されます。

## 接続設定

`id` は内部識別子、`label` は選択画面の機種名です。`model` と `firmwareVersion` は期待値として参照されます。`firmwareMatch` は `exact`（完全一致）または `prefix`（前方一致）です。

`inspectionType` は `gateway`（CGI / Network Server API 検査）または `firmware-only`（ルーターのログインと Status Summary 取得）です。`protocol` は `http` / `https`、ルーターの `authMode` は `session-header` / `cookie` です。`defaultIp`、`username`、`password` は入力欄の初期値です。本機に保存済みの IP と、空欄でない認証情報が優先されます。

`username` と `password` のキーは用意しています。公開 GitHub に設定した値は誰でも閲覧できます。ローカルで入力・保存した値は GitHub に送信されません。

## 読取リクエスト

`requests.cgi` は `key`、`id`、`core`、`base` を指定します。実行時は必ず `function: "get"` の POST になります。`requests.api` は `key` と `/api/` で始まる `path` を指定し、GET で取得します。

`key` は検査項目の `source` と取値パスに使います。CGI の `statusSummary` は報告書の Model / S/N 取得にも使うため、ゲートウェイの定義から削除しないでください。証明書を検査する項目が有効な場合は、別途証明書を export して期限を解析します。

`firmware-only` は `statusSummary` の CGI 読取リクエストを一つ指定し、API の一覧は空にします。ログイン処理の詳細は `authMode` に応じて実行されます。

## 検査項目

`checks` の配列順が画面と報告書の表示順です。項目を追加・削除すると表示件数も変わります。`enabled: false` で一時的に無効化できます。

```json
{
  "enabled": true,
  "id": "net-mtu",
  "group": "Network",
  "label": "MTU",
  "source": "cgi.wan",
  "expected": 1500,
  "actual": {
    "path": "cgi.wan.mtu"
  },
  "rule": {
    "type": "equal"
  }
}
```

`id` はファイル内で重複できません。`group` はカテゴリ、`label` は表示名です。`source` から CGI / API の取得方式と通信エラーを判定します。複数取得元を使う項目は `source` を配列で指定してください。

`expected` に直接期待値を記載します。Model / Firmware Version は代わりに `expectedFrom: "model"` / `"firmwareVersion"` を指定し、接続設定の値を参照します。`expected` と `expectedFrom` は同時に指定できません。

`actual.path` は取得値の場所です。`cgi.<key>` は CGI の `result[0].get[0].value`、`cgiResponse.<key>` は CGI レスポンス全体、`api.<key>` は `status` と `data`、`login` は機器情報、`certificate` は解析した証明書情報を表します。

`actual` は次の書式を使えます。

| 指定 | 用途 |
| --- | --- |
| `path` | `cgi.wan.static.ip_address` などの取値パス |
| `literal` | 固定値 |
| `transform: "boolean"` | true / 1 を「有効」、false / 0 を「無効」に変換 |
| `map` | 取得値の対応表。例: `{"4": "AS923-1"}` |
| `template` と `values` | `{0}`、`{1}` に子の取値指定を挿入して表示 |
| `when` | 子の取値指定が真のときのみ値を生成 |
| `find` と `field` | 一覧から指定のフィールド値に一致する要素を検索し、その要素の値を取得 |
| `classes` | `when` が真になる `label` をスペースで連結 |
| `join` | 配列の区切り文字 |
| `add` | 数値の加算 |
| `fallback` | 値が未取得または null の場合の代替値 |
| `emptyValue` | 未取得、null、空文字、空配列の場合の代替値 |

`rule.type` は次の判定方式を使えます。

| type | 判定 |
| --- | --- |
| `equal` | 期待値と一致 |
| `prefix` / `suffix` | 期待値で始まる / 終わる文字列 |
| `firmware` | `firmwareMatch` の指定に従うファームウェアバージョン判定 |
| `hex` | `length` で指定した桁数の大文字 16 進数 |
| `reference` | `value` の取値指定と一致。Gateway EUI の相互確認等 |
| `all` | `conditions` の全条件が成立。各条件で `actual`、`type`、`value` を指定 |
| `certificate` | 証明書残存日数。`warningDays` 以下を WARNING、`alertDays` 以下または失効済みを ALERT |

既存の証明書閾値は `warningDays: 1096`、`alertDays: 366` です。SN は報告書の機器情報に記載し、独立した検査項目にはしていません。

対応済みの取値・判定方式を使う変更は JSON だけで反映されます。新しいログイン方式や判定方式を追加する場合はアプリの更新が必要です。
