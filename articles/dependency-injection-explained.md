---
title: "PHPの依存性注入(Dependency Injection)を正しく理解する"
emoji: "🔧"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["php", "laravel"]
published: true
---

## はじめに

PHPのLaravelアプリケーションを学習している中で、依存性注入（Dependency Injection）という概念に出会いました。私自身、最初は「クラス間の依存性を下げる仕組み」として理解していましたが、それなのになぜ「依存性を注入する」という名前なのかという疑問を持っていました。依存性を高めているような印象を受けてしまうためです。

この記事では、依存性注入の本質的な仕組みと、なぜこの名前が付けられているのかを、改めて整理してまとめてみたいと思います。

## 依存性注入とは何か

まず、依存性注入（Dependency Injection）について確認しておきましょう。

この概念は、Martin Fowlerによって2004年に体系化されました。Fowlerは依存性注入を以下のように説明しています。

> The basic idea of the Dependency Injection is to have a separate object, an assembler, that populates a field in the lister class with an appropriate implementation for the finder interface.

引用：[Inversion of Control Containers and the Dependency Injection pattern - Martin Fowler](https://martinfowler.com/articles/injection.html)

簡潔に言うと、依存性注入とは「サービスの設定とサービスの使用を分離する」技術です。クラスが自分で依存するオブジェクトを作成するのではなく、外部から提供してもらう仕組みを指します。

## なぜ「注入」という名前なのか

それでは、なぜ「注入」という名前が付けられているのかを、具体的なコードで確認してみましょう。

### 依存性注入を使わない場合

```php
class UserController
{
    private $userRepository;
    
    public function __construct()
    {
        // クラス内部で具体的な実装を直接生成
        $this->userRepository = new MySQLUserRepository();
    }
}
```

### 依存性注入を使う場合

```php
class UserController
{
    private $userRepository;
    
    public function __construct(UserRepositoryInterface $userRepository)
    {
        // 外部から依存オブジェクトを「注入」してもらう
        $this->userRepository = $userRepository;
    }
}
```

### 「注入」の本質的な意味

「注入」という言葉が指しているのは以下の点です。

- クラスが**自分で依存オブジェクトを作らない**
- 代わりに**外部から渡してもらう（注入してもらう）**

つまり、「依存性を注入する」= 「必要なものを外から入れてあげる」という意味です。医療現場で薬を注射器で体内に「注入」するのと似たようなイメージで、必要なオブジェクトを外部から「注入」するということです。

## インターフェイスの重要な役割

ここで、上記のコード例に登場した `UserRepositoryInterface` について説明しておく必要があります。依存性注入を効果的に活用するためには、インターフェイスの理解が不可欠だからです。

インターフェイスは「何を実装するか」と「どのように実装するか」を分離する仕組みです。

### インターフェイス：「何を」実装するかを定義

```php
interface UserRepositoryInterface
{
    // 「何を」するかだけを定義
    public function create(array $data): User;
    public function findById(int $id): ?User;
    public function update(int $id, array $data): User;
    public function delete(int $id): bool;
}
```

### 実装クラス：「どのように」実装するかを定義

```php
// MySQLでの「どのような」実装か
class MySQLUserRepository implements UserRepositoryInterface
{
    public function create(array $data): User
    {
        // MySQLでの具体的な保存方法
        $stmt = $this->pdo->prepare("INSERT INTO users (name, email) VALUES (?, ?)");
        $stmt->execute([$data['name'], $data['email']]);
        // ...
    }
    
    public function findById(int $id): ?User
    {
        // MySQLでの具体的な取得方法
        $stmt = $this->pdo->prepare("SELECT * FROM users WHERE id = ?");
        // ...
    }
}
```

### この分離がもたらす効果

この分離により、UserControllerは「どのように」データが保存されるかを知る必要がなくなります。

```php
class UserController
{
    private UserRepositoryInterface $userRepository;
    
    public function __construct(UserRepositoryInterface $userRepository)
    {
        // MySQLなのか、Redisなのか、ファイルなのか知らなくてよい
        $this->userRepository = $userRepository;
    }
    
    public function store(Request $request)
    {
        // 「何を」するかだけに集中できる
        $user = $this->userRepository->create($request->all());
        return response()->json($user);
    }
}
```

## 依存性注入がもたらす4つの改善

依存性注入を使うことで、依存関係自体は残りますが、**依存の仕方が改善**されます。具体的には「密結合」を「疎結合」に変える効果があります。

### 1. 具体的な実装への依存が減る

インターフェイスに依存するようになり、実装の詳細を知る必要がなくなります。

### 2. クラスの責任が明確になる

オブジェクトの生成責任を持たなくなり、本来の処理に専念できます。

### 3. テストしやすくなる

モックオブジェクトを注入できるようになり、単体テストが容易になります。

### 4. 変更に強くなる

実装を切り替えやすくなり、メンテナンス性が向上します。

## Laravelでの実際の活用方法

ここまでの理論を踏まえて、Laravelでの実際の活用方法を見てみましょう。Laravelでは、サービスコンテナが自動的に依存性注入を行ってくれるため、非常に簡潔に実装できます。

### 1. サービスプロバイダでインターフェイスをバインド

```php
// app/Providers/AppServiceProvider.php
class AppServiceProvider extends ServiceProvider
{
    public function register()
    {
        // インターフェイスと実装クラスをバインド
        $this->app->bind(
            UserRepositoryInterface::class,
            MySQLUserRepository::class
        );
        
        $this->app->bind(
            EmailServiceInterface::class,
            SendGridEmailService::class
        );
    }
}
```

### 2. コンストラクタでインターフェイスを指定するだけ

```php
class UserService
{
    public function __construct(
        private UserRepositoryInterface $userRepository,
        private EmailServiceInterface $emailService
    ) {
        // Laravelのサービスコンテナが自動的に
        // MySQLUserRepositoryとSendGridEmailServiceを注入してくれる
    }
    
    public function createUser(array $userData): User
    {
        $user = $this->userRepository->create($userData);
        $this->emailService->sendWelcomeEmail($user);
        
        return $user;
    }
}
```

### 3. 実装を変更したい場合の柔軟性

将来的に実装を変更したい場合は、サービスプロバイダのバインドを変更するだけで済みます。

```php
// 実装を変更したい場合は、サービスプロバイダのバインドを変更するだけ
public function register()
{
    $this->app->bind(
        UserRepositoryInterface::class,
        RedisUserRepository::class  // MySQLからRedisに変更
    );
    
    $this->app->bind(
        EmailServiceInterface::class,
        MailgunEmailService::class  // SendGridからMailgunに変更
    );
}
```

この仕組みにより、UserServiceクラスやUserControllerクラスを一切変更することなく、データベースの実装やメールサービスの実装を切り替えることができます。

## テスト時の劇的な改善

依存性注入の最も分かりやすい恩恵は、テスト時に現れます。ビフォーアフターで比較してみましょう。

### Before: 依存性注入を使わない場合のテスト

```php
class UserService
{
    private $userRepository;
    private $emailService;
    
    public function __construct()
    {
        $this->userRepository = new MySQLUserRepository();
        $this->emailService = new SendGridEmailService();
    }
    
    public function createUser(array $userData): User
    {
        $user = $this->userRepository->create($userData);
        $this->emailService->sendWelcomeEmail($user);
        return $user;
    }
}

class UserServiceTest extends TestCase
{
    public function test_user_creation()
    {
        // 問題点：
        // - 実際のデータベース接続が必要
        // - 実際にメールが送信されてしまう
        // - テストが遅い
        // - 外部サービスに依存するため不安定
        
        $userService = new UserService();
        $user = $userService->createUser(['name' => 'Test User']);
        
        // データベースから実際に確認する必要がある
        $this->assertDatabaseHas('users', ['name' => 'Test User']);
        
        // メールが送信されたかどうかをどう確認する？
    }
}
```

### After: 依存性注入を使った場合のテスト

```php
class UserService
{
    public function __construct(
        private UserRepositoryInterface $userRepository,
        private EmailServiceInterface $emailService
    ) {}
    
    public function createUser(array $userData): User
    {
        $user = $this->userRepository->create($userData);
        $this->emailService->sendWelcomeEmail($user);
        return $user;
    }
}

class UserServiceTest extends TestCase
{
    public function test_user_creation()
    {
        // 改善点：
        // - データベース接続不要
        // - メール送信されない
        // - 高速なテスト
        // - 確実で安定したテスト
        
        // モックを作成
        $userRepository = Mockery::mock(UserRepositoryInterface::class);
        $emailService = Mockery::mock(EmailServiceInterface::class);
        
        // 期待する動作を設定
        $userRepository->shouldReceive('create')
                      ->once()
                      ->with(['name' => 'Test User'])
                      ->andReturn(new User(['id' => 1, 'name' => 'Test User']));
                      
        $emailService->shouldReceive('sendWelcomeEmail')
                    ->once()
                    ->with(Mockery::type(User::class));
        
        // テスト実行
        $userService = new UserService($userRepository, $emailService);
        $result = $userService->createUser(['name' => 'Test User']);
        
        // 結果検証
        $this->assertEquals('Test User', $result->name);
    }
}
```

この比較から分かるように、依存性注入により**テストが高速で確実、かつ外部に依存しないもの**に変わります。

## 依存性注入の3つのパターン

最後に、Martin Fowlerが定義した依存性注入の3つのパターンを確認しておきましょう。

### コンストラクタインジェクション（推奨）

```php
class OrderService
{
    public function __construct(
        private PaymentServiceInterface $paymentService,
        private NotificationServiceInterface $notificationService
    ) {}
}
```

### セッターインジェクション

```php
class OrderService
{
    private PaymentServiceInterface $paymentService;
    
    public function setPaymentService(PaymentServiceInterface $paymentService): void
    {
        $this->paymentService = $paymentService;
    }
}
```

### メソッドインジェクション

```php
class OrderService
{
    public function processOrder(
        Order $order,
        PaymentServiceInterface $paymentService
    ): void {
        // 処理
    }
}
```

Laravelでは主にコンストラクタインジェクションが使用され、これが最も安全で推奨される方法です。

## まとめ

依存性注入について整理すると以下のようになります。

- **「依存性」は残る** → UserControllerはRepositoryが必要
- **「依存の方法」が変わる** → 自分で作らず、注入してもらう
- **結果として疎結合になる** → 具体的な実装を知らなくて済む

「依存性注入」という名前は確かに紛らわしいですが、「依存するものを外から注入する」という**手法の名前**であって、依存性を増やすという意味ではありません。

また、インターフェイスによる「何を実装するか」と「どのように実装するか」の分離が、依存性注入の効果を最大化する重要な要素です。

この仕組みを理解することで、よりメンテナンスしやすく、テストしやすいPHPアプリケーションを構築できるようになります。依存性注入は現代的なPHP開発において必須の概念であり、Laravelをはじめとする多くのフレームワークで採用されている重要な設計パターンです。