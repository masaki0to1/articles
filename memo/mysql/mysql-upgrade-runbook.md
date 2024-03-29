# MySQLのメジャーアップグレード方法の比較検討

## メジャーアップグレード方法
３方式を比較

|移行方法|手順|ダウンタイム|切り戻しレベル|DBエンドポイント|
|:---|:--|:--|:--|:--|
|置換方式|シンプル|長い|スナップショット|同じ|
|ダンプ＆リストア方式|普通|普通|旧環境に切り戻し|変わる|
|レプリケーション方式|複雑|短い|旧環境に切り戻し|変わる|

## 考慮事項
- ダウンタイムが長くなると、ユーザービリティが大きく損なわれる
- データの完全性が失われた際の影響範囲・レベルが大きい
- アプリケーション実装側の挙動まで含めて未知数な要素が多い

## 方針
- ダウンタイムは最小にする
- スナップショットによるリストアではなく、複製による旧環境への早急な切り戻しができるようにする
- 影響範囲を最小限に留める

## 採用方式
- レプリケーション方式を採用する
    - その際、ロードバランサーまで含めたスコープで複製する
    - サブドメインを取得し、クライアントからのアクセス先を動作確認した新環境へ切り替えるような形にする


## 手順
1. ALB～DBまで含めたスコープの複製をする
    a. ALB,APサーバー,DBなど必要な構成の側を複製
    b. DBの中身について、スナップショットからDBバージョン、データまで同一のものを複製する
2. DBのメジャーアップデートを実施する
3. アプリケーションの設定をする
4. テストおよび検証をする
5. サブドメインを取得する
6. ALBのターゲットグループから、DBのエンドポイントまで、正しく疎通先の更新を行い、新環境への切り替えをする
