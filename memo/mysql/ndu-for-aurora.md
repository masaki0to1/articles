# Aurora MySQL の無停止アップグレードについて

### 手動レプリケーションと切り替え
#### 準備
1. アップグレード対象のクラスター内でbinlogが有効化されたライターインスタンスを作成する
    - クラスターパラメータグループの、binlog_formatパラメータの値をMIXEDに設定し適用する
    - 対象クラスター内で新規リーダーインスタンスを立ち上げる　
        - この際、フェイルオーバー優先順位をTier0にしておく (他のDBインスタンスより数字が小さくなること)
    - 新規リーダーインスタンスをライターに昇格する
    - Tips: 既存のDBインスタンスを再起動することでもパラメータの更新が可能だがダウンタイムが発生する。新規追加したインスタンスに最新パラメータが反映されることを利用して、binlogパラメータが有効化されたレプリケーションソース用ライターインスタンスを無停止で準備することができる。

2. binlogパラメータが有効になったレプリケーションソース用ライターインスタンスからスナップショットを作成する
    - リーダーインスタンスでは、レプリケーションに必要な情報が取得できないので注意

3. スナップショットから復元してDBインスタンスを作成する
    - 現状のインスタンスと全く同様の設定をして復元する
    - ここでは、まだエンジンのアップグレードはしない

4. レプリケーションの事前準備を行う
    - 復元が完了すると、ログイベントにbinlogのファイル名とポジティブが表示されるのでメモする

5. 手動でレプリケーションする

```
# マスターに接続
CALL mysql.rds_set_external_master ('%{DATABASE_HOST}', 3306, '%{DATABASE_USER_NAME}', '%{DATABASE_PASSWORD}', '%{4.でメモしたfilename}', %{4.でメモしたposition}, 0);

# レプリケーションを開始
CALL mysql.rds_start_replication;

# show slave status;
# いろいろ出てくる

# レプリケーションを中止する場合
CALL mysql.rds_stop_replication;

# レプリケーションが不要になった場合
CALL mysql.rds_reset_external_master;
```

6. スレーブ側（show slave status）で、 Slave_IO_Running と Slave_SQL_Running が共に yes になり、それ以降もずっと yes であることを確認する

7. マスター側（show master status）とスレーブ側（show slave status）で同期状態をチェックし、追いついたら同期完了
https://repost.aws/ja/knowledge-center/aurora-mysql-db-cluser-read-only-error

8. レプリケーション先のDBに対してインプレースアップグレードを実行する
    - この際、パラメータグループは Aurora 3 (MySQL8.0) 用のものを事前に用意しておき設定する

9. 無事アップグレードが完了したら、しばらく流しておきレプリケーションが止まらないことを確認する

#### 新系統への移行
- レプリケーション先でINSERTが発生するとレプリケーションが停止し、データの完全性が損なわれる
- その為、INSERTが発生しにくい時間帯でINSERTが発生する機能を止めて新系統で切り替えれば、サービスを無停止でアップグレードできる
- とはいえ、一部機能を停止する必要があるため、RDSのB/Gデプロイ機能を利用して、マネージドにレプリケーションする方がメリットが多そう
- B/Gデプロイ機能を利用する方が工数少なく、切り替え時の停止時間も書き込みの場合で１分から５分、読み込みの場合で数秒で済むため


### B/Gデプロイ機能の利用によるレプリケーションと切り替え
#### 準備
1. アップグレード対象のクラスター内でbinlogが有効化されたライターインスタンスを作成する
    - クラスターパラメータグループの、binlog_formatパラメータの値をMIXEDに設定し適用する
    - 対象クラスター内で新規リーダーインスタンスを立ち上げる　
        - この際、フェイルオーバー優先順位をTier0にしておく (他のDBインスタンスより数字が小さくなること)
    - 新規リーダーインスタンスをライターに昇格する
    - Tips: 既存のDBインスタンスを再起動することでもパラメータの更新が可能だがダウンタイムが発生する。新規追加したインスタンスに最新パラメータが反映されることを利用して、binlogパラメータが有効化されたレプリケーションソース用ライターインスタンスを無停止で準備することができる。

2. コンソールメニューから、B/Gデプロイの作成をクリックしよしなに進めて実行
    - 数十分でGreen環境が作成される
    - ほぼリアルタイムでBlue環境へのデータ更新がGreen環境へ反映される

3. 切り替えを実行する
    - データ量にもよるが、切り替え時のライターへの書き込み停止時間は１分から５分ほど（公式ドキュメントには、ほぼ１分以内に完了すると記載あり）
    - 切り替え時のリーダーへの読み込み停止時間は数秒程度