---
title: TypeGraphQLつかってみた
emoji: 🔥
type: tech
topics:
  - typescript
  - graphql
  - typegraphql
published: false

---
TypescriptでGraphQLサーバを書こうと思ったときにいろいろな選択肢があります。

[TypeGraphQL](https://typegraphql.com/)を業務で半年ほど使う機会がありましたので、そこで得られた知見を書いていきたいと思います。

TypeGraphQL自体の使い方については特に解説しません。

新しめのフレームワークで日本語の情報もかなり少ないため、利用しようと思っている方の参考になればと思います。

# TypeGraphQLとは
[TypeGraphQL](https://typegraphql.com/)はTypescriptで書かれたGraphQLサーバーフレームワークです。

次のような特徴があります。
- Code FirstでのGraphQLサーバ開発
- Typescriptとの親和性
- デコレータを利用したスキーマ定義

それでは詳しく見ていきましょう。

## Code FirstでのGraphQLサーバ開発
GraphQLサーバを開発するにあたって主要なアプローチが２つあります。
`Schema First`と`Code First`です.

`Schema First`はGraphQLスキーマ言語でスキーマを記述し、それをマスターとしてサーバコードを記述するアプローチです。

`Code First`はGraphQLサーバのコードを先に記述し、そこからGraphQLスキーマ定義を生成するアプローチです。

TypeGraphQLでは`Code First`で開発することができます。

## Typescriptとの親和性
TypeGraphQLは名前の通りTypescriptで書かれています。

全体的にきちんと型付けされており、フレームワークにありがちなTypescript用の初期設定等も必要ないため
Typescriptからはストレスなく使えるかと思います。

## デコレータを利用したスキーマ定義
TypeGraphQLではデコレータを利用してスキーマを定義します。

デコレータを実務利用することに賛否両論あるかと思います。
ただ、デコレータを利用していることで生産性は大幅に向上している印象です。

スキーマとサーバコードの差異に悩まされることはほぼありませんでした。

# 成功した取り組み
## Code Firstでサーバを実装する
`Schema First`はswaggerで実践しようとして結局断念し、苦い思い出ありました。

今回色々調べて`Code First`で実装してみましたが、これは正解でした。

`Schema First`に比べてスキーマとサーバコードを一致させるための苦労が圧倒的に少ないです。

デコレータがtypescriptではexperimental, 仕様的にもまだstage2であることを差し置いても採用する価値があると感じました。

（ちなみに、２つのアプローチの比較は[こちら](https://www.prisma.io/blog/the-problems-of-schema-first-graphql-development-x1mn4cb0tyl3)が大変参考になりました。）

## ディレクトリ構造のルールを決める
TypeGraphQLはスキーマ定義やResolverの実装をするための機能は提供していますが、
ディレクトリ構造の決まりやガイドラインはありません。

特に決めずに開発し始めたところ、ひどいことになりました。
- 1つのResolverにqueryやmutationが入り乱れる
- どこに何が入っているかわからなくなる
- 型が再利用しづらい

なんとなくで分けているとすぐにぐちゃぐちゃになりますし、
ぐちゃぐちゃな型がスキーマ定義にも現れてくるのでとても使いにくいスキーマになってしまいました。

自分のチームでは以下のようなディレクトリ構造を決めました

```
- mutations/
  - CreateUser
    - CreateUserArgs.ts
    - CreateUserInput.ts
    - CreateUserResolver.ts
    - CreateUserPayload.ts
- queries/
  - Users/
    - UsersArgs.ts
    - UsersResolver.ts
- resources/
  - User/
    - User.ts
    - UserEdge.ts
    - UserConnection.ts
    - UserResolver.ts
```

### mutations
- mutationはmutation毎に`mutations/Mutation名`の形でサブディレクトリを切る
- 引数はinput１つにし、inputにパラメータを追加していく
  - GraphQL BestPracticeから拝借
- mutationの戻り値は`Mutation名Payload`という名前にする
  - GitHubのGraphQLスキーマから拝借

### queries
- queryはquery毎に`queries/Query名`の形でサブディレクトリを切る
- 引数に直接パラメータを追加していく
- 戻り値の型は任意

### resources
- GraphQLスキーマで再利用するtypeを`resources/名前`の形でサブディレクトリを切る
- そのtypeに対するField Resolverも同じディレクトリで実装する
- [Relay Style Pagination](https://qiita.com/gipcompany/items/ffee8cf0b1522a741e12)用のEdge, Connectionもここで定義する

ここが一番肝でした。

mutationやquery、Field Resolver等から共通的に利用する型を定義出来るようになったことで再利用性が高まりました。
typeを再利用したことで、GraphQLクエリでオブジェクトをネストさせることが出来るようになって、汎用性や使いやすさが向上しました。

typeを再利用することは実装のしやすさ/わかりやすさだけではなく「よいスキーマ設計」を促す気がします。

## コードジェネレータを利用する
明確なルールが決まったことで、コードジェネレータを利用して雛形を生成出来るようになりました。

コードジェネレータには[Plop.js](https://plopjs.com/)を利用しました。

`plop mutation`を実行し、対話的に名前等を入力するだけで雛形を生成出来るようになり、かなり実装が楽になりました。

# 失敗した取り組み/やればよかったこと
## ユニットテストを実装しない
resolverは薄い層かなと思いユニットテストを実装しませんでした。
しかし、実装を進めていくとそんなに薄い層でも無いことがわかって来ました。

HTTPセッションやファサード的な便利機能はResolverで実装したほうが良いことに気づきましたが、
そのときにはすでにかなりの数のResolverを書いており、ユニットテストが入れづらい状況になってしまいました。

コードジェネレータでテストコードの雛形も同時生成すればよかったと思います。

## 1つのField Resolverにごちゃまぜにする
resourceに対するField Resolverは１つのクラスに実装していました。

`User`型でfollowersとarticlesを取りたいとすると次のようなクエリになるかと思います。

```graphql
qeury {
  user(id: "user id") {
    id
    username
    followers {
      edges {
        node {
          id
          username
        }
      }
    }
    articles {
      edges {
        node {
          id
          title
        }
      }
    }
  }
}
```


この場合`User`型には次の2つのField Resolverが必要になります。
- `followers`: ユーザIDからフォロワーを取得する処理
- `articles`: ユーザIDから投稿記事を取得する処理

これを1つのResolverにまとめて実装していましたが、必要とする依存クラスや処理が全く異なることに気づきました。

Field Resolverは別個のクラスにすべきでした。

## EdgeとConnectionの処理を共通化しない
[Relay Style Pagination](https://qiita.com/gipcompany/items/ffee8cf0b1522a741e12)ではEdge, Connectionという型が登場します。

これはTypescriptだと`Edge<T>` `Connection<T>`のようにしたいところですが、
GraphQLにはジェネリックは無いため`HogeEdge``HogeConnection`のように別個に作ってあげる必要があります。

コードジェネレータでそれらを自動生成するところまでは良かったのですが、
PageInfoを作ったりcursorを設定する処理等が共通化できなかったため同じコードがコピペされて利用される状況になってしまいました。

これはcursor周りをうまく抽象化できていなかったせいでもあるのですが、EdgeやConnectionの処理は共通化すべきでした。

# まとめ
いろいろ不都合な部分もありましたが、全体的には導入して正解でした。
次に業務でGraphQLサーバを書く機会があれば筆頭候補に挙げるかと思います。
