# Go言語でClean Architectureを実現します。

## DI(Dependency Injection)パターンとは

DIはDependency Injection(依存性の注入)の略称で。ある処理に必要なオフジェクト(や関数)を外部から注入(指定)できるようにする実装パターンです。依存するオプジェクトを注入すること自体を指す場合もあります。
具体的には依存するオプジェクトをInterfaceとして定義し、Interfaceにのみ依存させることで、実装を入れ替えられるようにするというものです。

## なぜDIするか？

DIする利点として挙げられるのは主に次の二点です。
1. ユニットテストが書きやすくなる
2. オプジェクト間の結合度を下げやすくなる
一番大きいのはユニットテストが書きやすくなることです。依存先がInterfaceになっていることで、その実装を入れ替えることができ、テスト時にモックを使ったユニットテストができるようになります。
また、Interfaceになっていることで、内部のフィールドにアクセスできなくなり、オプジェクト間の結合度が下げやすくなるという利点があります。だだし、無闇にGetterやSetterを追加することの利点は失われてしまうので注意が必要です。

## Go言語でDIの実現方法
Goでは依存先をInterfaceとして定義し、structのフィールドとして持たせるによって、DIができるようになります。
例題として、次のような機能を考えてみましょう。
- メールアドレスとパスワードでユーザーを作成して保存する
- 成功したら登録完了メールを送信する。

```go
type SignUpService interface {
    SignUp(email, password string) error
}

type signUpService struct {}

func (s signUpService) SignUp(email, password string) error {
    // to be implemented
}
```
`SignUpService`は`User`を保存する処理と、メールを送信する処理に依存します。なので`User`を保存する処理として`UserReponsitory`、メールを送信するための処理として`Mailer`を`interface`として定義します。
```go
type UserRepository interface {
    Save(u User) error
}

type Mailer interface {
    SendEmail(to, message) error
}
```
この二つのinterfaceをstructのフィールドに持たせて`SignUpService`を完成させます。
```go
type signUpService struct {
    repo   UserRepository
    mailer Mailer
}

func (s signUpService) SignUp(email, password string) error {
    u := NewUser(email, password)
    if err := s.repo.Save(u); err != nil {
        return err
    }
    return s.mailer.SendEmail(email, "登録完了")
}

func NewSignUpService(repo UserRepository, mailer Mailer) SignUpService {
    return signUpService{
        repo,
        mailer,
    }
}
```
これで`SignUpService`は依存先がinterfaceのみになったため、モックを使ったユニットテストが書けます。ただし、アプリケーションを実行するときにはこれで終わりではありません。`UserReponsitory`や`Mailer`のインスタンスを取得するためにはコンストラクタが必要ので用意します。
```go
func NewUserRepository(db DB) UserRepository {
    ...
}

func NewMailer() {
    ...
}
```
`UserRepository`は`DB`に依存しています。`DB`は`*sql.DB`のメソッドを定義した`interface`だとしましょう。 `Mailer`は依存がないため引数なしで実装を取得できます。

次はこのコンストラクタを使い、どうやって依存関係を解決するかについて次の3つの方法を紹介します。
- mainに書く
- DIコンテナを使う
- DI用に関数を定義する

### mainに書く
ひとつ目は`main`で全ての依存関数を解決する方法です。
```go
func main() {
    db, err := sql.Open("db", "dsn")
    if err != nil {
        panic(err)
    }
    repo := NewUserRepository(db)
    mailer := NewMailer()
    service := NewSignUpService(repo, mailer)
    ...
}
```
シンプルでsyが、使用するオプジェクト数に比例として`main`が肥大していくという問題があります。
### DIコンテナを使う

