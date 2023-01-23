# MySQLのインデックスについて

---
## 目次
- [目次](#目次)
- [インデックスとは](#インデックスとは)
  - [概要](#概要)
  - [インデックスの種類](#インデックスの種類)
  - [インデックスの落とし穴](#インデックスの落とし穴)
- [簡単な実験](#簡単な実験)
- [まとめ](#まとめ)
- [参考](#参考)
---

## インデックスとは  

### 概要
**インデックス（index）** とは、「レコードの検索を速くするための機能」の一つ。  
テーブルの任意のカラムに対してインデックスを張ることで、そのカラムを対象とするWHERE句によるSELECT文の検索速度が高速になる  

B Tree, B+Tree, R Tree等、主にツリー型のデータ構造を用いて実際のテーブルとは別にレコード検索用のテーブルを作成し、検索時に参照することでレコード全検索（***フルテーブルスキャン***） を避けて効率的に検索することを可能にする

作成したインデックスは `information_schema`の`statistics`テーブルに保存されている. 
<img width="585" alt="Screenshot 2023-01-23 at 20 01 03" src="https://user-images.githubusercontent.com/121989578/214168090-c0892702-2c47-494f-a201-40fb8b068b48.png">


### インデックスの種類

- KEYインデックス
  - PRIMARY KEY / UNIQUE KEYを設定したカラムに自動的に設定される. 最も一般的
- SIMPLE / REGULAR / NORMALインデックス
  - KEYインデックスと同様、一般的に使われるインデックス. KEYインデックスとは違い、NULLが許容されて重複が可能
- FULLTEXTインデックス
  - テキストデータ型（CHAR, VARCHAR, TEXT）用インデックス. 全文検索（fulltext-search）を行う必要のあるカラムに設定される
- SPATIALインデックス
  - GISデータ型（GEOMETRY型）用インデックス. 空間データを扱う

### インデックスの落とし穴
1. なんでもかんでも設定すればいいというものではない
  - インデックスを貼るということは、参照用の別テーブルを用意するということ.  
  つまり、**挿入(INSERT)** や**更新(UPDATE)** 時には元のテーブルだけでなく、参照用のテーブルにも自動的に更新がかかってしまい、結果としてクエリの実行時間が長く（サービス全体として遅く）なってしまう可能性がある.  
  特にカラム数が多いテーブルに対して、すべての組み合わせでマルチカラムインデックスを貼るとかしてしまうと速度的にも容量的にも大変なことになる 
2. カーディナリティが低いカラムに対しては効果的ではない
  - **カーディナリティ（多重度）** が高いカラム、つまり、取りうる値の種類がそもそも少ない場合はインデックスを貼ったとしても絞り込みが効果的にできず、結果として大して速くならない可能性がある.    
  Ex. ユーザーが退会しているかどうか（boolean）、ステータスID（例えば1~5のInteger）  
  これらは前者が約50%、後者は約20%ほどの絞り込みが限度となる.  
  反対に、ユーザー名に対してインデックスを貼った場合は、そうそう被っているレコードは存在しないためカーディナリティが低く、インデックスを使った絞り込みが効果的になる
3. 使用にはSQLクエリの条件がいくつかある
  - **%** から始まるLIKE演算子には使用できない（FULLTEXTインデックスを除く）
    - ```SELECT * FROM tablename WHERE column = '%xxx';```  
    には効かない
  - **マルチカラムインデックス** を使用した上で、単一カラムで検索する際は、左端のprefixに対応するカラムの場合のみ効く
    - ```ADD INDEX (last_name, first_name)```  
    みたいな形で複数のカラムに対してインデックスを貼り、単独でWHERE条件を付加する場合、  
    ```SELECT * FROM tablename WHERE last_name = xxx;```  
    のみインデックスが効き、  
    ```SELECT * FROM tablename WHERE first_name = xxx;```  
    の場合はインデックスが効かない
  - 条件がORグループに所属している場合
    - ```SELECT * FROM tablename WHERE last_name = 'xxx' OR first_name = 'xxx';```  
    は効かない.  ただし、  
    ```SELECT * FROM tablename WHERE last_name = 'xxx' AND (first_name = 'xxx' OR first_name = 'yyy');```  
    は効く.  
    インデックスを指定したカラムがAND条件に属している場合は、その条件の中がORであったとしてもインデックスが作用する
  - フルテーブルスキャンの方が速いとDBが判断した場合
    - レコード数が少ない、インデックスを使っても絞り込みがほとんどできない等、DB側がインデックスを使わない判断を下す場合がある
    - ただし、絞り込みがあまりできなかったとしてもLIMIT句が付いていたりする場合はインデックスが使われたりする（あくまでDB側の判断）
  - サブクエリと使うと落とし穴が多い
    - IN句とサブクエリを組み合わせるとダメ
    - サブクエリ内でGROUP BYするとダメ
    - サブクエリはMySQLのクエリオプティマイザの対象外らしく、内部的によしなにやってくれなくなってしまうらしい

## 簡単な実験
// TBD...

## まとめ
- インデックスとは、レコード検索を速くする機能の一つ
- Tree系のアルゴリズムを用いて参照用のテーブルを作成し、検索時にそれを参照することでクエリ実行速度が高速化される
- 大規模なデータを持つテーブルに対して適切なインデックスを張ることで、フルテーブルスキャンを避けて効率的なレコード検索が可能になる
- 適切な貼り方とは  
  1. レコード数が多い
  2. 挿入・更新の回数が（比較的）多くない（<=> 検索されることが多い）
  3. カーディナリティが高く、設定したインデックスの情報で十分に絞り込むことができる

## 参考
- https://www.educba.com/mysql-index-types/
- https://dev.mysql.com/doc/refman/8.0/ja/optimization-indexes.html
- https://zenn.dev/canalun/articles/all_about_mysql_index
- https://zenn.dev/hk_206/articles/ec5f4e347caff4
- http://tech.aainc.co.jp/archives/397
