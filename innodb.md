* [トランザクション分離レベル](#トランザクション分離レベル)
* [インデックスの構造](#インデックスの構造)
* [ロックの種類](#ロックの種類)
* [行ロックについて](#行ロックについて)
* [シャドーロックについて](#シャドーロックについて)
* [あるあるロック問題](#あるあるロック問題)
* [パーティション下のロックの挙動](#パーティション下のロックの挙動)

##### サブ資料
[外部キー制約に伴うロックの小話](http://www.slideshare.net/ichirin2501/ss-44642631)

いちりんメモです。  
掲載されている内容が正しいとは限りません、真実は自らの手で摑み取ってください。  
ロックの挙動についてはmysql-5.5.27,**REPEATABLE READ** で調べたものです。  
READ COMMITEDは気が向いたら書きます。  


## トランザクション分離レベル
ACID特性のうちIsolationに関する概念。以下はmysql特有のものではなく、ANSI/ISO SQL標準で定められている。  
InnoDBはいずれのトランザクション分離レベルもサポートしている。

* READ UNCOMMITTED
* READ COMMITTED
* REPEATABLE READ
* SERIALIZABLE


ACID特性とは以下のことである。

1. Atomicity: タスクが全て実行されるか、あるいは全く実行されないことを保証する性質
1. Consistency: 整合性を満たすことを保証する性質
1. Isolation: 操作の過程が他の操作から隠蔽される
1. Durability: 完了通知をユーザが受けた時点でその操作は永続的となり結果が失われないこと

それぞれの分離レベルはトランザクション処理の影響度合いが異なる。  
ANSI/ISO SQLでは以下のように定義されている。

| 分離レベル | ダーティリード | ファジーリード | ファントムリード | 
|:----------:|:-----------:|:------------:|:----:|
| READ UNCOMMITTED | あり | あり | あり |
| READ COMMITTED | なし | あり | あり |
| REPEATABLE READ | なし | なし | **あり** |
| SERIALIZABLE | なし | なし | なし | なし |

あくまで仕様（）    
各ストレージエンジンで実装・動作が異なる。  
**InnoDBのREPEATABLE READはファントムリード現象も防ぐ実装** になっている。  
表には載せてないがロストアップデートはSERIALIZABLE以外全てに起こりうる。  

* ダーティリード: 未コミットのトランザクションの更新を別トランザクションが読み取る現象
* ファジーリード: トランザクション内で一度読み取ったデータを再度読み取るときに、コミット済みの別トランザクション(更新or削除)によって結果が変わる現象
* ファントムリード: トランザクション内で一度読み取ったデータを再度読み取るときに、コミット済みの別トランザクション(挿入)によって結果が変わる現象
* ロストアップデート: 後続トランザクションの更新で先行していたトランザクションの更新内容を失う現象



## インデックスの構造

* [知って得するInnoDBセカンダリインデックス活用術！ - 漢のコンピュータ道](http://nippondanji.blogspot.jp/2010/10/innodb.html)
* [INDEX FULL SCANを狙う - MySQL Casual Advent Calendar 2011 - SH2の日記](http://d.hatena.ne.jp/sh2/20111217)

大変参考になります。この辺りを読んでおけば十分でしょう。  
primary-keyがクラスタインデックス、それ以外のindexはセカンダリインデックスで押さえておけばよいです。


## ロックの種類
テーブルロックと行ロック！  
テーブルロックについては詳しく調べてないです（）   

それから基本は以下の2つのモードです

* 共有ロック(S-Lock)  
SELECT ~ LOCK IN SHARE MODE, INSERT失敗時(DUPLICATE)
* 排他的ロック(X-Lock)  
INSERT文, UPDATE文, DELETE文, SELECT ~ FOR UPDATE;

ろっくまとりーっくす

|   | X | S |
|:-:|:-:|:-:|
| X | Conflict | Conflict |
| S | Conflict | Compatible |

共有ロック同士はブロックが発生せずにレコードを読み取ることができる。  
本来なら他にもロックの種類があるが割愛。  
ロックを獲得してレコードを読むとき、必ず最新のデータを読み取る。  
InnoDBにはMVCC(Multi Version Concurrency Control)システムが実装されており、  
この機能でダーティリード現象とファジーリード現象を防いでいる。  
これを利用してREPEATABLE-READでは、  
トランザクションを開始して最初のクエリを発行したタイミングのデータベースのスナップショットを取り、  
以降の読み取りにおいてはそのときの状態を返すが、ロックを獲得する読み取りにおいてはそうではないことに注意  
その辺りの話については、以下の記事の「Locking Read - 問題点：100% 一貫性読み取りではない」の項目でも触れられている。  
* [InnoDBのREPEATABLE READにおけるLocking Readについての注意点](http://nippondanji.blogspot.jp/2013/12/innodbrepeatable-readlocking-read.html)

```sql
create table t1 (id int auto_increment, number int, primary key(id));
mysql> select * from t1;
+----+--------+
| id | number |
+----+--------+
|  1 |      1 |
|  2 |      5 |
|  3 |      5 |
+----+--------+

TA> begin;
TB> begin;
TA> select * from t1 where id = 2;
+----+--------+
| id | number |
+----+--------+
|  2 |      5 |
+----+--------+
TB> update t1 set number = 10 where id = 2;
TB> commit;
TA> select * from t1 where id = 2 for update;
+----+--------+
| id | number |
+----+--------+
|  2 |     10 |
+----+--------+
TA> select * from t1 where id = 2;
+----+--------+
| id | number |
+----+--------+
|  2 |      5 |
+----+--------+
```

### ALTER文
運営していると避けられないALTER文  

* [ALTER TABLEを上手に使いこなそう。 - 漢のコンピュータ道](http://nippondanji.blogspot.jp/2009/05/alter-table.html)
* [開発スピードアクセル全開ぶっちぎり！日本よ、これがMySQL 5.6だッ！！ - 漢のコンピュータ道](http://nippondanji.blogspot.jp/2012/10/mysql-56.html)

>MySQL 5.6ではALTER TABLE...ADD INDEX/DROP INDEX中であっても参照・更新共に実行可能になったのである。

最高ですね！  
MySQL5.6から変更内容によってはブロックせずに済むらしい（そのうちまとめます）   
MySQL5.5以下ならこんな感じ、たぶん。  

* テーブルを共有ロック
* テーブルを全コピーするので一時的に容量が2倍
* ALTER文の回数だけテーブル全コピー、同テーブルへの変更点は1行にまとめる
* ALTERは暗黙のコミットを引き起こすステートメントの一つ

手動でALTER文を含めた他のSQL作業が入るときに、ALTER文は暗黙的にコミットされることに気をつける。  
 
```text
> begin;
> update t1 set number = 10 where id = 20;
> alter table t2 ~; # この時点でalter実行前に暗黙commitが走るので t1に対するupdateもcommitされる
> rollback; # alterの内容は勿論、t1へのupdateもrollbackされない
```


## 行ロックについて

* レコードロック : インデックスレコードのロック
* ギャップロック : インデックスレコード間にあるギャップのロック、先頭のインデックスレコードの前や末尾のインデックスレコードのあとにあるギャップのロック、のいずれか
* ネクストキーロック : インデックスレコードに対するレコードロックと、そのインデックスレコードの前にあるギャップに対するギャップロックとを組み合わせ

インデックスレコードとは、クラスタインデックスとセカンダリインデックスのこと。  
'レコードロック'ではなく、 **"インデックスレコードロック"** だということが重要です。  
例えば、
```sql
create table t1 (id int auto_increment, number int, hoge int, primary key(id), index(number));
mysql> select * from t1;
+----+--------+------+
| id | number | hoge |
+----+--------+------+
|  1 |      1 |    1 |
|  2 |      5 |    2 |
|  3 |      5 |    3 |
|  4 |     10 |    4 |
|  5 |     50 |    5 |
|  6 |     51 |    6 |
|  7 |    100 |    7 |
+----+--------+------+

mysql> begin;
mysql> select * from t1 where number = 5 and hoge = 2 for update;
+----+--------+------+
| id | number | hoge |
+----+--------+------+
|  2 |      5 |    2 |
+----+--------+------+
1 rows in set (0.00 sec)
```

検索結果は1行ですが、このとき(number,hoge)=(5,3)の行もロック(+前後のギャップ）される。    

1. オプティマイザが選択したインデックスで検索されたレコードをロックする  
1. ロック獲得後、それらのうち条件にマッチする行を検索結果として返す

インデックスレコードロックなので**検索結果の行と実際にロックがかかる行は必ずしも等しいわけではない**

ギャップロックに対する認識は、とりあえず、  

1. 他TXが獲得したギャップロック範囲に対するINSERT文は全てブロックされる  
2. ギャップロックはギャップロックをブロックしない（排他的ロックだけど...）  

と覚えておけば良いでしょう。（あとで解説します）

それから、ある行をロックしようとするとき、**インデックスの種類(pkey,ukey,key)でロック範囲が異なります**  

* [InnoDBのロックの範囲とネクストキーロックの話](http://blog.kamipo.net/entry/2013/12/03/235900)
* [InnoDBで行ロック/テーブルロックになる条件を調べた #mysqlcasual Advent Calendar 2013](http://bluerabbit.hatenablog.com/entry/2013/12/07/075759)

上記の記事でだいたい触れられているが、種類によって挙動が異なる件に関して触れられていないので、  
ここではそれを補足します。  

### 非indexのとき
テーブルロックと効果が等しい  
実際は全ての行に対してロックをかけているだけであり、InnoDBはロックエスカレーションしない  
```text
create table t1 (id int auto_increment, number int, hoge int, primary key(id), index(number));
SELECT * FROM t1 WHERE hoge = 4 FOR UPDATE;  
# t1テーブルの全ての行とギャップにロックがかかります。
```  

### 通常indexのとき
対象の行と、その周辺にギャップロックがかかります
```text
create table t3 (id int auto_increment, number int, primary key(id), index(number));
mysql> select * from t3;
+----+--------+
| id | number |
+----+--------+
|  1 |      1 |
|  2 |      5 |
|  3 |      5 |
|  4 |     10 |
|  5 |     50 |
|  6 |     51 |
|  7 |    100 |
+----+--------+

# パターン1: 5の手前にinsertしようとしたとき
TA> begin;
TB> begin;
TA> select * from t3 where number = 5 for update;
TB> insert into t3 (number) values(2); # 待たされる

# パターン2: 5の後にinsertしようとしたとき
TA> begin;
TB> begin;
TA> select * from t3 where number = 5 for update;
TB> insert into t3 (number) values(6); # 待たされる

number=5 for updateで、number=5の存在する行のロックと、
number = [1,5),[5,10)
までがギャップロックされるので上記のinsertが待たされることになります


面白いことに処理順序を逆転すると成功する
TA> insert into t3 (number) values(6);
TB> select * from t3 where number = 5 for update; # 待たされない
insert->selectの処理順だと、insert後はnumber[5,10)までのgapを (LOCK_X | LOCK_GAP | LOCK_INSERT_INTENTION) の状態にする
selectの範囲は5の前後のgapに対してもLOCK_XになるがLOCK_INSERT_INTENTIONフラグが含まれてるため、
待たされることはない。最初のselect->insertの処理順だと、selectで5の前後のgapに対してもLOCK_Xになる。
insert時に挿入先のgapとロック状態が衝突してLOCK_INSERT_INTENTIONフラグがなく、かつ、LOCK_GAPがセットされてる場合は待つことになる。

gap-lock magic!


通常のindexは常にギャップロックがセットになるので、末端に注意が必要です
TA> BEGIN;
TB> BEGIN;
TA> SELECT * FROM t3 WHERE number = 100 FOR UPDATE;
TB> INSERT INTO t3 (number) VALUES(300); # 待たされる
ギャップロック範囲(number) = [51,100),[100,positive infinity)

```
複合indexのギャップロックの範囲は昇順ソートしたときのギャップの範囲になります。
```text
CREATE TABLE `benio` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `a` int(11) NOT NULL,
  `b` int(11) NOT NULL,
  `c` int(11) NOT NULL,
  PRIMARY KEY (`id`),
  KEY `a` (`a`,`b`,`c`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8

+----+----+-----+----+
| id | a  | b   | c  |
+----+----+-----+----+
|  1 |  1 |   1 | 10 |
|  2 |  1 |   1 | 20 |
|  3 |  1 | 100 | 10 |
|  4 | 10 |  50 | 15 |
|  5 | 20 |  10 | 30 |
+----+----+-----+----+

SELECT * FROM benio WHERE a = 1 AND b = 1 AND c = 20 FOR UPDATE;
このときのギャップロックの範囲は以下のようになります
(a,b,c) = [ [1,1,10],[1,1,20] ) , [ [1,1,20],[1,100,10] )

INSERT INTO benio (a,b,c) VALUES(1,1,9);    # 待たされない
INSERT INTO benio (a,b,c) VALUES(1,1,10);   # 待たされる
INSERT INTO benio (a,b,c) VALUES(1,1,21);   # 待たされる
INSERT INTO benio (a,b,c) VALUES(1,10,10);  # 待たされる
INSERT INTO benio (a,b,c) VALUES(1,100,9);  # 待たされる
INSERT INTO benio (a,b,c) VALUES(1,100,11); # 待たされない

要素を一つ減らした場合は単純に範囲が広がるだけです
SELECT * FROM benio WHERE a = 10 AND b = 50 FOR UPDATE;
ギャップロック範囲(a,b,c) = [ [1,100,10],[10,50] ) , [ [10,50], [20,10,30] )

left-most index!
```

### primary-key, unique-keyのとき
pkeyとukeyの挙動は等しく、基本的にはギャップロックが発生しない
```text
CREATE TABLE `kobeni` (
  `id` int(11) NOT NULL,
  `number` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `number` (`number`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8

+-----+--------+
| id  | number |
+-----+--------+
|   1 |      1 |
|   5 |      5 |
|  10 |     10 |
+-----+--------+

TA> BEGIN;
TB> BEGIN;
TC> BEGIN;

TA> SELECT * FROM kobeni WHERE id = 5 FOR UPDATE; # idはprimary-key  
TB> INSERT INTO kobeni (id,number) VALUES(4,4); # 待たされない
TC> INSERT INTO kobeni (id,number) VALUES(6,6); # 待たされない

# pkeyやukeyは一意に定まる。存在する行であれば対象行のみをロックし、ギャップロックは発生しない
```

**複合の場合は少し注意が必要**  
```text
player_quest_nonauto` (
  `id` bigint(20) unsigned NOT NULL,
  `player_id` bigint(20) unsigned NOT NULL,
  `quest_id` smallint(5) unsigned NOT NULL,
  PRIMARY KEY (`id`,`quest_id`),
  UNIQUE KEY `player_quest_idx` (`player_id`,`quest_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4

+----+-----------+----------+
| id | player_id | quest_id |
+----+-----------+----------+
|  1 |         1 |     1000 |
|  2 |         2 |     1000 |
|  3 |         3 |     1000 |
|  6 |         5 |     1020 |
| 27 |         8 |     1020 |
|  4 |        10 |     2000 |
| 10 |        20 |      900 |
| 11 |        20 |     1100 |
| 12 |        20 |     1200 |
| 13 |        30 |     2001 |
| 18 |        50 |     1010 |
+----+-----------+----------+

> SELECT * FROM player_quest_nonauto WHERE id = 18 AND quest_id = 1010 FOR UPDATE;
複合pkeyの要素を正しく指定していれば単一カラムのpkey(ukeyも同様)と挙動は同じになります

しかし、
> SELECT * FROM player_quest_nonauto WHERE id = 18 FOR UPDATE;
複合要素が欠けてしまうと、ロックの挙動が変化します
このとき、複合pkey(id,quest_id)なのでidのみでもインデックスは効きますが、ロックの挙動は通常indexと等しくなります
つまり、id=18の行ロックと、
(id,quest_id) = [ [13,2002] , [18,1010] ), [ [18,1011] ~ [27,1020] ) の範囲に対してギャップロックがかかります
```

### ギャップロックの罠
>ギャップロックはギャップロックをブロックしない（排他的ロックだけど...)  

良い資料  
* [MySQL InnoDBのinsertとlockの話](http://tech.voyagegroup.com/archives/8085782.html)  


例を見たほうが早いですね
```text
CREATE TABLE `t4` (
  `id` int(11) NOT NULL DEFAULT '0',
  `number` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `number` (`number`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

+----+--------+
| id | number |
+----+--------+
|  1 |      1 |
| 15 |      5 |
| 17 |      8 |
| 20 |     10 |
| 26 |     55 |
| 27 |     60 |
| 28 |     65 |
| 29 |     70 |
+----+--------+

TA> BEGIN;
TB> BEGIN;
TA> SELECT * FROM t4 WHERE id = 22 FOR UPDATE;
Empty set (0.01 sec)
TB> SELECT * FROM t4 WHERE id = 25 FOR UPDATE; # 待たされない
Empty set (0.01 sec)

空打ちになっても、そのギャップに対してロックを獲得している状態になります
この時点で、TA,TBともに 範囲id=[21,26)に対してギャップロックを獲得しています
続いて、INSERTとしようとするとデッドロックになります

TA> INSERT INTO t4 (id,number) VALUES(22,100); # 待たされる!!
TB> INSERT INTO t4 (id,number) VALUES(25,200); 
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction

TB側がデッドロックエラーとなり、TA側のINSERTが成功します
```
INSERTしようと思ってる範囲に自TXがギャップロックを獲得しても、他TXも同じ範囲でギャップロック獲得可能です。  
他TXがギャップロックした範囲に対してINSERT文はブロックされるため、このようなことになります。  
**行があったらFOR-UPDATEで排他的ロック、なかったらINSERTする**ような処理を書くときは注意が必要です。  
REPEATABLE-READでのベストな解決方法は自分の中で見出せていない。（現在の見解については後述  
空打ちロックに限らず、通常indexは常にギャップロックを伴うので思わぬところで、  
デッドロック・パフォーマンス悪化に繋がる可能性があります。  

### ネクストキーロック
InnoDB-REPEATABLEREADに存在するロックの種類`ギャップロック+レコードロック = ネクストキーロック`  
MVCCと合わせてファントムリードを防ぐ仕組みである.  
InnoDB-REPEATABLEREADの範囲ロック(between,<,etc)は広めに取られる。以下が分かり易い  
[MySQL InnoDBのネクストキーロック おさらい - SH2日記](http://d.hatena.ne.jp/sh2/20090112)


###  INSERT ON DUPLICATE KEY UPDATEの挙動に注意

```text
CREATE TABLE `t4` (
  `id` int(11) NOT NULL DEFAULT '0',
  `number` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `number` (`number`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

TA> SELECT * FROM t4;
Empty set (0.00 sec)

TA> BEGIN;
TB> BEGIN;  
TA> INSERT INTO t4 (id,number) values(1,100) ON DUPLICATE KEY UPDATE num = 300; # insert成功
TB> INSERT INTO t4 (id,number) values(1,100) ON DUPLICATE KEY UPDATE num = 300; # 待たされる
TA> COMMIT;
TB> # ロック獲得, num = 300でupdateされる
TB> COMMIT;

TA> SELECT * FROM t4;
+----+--------+
| id | number |
+----+--------+
|  1 |    300 |
+----+--------+
```
どちらのトランザクションも開始時点では行が存在しない状態だったが、  
いずれかのトランザクションでinsertされた後、続く別のトランザクションではupdate処理になる  

## シャドーロックについて
注意：自分が勝手に呼んでる現象  
待たされているクエリも部分的にロック獲得している現象のことをここでは便宜上シャドーロックとする  
```text
> SELECT * FROM t4;
+----+--------+
| id | number |
+----+--------+
|  1 |      1 |
| 15 |      5 |
| 17 |      8 |
| 20 |     10 |
| 26 |     55 |
| 27 |     60 |
| 28 |     65 |
| 29 |     70 |
| 30 |    100 |
....

> SELECT COUNT(*) FROM t4;
+----------+
| count(*) |
+----------+
|      160 |
+----------+

TA> BEGIN;
TB> BEGIN;
TC> BEGIN;
TA> SELECT * FROM t4 WHERE id = 28 FOR UPDATE;
TB> SELECT * FROM t4 WHERE id IN(26,27,28,29,30) FOR UPDATE; # 待たされる
TC> SELECT * FROM t4 WHERE id = 29 FOR UPDATE; # これは問題ない
TC> SELECT * FROM t4 WHERE id = 27 FOR UPDATE; # 待たされる
```
TBのIN句によるクエリ-ロックは、インデックスを昇順から辿ってid=26,27を順に排他ロックを獲得し、  
id=28の排他ロックを獲得しようとして待たされている（TAが既に獲得済み  
そのため、TBはid=29,30に対してロックは未獲得の状態のまま待たされることになる。  
当然、その間にid=29,30のロックを他TXから獲得することが可能であり、また、  
id=26,27に対しては他TXがロックを獲得することはできない状態である。  
シャドーロックとは、このようにクエリが待たされていても部分的にロック獲得する現象のことを言う。  
IN句に限らず、BETWEENなど複数行ロックするクエリは例外なくシャドーロックが発生する。  
ここでTAがCOMMITするとどうなるか？  
```text
TA> BEGIN;
TB> BEGIN;
TC> BEGIN;
TA> SELECT * FROM t4 WHERE id = 28 FOR UPDATE;
TB> SELECT * FROM t4 WHERE id IN(26,27,28,29,30) FOR UPDATE; # 待たされる
TC> SELECT * FROM t4 WHERE id = 29 FOR UPDATE; # これは問題ない
TC> SELECT * FROM t4 WHERE id = 27 FOR UPDATE; # 待たされる
TA> COMMIT;
TC!> ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```
TAがCOMMITしたことでTBがid=28のロックを獲得、次にid=29のロックを獲得しようとしてTCとデッドロックが発生する。  
繰り返しになるがInnoDBのロックはインデックスレコードロックであり、走査順にロックを獲得していく実装になっている。  
つまり、`order by`でインデックスの走査順が異なるとシャドーロックの挙動も少し違ってくる。  
検証1  
```text
TA> BEGIN;
TB> BEGIN;
TC> BEGIN;
TA> SELECT * FROM t4 WHERE id = 28 FOR UPDATE;
TB> SELECT * FROM t4 WHERE id IN(26,27,28,29,30) FOR UPDATE; # 待たされる
TC> SELECT * FROM t4 WHERE id IN(26,27,28,29,30) FOR UPDATE; # 待たされる
TA> COMMIT;
TB> # ブロック解除
TC> # 待たされたまま, 正常！
```
検証2  
```text
TA> BEGIN;
TB> BEGIN;
TC> BEGIN;
TA> SELECT * FROM t4 WHERE id = 28 FOR UPDATE;
TB> SELECT * FROM t4 WHERE id IN(26,27,28,29,30) FOR UPDATE; # 待たされる
TC> SELECT * FROM t4 WHERE id IN(26,27,28,29,30) ORDER BY id DESC FOR UPDATE; # 待たされる
TA> COMMIT;
TB> # ブロック解除
TC!> ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```
原因は先程のケースと同じである。  
実際あまり意識する機会はないかもしれないが、待たされているからと言って、  
ロックを獲得していないとは限らない。  


## あるあるロック問題
### 1.よくある交差のパターン  
```sql
TA: BEGIN;
TA: SELECT * FROM tableA WHERE id = 2501 FOR UPDATE;
TB: BEGIN;
TB: SELECT * FROM tableA WHERE id = 2502 FOR UPDATE;
TA: SELECT * FROM tableA WHERE id = 2501 FOR UPDATE; # TBがロック獲得してるので待たされる
TB: SELECT * FROM tableA WHERE id = 2502 FOR UPDATE; # ここでデッドロック
```
多:多の関係で更新するような処理でありがちである  
例えば、フォロー(A->B, B->Aが同時に走る)、チーム移動など  
ロック順序の問題ではなく構造上仕方ないのだが、  
同じテーブルに限りデッドロックを避けることができる  
```text
TA: BEGIN;
TA: SELECT * FROM tableA WHERE id IN (2501,2502) FOR UPDATE;
TB: BEGIN;
TB: SELECT * FROM tableA WHERE id IN (2502,2501) FOR UPDATE; # 待たされる
TA: #ごにょごにょ
TA: COMMIT;
TB: #ごにょごにょ #ロックから解放されて処理が進む
TB: COMMIT; # 幸せ
```
実はIN句のなかの順序はロック獲得処理において関係がない  
__解決方法: 同じテーブルなら一度のクエリでまとめてロックを取る__


### 2.なかったら挿入、あったらロックしたい  
ギャップロックの悲しい事実を上記で説明した通りだが、あるあるパターンの一つである。    
よくあるケースにも関わらず、解決が難しい問題である。現在の自分の見解を述べておく。  
REPEATABLE-READの場合、ギャップロックが発生するため以下のような処理を行っていると、  
Deadlockだけでなく、そのギャップに対するINSERTは全てブロックされることからパフォーマンス悪化にも繋がります。
```text
> SELECT ~ FOR UPDATE
> データがあるなら -> なにもしない
> データがないなら -> INSERT ~
```
READ-COMMITEDなどギャップロックが発生しない分離レベルなら上記でも問題ない。  
あくまでREPEATABLE-READでDuplicateは諦めてパフォーマンス悪化だけを避けたいなら以下のような対処で良い。
```text
> SELECT ~ 
> データがあるなら -> SELECT ~ FOR UPDATE
> データがないなら -> INSERT ~
```
それでも可能ならDuplicateも避けたいという場合は、  
`INSERT ON DUPLICATE KEY UPDATE`を応用すれば回避することができる。
これはデータがなかったら挿入、なかったら更新するMySQL特有の構文である。  
これで無意味な更新を行えば排他ロックだけの獲得になり、直後にSELECTでrowを取得するだけで良くなる。

```text
TA> BEGIN;
TB> BEGIN;
TA> INSERT INTO t4 (id,number) VALUES(30, 100) ON DUPLICATE KEY UPDATE number = number;
TB> INSERT INTO t4 (id,number) VALUES(30, 300) ON DUPLICATE KEY UPDATE number = number; # 待たされる
TA> SELECT * FROM t4 WHERE id = 30;
+----+--------+
| id | number |
+----+--------+
| 30 |    100 |
+----+--------+
1 row in set (0.00 sec)
TA> COMMIT; # TBのブロックが解除
TB> SELECT * FROM t4 WHERE id = 30;
+----+--------+
| id | number |
+----+--------+
| 30 |    100 |
+----+--------+
1 row in set (0.00 sec)
# TA, TBでnumberの値が異なっていても、後続のINSERTパラメーターで上書きされない
```

### 3.外部キー制約による共有ロック  
外部キー制約が設定されていると、挿入時に外部キーに共有ロックがかかる
```text
TA: BEGIN;
TA: INSERT INTO tableA (id,foreign_id) values (1,10);
# foreign_id=10の行にも共有ロックがかかる、例えば
TB: BEGIN;
TB: SELECT * FROM foreign_table WHERE id = 10 FOR UPDATE; # 待たされる
TA: UPDATE foreign_table SET num = num + 1 WHERE id = 10; # デッドロック
```
最後のクエリはupdate文に限らず排他的ロックならデッドロックになる。なぜか？  
同じトランザクション内でも、共有ロック獲得と排他的ロック獲得は別物になります  
ちなみに、ロックの性質から排他的ロックは共有ロックを含んでいるので順序次第ではデッドロックになりません.
```sql
TA: BEGIN;
TA: SELECT * FROM tableA WHERE id = 1001 LOCK IN SHARE MODE;
TB: BEGIN;
TB: SELECT * FROM tableA WHERE id = 1001 FOR UPDATE; # wait
TA: SELECT * FROM tableA WHERE id = 1001 FOR UPDATE; # deadlock
...

TA: BEGIN;
TA: SELECT * FROM tableA WHERE id = 1001 FOR UPDATE;
TB: BEGIN;
TB: SELECT * FROM tableA WHERE id = 1001 FOR UPDATE; # wait
TA: SELECT * FROM tableA WHERE id = 1001 LOCK IN SHARE MODE; # no deadlock !!!!
```
今回の場合は前者です。外部キー制約があり、TAがINSERTで共有ロックを取ってるからと言って、  
あとから排他的ロックが素直に取れるかと言うとそうではありません。  
この隙間に別トランザクションがロックで待たされてると交差扱いとなるデッドロックになります  
外部キー制約を設定しているテーブルの挿入時は、ロックの獲得順序を考えなければいけない
（最初に外部キー側のテーブルをロックで済むなら良いですね＞＜）  

### 4.スナップショットを取るタイミング  
先にも述べたが、REPEATABLE-READではトランザクションを開始して最初のクエリを発行したタイミングのデータベースのスナップショットを取る。  
これは最初のクエリのテーブルだけに限らず、__同じデータベース内の全てが対象__である  
ロックを取るタイミングが遅いと、古い情報を参照してバグに繋がってしまう  
背景：あるユーザーの所属チームを移籍する。移籍後は人数を更新する  
人数管理は行数を数え直しており、仕様の都合で人数によって処理が若干異なる    
今回は同じチームメンバーが別チームに移籍しようとしたとき
```text
TA: BEGIN;
TA: SELECT * FROM user WHERE id = 1 FOR UPDATE; # ユーザー->チームの順でロック
TA: SELECT * FROM team WHERE id = 1001 FOR UPDATE;
TB: SELECT * FROM user WHERE id = 2 FOR UPDATE; # 地点A
TB: SELECT * FROM team WHERE id = 1001 FOR UPDATE; # 同じチームでここで待たされる
TA: SELECT COUNT(*) FROM team_member WHERE team = 1001;
# 人数次第でアプリ側の処理が変わる
TA: COMMIT; # 諸々処理が終わる
TB: SELECT count(*) FROM team_member WHERE team = 1001; # 処理が進む
# TBのcountで獲得したチーム人数は地点A時点での人数になる
# TBで処理してたアプリ側の処理がバグってしまう
```
このようなバグを避けるためには、トランザクションが開始してからは必ず最初のクエリでロックを取り、  
影響がある処理は最初のクエリ(ロック)で止まるようにプログラムを組まなければなりません。   
もしくはRedisを使った排他制御を使うなど、解決策はありますが
InnoDBのREPEATABLE-READのスナップショットの挙動は把握しておきましょう  

### 5.範囲ロック
範囲ロックでまとめてロックを取るときに最後の処理でギャップロックになり、  
INSERTが全て詰まるケースが存在する.  
バッチ処理などで古いデータの更新していくツールを書くときに注意である.
```sql
mysql> desc t2;
+--------+---------+------+-----+---------+----------------+
| Field  | Type    | Null | Key | Default | Extra          |
+--------+---------+------+-----+---------+----------------+
| id     | int(11) | NO   | PRI | NULL    | auto_increment |
| number | int(11) | YES  |     | NULL    |                |
+--------+---------+------+-----+---------+----------------+
2 rows in set (0.00 sec)
mysql> select * from t2;
+----+--------+
| id | number |
+----+--------+
|  1 |      1 |
|  2 |      2 |
|  3 |     10 |
|  4 |     50 |
|  5 |    100 |
|  6 |    110 |
|  7 |    120 |
|  8 |    130 |
|  9 |    140 |
| 10 |    150 |
| 11 |    160 |
| 12 |    170 |
| 13 |    180 |
| 14 |    210 |
| 15 |    220 |
+----+--------+
15 rows in set (0.00 sec)

-----
TA: BEGIN;
TA: UPDATE t2 SET number = number + 1 WHERE id BETWEEN 1 AND 3;
TA: COMMIT;
TA: BEGIN;
TA: UPDATE t2 SET number = number + 1 WHERE id BETWEEN 4 AND 6;
TA: COMMIT;
...
TA: BEGIN;
TA: UPDATE t2 SET number = number + 1 WHERE id BETWEEN 13 AND 15;
TB: INSERT INTO t2 (number) VALUES (999); # 止まる
TC: INSERT INTO t2 (number) VALUES (2000); # 止まる...
```
InnoDB-REPEATABLE-READでは範囲ロックではギャップロック+次の本当のレコードロック(意味不明)になる.
最後の範囲処理だとid=[16,+inf)のギャップロックとなり、挿入が全て止まってしまう.  
現状では範囲ロックで実行する以上避けられないので、  
最後だけ`id = 16 FOR UPDATE`とかでレコードロックを獲得して処理するしかない


### 6.インデックス走査順によるデッドロック  
シャドーロックの説明で走査順でロックを獲得することの例を簡単に示した。  
それの影響を受けてよくあるデッドロックパターンとして、  
あるテーブルに複数のインデックスが張られており、異なるインデックスを利用するクエリが
並行して複数行のロックを獲得しようするときに発生するものがある。  
```text
テーブル定義
CREATE TABLE `shadow_lock` (
  `id` INTEGER unsigned NOT NULL auto_increment,
  `code_id` INTEGER unsigned NOT NULL,
  `token_id` INTEGER unsigned NOT NULL,
  `value` INTEGER unsigned NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE `shadow_lock_code_id` (`code_id`),
  UNIQUE `shadow_lock_token_id` (`token_id`)
) ENGINE=InnoDB DEFAULT CHARACTER SET utf8mb4;

mysql> select * from shadow_lock;
+------+---------+----------+---------+
| id   | code_id | token_id | value   |
+------+---------+----------+---------+
| 1000 |       1 |      299 |   10000 |
| 1001 |       2 |      298 |   20000 |
| 1002 |       3 |      297 |   30000 |
…
| 1099 |     100 |      200 | 1000000 |
| 1100 |     101 |      199 | 1010000 |
| 1101 |     102 |      198 | 1020000 |
| 1102 |     103 |      197 | 1030000 |
| 1103 |     104 |      196 | 1040000 |
| 1104 |     105 |      195 | 1050000 |
| 1105 |     106 |      194 | 1060000 |
| 1106 |     107 |      193 | 1070000 |
| 1107 |     108 |      192 | 1080000 |
| 1108 |     109 |      191 | 1090000 |
| 1109 |     110 |      190 | 1100000 |
| 1110 |     111 |      189 | 1110000 |
| 1111 |     112 |      188 | 1120000 |
...
```

簡単に再現する例として、  
```text
カラムA（昇順） -> id（昇順）=> code_id
カラムB（昇順） -> id（降順）=> token_id
```
このようにセカンダリインデックスからクラスタインデックスへのアクセスが  
逆順になるようなテーブル定義・データを用意する。  

そして、以下のようなクエリを用意する  
```text
TA> BEGIN;
TB> BEGIN;
TA> SELECT * FROM shadow_lock WHERE code_id  IN('100','101','102','103','104','105','106','107','108','109','110') FOR UPDATE;
TB> SELECT * FROM shadow_lock WHERE token_id IN('190','191','192','193','194','195','196','197','198','199','200') FOR UPDATE;
```
**のんびりと手動で実行するとデッドロックは発生しない**  
**のんびりと手動で実行するとデッドロックは発生しない**  

並列処理で高速に実行するとデッドロックが発生する  
検証コード : https://gist.github.com/ichirin2501/f4b22e50356890a52621  

**トランザクションAのロック獲得の概要**
```text
1.セカンダリインデックスのcode_id=100からクラスタインデックスのid=1099をロック  
2.セカンダリインデックスのcode_id=101からクラスタインデックスのid=1100をロック  
...  
9.セカンダリインデックスのcode_id=109からクラスタインデックスのid=1108をロック  
10.セカンダリインデックスのcode_id=110からクラスタインデックスのid=1109をロック  
```

**トランザクションBのロック獲得の概要**  
```text
1.セカンダリインデックスのtoken_id=190からクラスタインデックスのid=1109をロック  
2.セカンダリインデックスのtoken_id=191からクラスタインデックスのid=1108をロック  
...  
9.セカンダリインデックスのtoken_id=199からクラスタインデックスのid=1100をロック  
10.セカンダリインデックスのtoken_id=200からクラスタインデックスのid=1099をロック  
```

上記の通り、各クエリのクラスタインデックスに対するアクセス順が原因でデッドロックになる。  
また、IN句のtoken_idを逆順にしても意味がない。このような場合もあると頭の片隅に置いておきましょう。  
さすがにここまで考慮してアプリは書きたくない気持ちはあるが、回避したいなら  
ロックを獲得する際に用いるインデックスは統一するようにする、ぐらいかと思います。  
例えば、token_idを条件にSELECTで引いたあと、code_idでSELECT-FOR-UPDATEで再取得するなど。  
一度引いてロック獲得のために再取得するのを徹底するなら、PRIMARY-KEYが良いでしょう。 

## パーティション下のロックの挙動

まず、MySQL-5.1から利用できるパーティショニングには以下の種類がある. 

1. RANGE
2. LIST
3. [LINEAR] HASH
4. [LINEAR] KEY

今回は面倒なのでRANGEのみ調査を行った.  
基本を押さえておけば（たぶん）、どれも同じだと思う.  
押さえておくべき基本事項としては、  
__パーティション毎にクラスタインデックスを構築している__  
という点である.  
 
### パーティションの区分けはギャップの切れ目
パーティショニングしていないテーブルのギャップロックは,  
指定箇所のギャップ区間がそのままロックされるのを思い出してください.

```
CREATE TABLE `range_test01` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `num` int(10) unsigned NOT NULL DEFAULT '0',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=2001 DEFAULT CHARSET=utf8mb4;
 
# こんな感じのデータが入っていると仮定
+-----+-----+
| id  | num |
+-----+-----+
...
|  69 |   0 |
|  70 |   0 |
| 108 |  20 |
| 110 |   0 |
| 111 |   0 |
...

# 
mysql> SELECT * FROM range_test01 WHERE id = 80 FOR UPDATE;
# 空打ち、id = [71,100) の範囲にギャップロック.

mysql> SELECT * FROM range_test01 WHERE id < 80 FOR UPDATE;
# 範囲ロックなので、id = [-inf, 108] の区間を排他的ロック
```
パーティショニングされていると、少しギャップロックの範囲が変わります
```
CREATE TABLE `range_test01` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `num` int(10) unsigned NOT NULL DEFAULT '0',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=2001 DEFAULT CHARSET=utf8mb4
/*!50500 PARTITION BY RANGE  COLUMNS(id)
(PARTITION p_type_code0 VALUES LESS THAN (100) ENGINE = InnoDB,
 PARTITION p_type_code1 VALUES LESS THAN (200) ENGINE = InnoDB,
 PARTITION p_type_code2 VALUES LESS THAN (300) ENGINE = InnoDB,
 PARTITION pmax VALUES LESS THAN (MAXVALUE) ENGINE = InnoDB) */
 
 # こんな感じのデータが入っていると仮定
+-----+-----+
| id  | num |
+-----+-----+
...
|  69 |   0 |
|  70 |   0 |
| 108 |  20 |
| 110 |   0 |
| 111 |   0 |
...

# 
mysql> SELECT * FROM range_test01 WHERE id = 80 FOR UPDATE;
# 空打ち、id = [71,100) の範囲にギャップロック.

mysql> SELECT * FROM range_test01 WHERE id < 80 FOR UPDATE;
# 範囲ロックなので、id = [-inf, 100) の区間を排他的ロック

# 刈り込みされていると、パーティションを跨ぐようなロックはかかりません.

mysql> EXPLAIN PARTITIONS SELECT * FROM range_test01 WHERE id < 80 FOR UPDATE;
+----+-------------+--------------+--------------+-------+---------------+---------+---------+------+------+-------------+
| id | select_type | table        | partitions   | type  | possible_keys | key     | key_len | ref  | rows | Extra       |
+----+-------------+--------------+--------------+-------+---------------+---------+---------+------+------+-------------+
|  1 | SIMPLE      | range_test01 | p_type_code0 | range | PRIMARY       | PRIMARY | 8       | NULL |   32 | Using where |
+----+-------------+--------------+--------------+-------+---------------+---------+---------+------+------+-------------+

# partitions項目を見るとわかりますが、検索対象のパーティションが一つになっているからです
# パーティションを跨ぐような条件だと、
mysql> EXPLAIN PARTITIONS SELECT * FROM range_test01 WHERE id < 101 FOR UPDATE;
+----+-------------+--------------+---------------------------+-------+---------------+---------+---------+------+------+-------------+
| id | select_type | table        | partitions                | type  | possible_keys | key     | key_len | ref  | rows | Extra       |
+----+-------------+--------------+---------------------------+-------+---------------+---------+---------+------+------+-------------+
|  1 | SIMPLE      | range_test01 | p_type_code0,p_type_code1 | range | PRIMARY       | PRIMARY | 8       | NULL |   33 | Using where |
+----+-------------+--------------+---------------------------+-------+---------------+---------+---------+------+------+-------------+
1 row in set (0.00 sec)

# 次のパーティションも対象、範囲ロックであるため、id = [-inf, 108] の区間を排他的ロックになります
```

### 刈り込みなしのロックは死が見えるぞい
今度はよくある(id,datetime)をprimary-keyとし,  
datetimeカラムでパーティショニングした想定で動作確認しました.  
datetimeカラムじゃないのはご愛嬌.  
結論から言うと、刈り込みが効かない場合は全パーティションを走査します.  
つまり、各クラスタインデックスに対して、ロックをかけにいくことになります.  
```
CREATE TABLE `range_test02` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `serial` bigint(20) unsigned NOT NULL DEFAULT '0',
  `num` int(10) unsigned NOT NULL DEFAULT '0',
  `rank` int(10) unsigned NOT NULL DEFAULT '0',
  PRIMARY KEY (`id`,`serial`),
  KEY `range_test02_idx_rank` (`rank`)
) ENGINE=InnoDB AUTO_INCREMENT=103 DEFAULT CHARSET=utf8mb4
/*!50500 PARTITION BY RANGE  COLUMNS(`serial`)
(PARTITION p_type_code0 VALUES LESS THAN (100) ENGINE = InnoDB,
 PARTITION p_type_code1 VALUES LESS THAN (200) ENGINE = InnoDB,
 PARTITION p_type_code2 VALUES LESS THAN (300) ENGINE = InnoDB,
 PARTITION pmax VALUES LESS THAN (MAXVALUE) ENGINE = InnoDB) */

+-----+--------+-----+------+
| id  | serial | num | rank |
+-----+--------+-----+------+
|  15 |     53 |   0 |   15 |
|  22 |     60 |   0 |   22 |
|  33 |    110 |   0 |   33 |
|  34 |    111 |   0 |   34 |
|  35 |    112 |   0 |   35 |
|  65 |    210 |   0 |   65 |
|  66 |    211 |   0 |   66 |
|  67 |    212 |   0 |   67 |
+-----+--------+-----+------+
# 実際はもっとデータを入れてます

mysql> SELECT * FROM range_test02 WHERE id = 15 FOR UPDATE;
+----+--------+-----+------+
| id | serial | num | rank |
+----+--------+-----+------+
| 15 |     53 |   0 |   15 |
+----+--------+-----+------+
1 row in set (0.00 sec)

# ('-') 死んだ, explainを見てみましょう

mysql> EXPLAIN PARTITIONS SELECT * FROM range_test02 WHERE id = 20 FOR UPDATE;
+----+-------------+--------------+---------------------------------------------+------+---------------+---------+---------+-------+------+-------+
| id | select_type | table        | partitions                                  | type | possible_keys | key     | key_len | ref   | rows | Extra |
+----+-------------+--------------+---------------------------------------------+------+---------------+---------+---------+-------+------+-------+
|  1 | SIMPLE      | range_test02 | p_type_code0,p_type_code1,p_type_code2,pmax | ref  | PRIMARY       | PRIMARY | 8       | const |    3 |       |
+----+-------------+--------------+---------------------------------------------+------+---------------+---------+---------+-------+------+-------+

# partitions項目を見るとが全て対象になっています.
# これは(id,serial)で複合pkey, かつ, パーティション設定のカラムはserialなので
# idの条件だけでは, SQL側でパーティションの刈り込みが判断できず、全探索じゃ〜ってなってます

# 肝心のロック範囲ですが id = 15 のレコードロック + 
# [p_type_code0] = [ [-inf,-inf], [15,53] )
# [p_type_code0] = [ [15,54], [22,60] )
# [p_type_code1] = [ [-inf,100], [33,110) )
# [p_type_code2] = [ [-inf,200], [65,210] )
# [p_type_code3] = [-inf,+inf]
# [    pmax    ] = [-inf,+inf]
# 以上のギャップがロックされます. 

# ちなみにクエリを並べると以下のようになります
> INSERT INTO range_test02 (id,serial,num,rank) VALUES(13,21,0,13);  # wait
> INSERT INTO range_test02 (id,serial,num,rank) VALUES(20,21,0,20);  # wait
> INSERT INTO range_test02 (id,serial,num,rank) VALUES(23,21,0,23);  # no wait
> INSERT INTO range_test02 (id,serial,num,rank) VALUES(1,120,0,1);   # wait
> INSERT INTO range_test02 (id,serial,num,rank) VALUES(33,111,0,33); # no wait
> ...
```
__ロックするとき、刈り込み、絶対__  
検索するパーティションが決まった後のロックの挙動はパーティションなしと同じです.  
刈り込みなしだとそれらの合成になってしまうため、非常に広いロック範囲になってしまいます.  
LOCK_INSERT_INTENTIONのないGAP-LOCK同士は互いにブロックしません.  
(SELECT-FOR-UPDATE文では止まらないということ)
上記のように刈り込みなしクエリロックだと、そのGAP-LOCKが大量に発生するので  
LOCK_INSERT_INTENTIONフラグを持つINSERT文を同トランザクション内で発行してると、  
あっという間にデッドロック・パフォーマンス悪化に見舞われてしまいます.  