[goldi](https://github.com/fgrosse/goldi)ではyamlで依存関係を解決します。
```yaml
types:
    db:
        package: database/sql
        type: *DB
        factory: Open
        arguments:
            - "db"
            - "dsn"
    repository:
        package: github.com/morikuni/hoge
        type: UserRepository
        factory: NewUserRepository
        arguments:
            - "@db"
    mailer:
        package: github.com/morikuni/hoge
        type: Mailer
        factory: NewMailer
    service:
        package: github.com/morikuni/hoge
        type: SignUpService
        factory: NewSignUpService
        arguments:
            - "@repository"
            - "@mailer"
```
`goldigen`というコマンドにこの`yaml`を渡すことで`RegisterTypes`という関数が生成されます。DIコンテナにこの関数を適用することで依存関係が解決できるようになります。
```go
func main() {
    registry := goldi.NewTypeRegistry()
    RegisterTypes(registry)
    container := goldi.NewContainer(registry, nil)

    service := container.MustGet("service").(SignUpService)
    ...
}
```
DIコンテナを使うことでmainが肥大化していくことはなくなります。
ただし、DIコンテナの使い方を覚える必要があったり、最終的にはGoのコードになるといえyamlは直接コンパイルできないので、コンパイルエラーのフィードバックを得られるまでの手間が増えてしまうという問題があります。

### DI用の関数を定義する
3つめはDI用の関数を用意する方法です。 この関数を`Inject`関数と呼ぶことにします。 `Inject`関数は次のような関数です。
- オプジェクトを引数0で取得できるようにする
- オブジェクトのコンストラクタに対して他の`Inject`関数を使ってオブジェクトを注入する
  
あるオブジェクトについて、依存先が引数0個で取得できれば、そのアブジェクトも引数0個で取得できるので、これを組み合わせるという物です。
```go
func InjectDB() DB {
    db, err := sql.Open("db", "dsn")
    if err != nil {
        panic(err)
    }
    return db
}

func InjectUserRepository() UserRepository {
    return NewUserRepository(
        InjectDB(),
    )
}

func InjectMailer() Mailer {
    return NewMailer()
}

func InjectSignUpService() SignUpService {
    return NewSignUpService(
        InjectUserRepository(),
        InjectMailer(),
    )
}

func main() {
    service := InjectSignUpService()
    ...
}
```
最初に`InjectDB`を定義しています。 これはsql.Openを使って*sql.DBを返す関数です。 `InjectDB`を使うことでDBが引数0個で取得できるので、`UserRepository`も引数0個で取得できるようになります。 同様に`SignUpService`も`InjectRepository`と`InjectMailer`を使うことで引数0個で取得できます。 このように`Inject`関数を組み合わせること依存関係を解決するのが`Inject`関数です。

1つ気になるのは、`InjectDB`内で`panic`を使っていることです。 `Go`の文化としてはできる限り`panic`を使わないことが望ましいと思いますが、`Inject`関数を使うのは`main`関数内だけのはずです。 つまり`Inject`関数はアプリケーションの初期化の時だけに呼ばれることになります。 `sql.Open`などに失敗すると言うことは初期化に失敗したということなので、おかしな状態で起動するよりは`panic`で起動に失敗してしまったほうがよいのではないかと思います。 どうしても`panic`させたくなければ、ログを吐いた後で`os.Exit`するという方法でも構いません。

`Inject`関数を定義することの利点としては次のようなことを挙げられます。
-  依存先が増減しても影響範囲は`Inject関数内のみ`
   - 例：`SignUpService`が`Logger`に依存するようになっても、`InjectSignUpService`に1行足すだでよい
- 実装が書かれる`package`が変わっても影響範囲は`Inject`関数内のみ
  - 例: `DB`の実装が`database/sql`パッケージから`datastore`パッケージに変わっても`InjectDB`で呼び出すコンストラクタを変更するだけでよい (interfaceが同じ限りは)

## Clean Architectureについて
ソフトウェアの各レイヤーの依存関係を工夫し、保守性を向上させたアーキテクチャパターンです。`Clean Architecture`では、上記のDIパターンによって、依存性を逆転させます。これにより、ビジネスロジックが`DB`、Webフレームワークに依存しなくなり、`DB`や`UI`の変更が発生した際に、ビジネスロジックに手を入れる必要がなくなります。