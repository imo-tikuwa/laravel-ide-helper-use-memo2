# laravel-ide-helper-use-memo2
PHP8.2 + Laravel 10.x + Postgresql 16.x の開発環境において、[laravel-ide-helper](https://github.com/barryvdh/laravel-ide-helper)を使ったモデルクラスへのPHPDocの補完を行う際に詰まった内容についてまとめる2

## 問題点
- Laravel10.xにおいて現形式のアクセサの記述を行った場合、PHPDocに期待する@property-readのプロパティが生えない
- Laravel8.xの頃のget〇〇Attributeの記述を行った場合、PHPDocに期待する@property-readのプロパティが生える

## 2種類のアクセサについて
まずモデルクラスに記述するアクセサについておさらい

---
以下はLaravel8.xの頃の記述
```php
class User extends Model
{
    /**
     * @comment ユーザー名とパスワードを連結して取得1
     *
     * @return string
     */
    public function getNameAndEmail1Attribute(): string
    {
        return "{$this->name} {$this->email}";
    }
}
```

- get〇〇Attributeというルールで記述する
- メソッドの戻り値としてアクセサとして返したい値を自由に設定する
- 参考：https://readouble.com/laravel/8.x/ja/eloquent-mutators.html

---
以下はLaravel9.x以降の記述
```php
class User extends Model
{
    /**
     * @comment ユーザー名とパスワードを連結して取得2
     *
     * @return \Illuminate\Database\Eloquent\Casts\Attribute
     */
    protected function nameAndEmail2(): Attribute
    {
        return Attribute::make(
            get: fn () => "{$this->name} {$this->email}",
        );
    }
}
```

- `\Illuminate\Database\Eloquent\Casts\Attribute`というインスタンスを返す
- `Attribute::make`のget引数にセットする関数によってアクセサとして返したい値を設定する
- 参考：https://readouble.com/laravel/9.x/ja/eloquent-mutators.html
- 参考：https://readouble.com/laravel/10.x/ja/eloquent-mutators.html
- ちなみに9.x↑のLaravelでも8.xの記述は可能

## テーブル構成
テーブルを作成するマイグレーションは以下の通り
```php:2014_10_12_000000_create_users_table.php
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('email')->unique();
    $table->timestamp('email_verified_at')->nullable();
    $table->string('password');
    $table->rememberToken();
    $table->timestamps();
});
```

## 検証1
2024/01/30現在の最新のリリース([v2.13](https://github.com/barryvdh/laravel-ide-helper/releases/tag/v2.13.0))で確認
```bash
$ composer require --dev barryvdh/laravel-ide-helper:^2.13
```

このとき `php artisan ide-helper:models --write --reset` によってUserモデルのPHPDocを補完を行ったところ以下のようになった

```php
<?php

namespace App\Models;

// use Illuminate\Contracts\Auth\MustVerifyEmail;

use Illuminate\Database\Eloquent\Casts\Attribute;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Sanctum\HasApiTokens;

/**
 * App\Models\User
 *
 * @property int $id
 * @property string $name
 * @property string $email
 * @property \Illuminate\Support\Carbon|null $email_verified_at
 * @property mixed $password
 * @property string|null $remember_token
 * @property \Illuminate\Support\Carbon|null $created_at
 * @property \Illuminate\Support\Carbon|null $updated_at
 * @property-read string $name_and_email1 ユーザー名とパスワードを連結して取得1
 * @property-read \Illuminate\Notifications\DatabaseNotificationCollection<int, \Illuminate\Notifications\DatabaseNotification> $notifications
 * @property-read int|null $notifications_count
 * @property-read \Illuminate\Database\Eloquent\Collection<int, \Laravel\Sanctum\PersonalAccessToken> $tokens
 * @property-read int|null $tokens_count
 * @method static \Database\Factories\UserFactory factory($count = null, $state = [])
 * @method static \Illuminate\Database\Eloquent\Builder|User newModelQuery()
 * @method static \Illuminate\Database\Eloquent\Builder|User newQuery()
 * @method static \Illuminate\Database\Eloquent\Builder|User query()
 * @method static \Illuminate\Database\Eloquent\Builder|User whereCreatedAt($value)
 * @method static \Illuminate\Database\Eloquent\Builder|User whereEmail($value)
 * @method static \Illuminate\Database\Eloquent\Builder|User whereEmailVerifiedAt($value)
 * @method static \Illuminate\Database\Eloquent\Builder|User whereId($value)
 * @method static \Illuminate\Database\Eloquent\Builder|User whereName($value)
 * @method static \Illuminate\Database\Eloquent\Builder|User wherePassword($value)
 * @method static \Illuminate\Database\Eloquent\Builder|User whereRememberToken($value)
 * @method static \Illuminate\Database\Eloquent\Builder|User whereUpdatedAt($value)
 * @mixin \Eloquent
 */
class User extends Authenticatable
{
    // use宣言、$fillable、$hidden、$castsは省略

    /**
     * @comment ユーザー名とパスワードを連結して取得1
     *
     * @return string
     */
    public function getNameAndEmail1Attribute(): string
    {
        return "{$this->name} {$this->email}";
    }

    /**
     * @comment ユーザー名とパスワードを連結して取得2
     *
     * @return \Illuminate\Database\Eloquent\Casts\Attribute
     */
    protected function nameAndEmail2(): Attribute
    {
        return Attribute::make(
            get: fn () => "{$this->name} {$this->email}",
        );
    }
}
```

- Laravel8.xの書き方の方はPHPDocに`name_and_email1`というプロパティでアクセスできることを示すアノテーション定義が生えている
- Laravel9.x以降の書き方をした方について`name_and_email2`のようなプロパティでアクセスできることを示すアノテーション定義が生えていない

## 検証2
2024/01/30現在の最新のmasterのコミットID([726e5955786969b7c650d989657b55c12d4521e1](https://github.com/barryvdh/laravel-ide-helper/tree/726e5955786969b7c650d989657b55c12d4521e1))で確認
```bash
$ composer require --dev barryvdh/laravel-ide-helper:dev-master#726e5955786969b7c650d989657b55c12d4521e1
```

このとき `php artisan ide-helper:models --write --reset` によってUserモデルのPHPDocを補完を行ったところ以下のようになった

```php
<?php

namespace App\Models;

// use Illuminate\Contracts\Auth\MustVerifyEmail;

use Illuminate\Database\Eloquent\Casts\Attribute;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Sanctum\HasApiTokens;

/**
 * App\Models\User
 *
 * @property int $id
 * @property string $name
 * @property string $email
 * @property \Illuminate\Support\Carbon|null $email_verified_at
 * @property mixed $password
 * @property string|null $remember_token
 * @property \Illuminate\Support\Carbon|null $created_at
 * @property \Illuminate\Support\Carbon|null $updated_at
 * @property-read string $name_and_email1 ユーザー名とパスワードを連結して取得1
 * @property-read \Illuminate\Notifications\DatabaseNotificationCollection<int, \Illuminate\Notifications\DatabaseNotification> $notifications
 * @property-read int|null $notifications_count
 * @property-read \Illuminate\Database\Eloquent\Collection<int, \Laravel\Sanctum\PersonalAccessToken> $tokens
 * @property-read int|null $tokens_count
 * @method static \Database\Factories\UserFactory factory($count = null, $state = [])
 * @method static \Illuminate\Database\Eloquent\Builder|User newModelQuery()
 * @method static \Illuminate\Database\Eloquent\Builder|User newQuery()
 * @method static \Illuminate\Database\Eloquent\Builder|User query()
 * @method static \Illuminate\Database\Eloquent\Builder|User whereCreatedAt($value)
 * @method static \Illuminate\Database\Eloquent\Builder|User whereEmail($value)
 * @method static \Illuminate\Database\Eloquent\Builder|User whereEmailVerifiedAt($value)
 * @method static \Illuminate\Database\Eloquent\Builder|User whereId($value)
 * @method static \Illuminate\Database\Eloquent\Builder|User whereName($value)
 * @method static \Illuminate\Database\Eloquent\Builder|User wherePassword($value)
 * @method static \Illuminate\Database\Eloquent\Builder|User whereRememberToken($value)
 * @method static \Illuminate\Database\Eloquent\Builder|User whereUpdatedAt($value)
 * @mixin \Eloquent
 */
class User extends Authenticatable
{
    // use宣言、$fillable、$hidden、$castsは省略

    /**
     * @comment ユーザー名とパスワードを連結して取得1
     *
     * @return string
     */
    public function getNameAndEmail1Attribute(): string
    {
        return "{$this->name} {$this->email}";
    }

    /**
     * @comment ユーザー名とパスワードを連結して取得2
     *
     * @return \Illuminate\Database\Eloquent\Casts\Attribute
     */
    protected function nameAndEmail2(): Attribute
    {
        return Attribute::make(
            get: fn () => "{$this->name} {$this->email}",
        );
    }
}
```

- 検証1の頃と比べて変化なし
- 依然として現形式のアクセサについてPHPDocには補完のためのプロパティが生えないことを確認

## 検証3
- laravel-ide-helperのModelsCommand内を調査した結果、`getPropertiesFromMethods`メソッドで現形式のアクセサについてPHPDocにプロパティを生やすためには`Attribute::make`のget引数にセットする関数に戻り値の型の明示が必要なことが分かった。
  - 該当箇所 ⇒ [getPropertiesFromMethodsメソッドのAttributeの@property-readを設定するためのコード](https://github.com/barryvdh/laravel-ide-helper/blob/726e5955786969b7c650d989657b55c12d4521e1/src/Console/ModelsCommand.php#L645-L652)
- 検証2の状態に加えて以下の修正を実施

```diff
class User extends Authenticatable
{
    /**
     * @comment ユーザー名とパスワードを連結して取得2
     *
     * @return \Illuminate\Database\Eloquent\Casts\Attribute
     */
    protected function nameAndEmail2(): Attribute
    {
        return Attribute::make(
-            get: fn () => "{$this->name} {$this->email}",
+            get: fn (): string => "{$this->name} {$this->email}",
        );
    }
}
```

このとき `php artisan ide-helper:models --write --reset` によってUserモデルのPHPDocを補完を行ったところ以下のようになった

```php
<?php

namespace App\Models;

// use Illuminate\Contracts\Auth\MustVerifyEmail;

use Illuminate\Database\Eloquent\Casts\Attribute;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Sanctum\HasApiTokens;

/**
 * App\Models\User
 *
 * @property int $id
 * @property string $name
 * @property string $email
 * @property \Illuminate\Support\Carbon|null $email_verified_at
 * @property mixed $password
 * @property string|null $remember_token
 * @property \Illuminate\Support\Carbon|null $created_at
 * @property \Illuminate\Support\Carbon|null $updated_at
 * @property-read string $name_and_email1 ユーザー名とパスワードを連結して取得1
 * @property-read string $name_and_email2 ユーザー名とパスワードを連結して取得2
 * @property-read \Illuminate\Notifications\DatabaseNotificationCollection<int, \Illuminate\Notifications\DatabaseNotification> $notifications
 * @property-read int|null $notifications_count
 * @property-read \Illuminate\Database\Eloquent\Collection<int, \Laravel\Sanctum\PersonalAccessToken> $tokens
 * @property-read int|null $tokens_count
 * @method static \Database\Factories\UserFactory factory($count = null, $state = [])
 * @method static \Illuminate\Database\Eloquent\Builder|User newModelQuery()
 * @method static \Illuminate\Database\Eloquent\Builder|User newQuery()
 * @method static \Illuminate\Database\Eloquent\Builder|User query()
 * @method static \Illuminate\Database\Eloquent\Builder|User whereCreatedAt($value)
 * @method static \Illuminate\Database\Eloquent\Builder|User whereEmail($value)
 * @method static \Illuminate\Database\Eloquent\Builder|User whereEmailVerifiedAt($value)
 * @method static \Illuminate\Database\Eloquent\Builder|User whereId($value)
 * @method static \Illuminate\Database\Eloquent\Builder|User whereName($value)
 * @method static \Illuminate\Database\Eloquent\Builder|User wherePassword($value)
 * @method static \Illuminate\Database\Eloquent\Builder|User whereRememberToken($value)
 * @method static \Illuminate\Database\Eloquent\Builder|User whereUpdatedAt($value)
 * @mixin \Eloquent
 */
class User extends Authenticatable
{
    // use宣言、$fillable、$hidden、$castsは省略

    /**
     * @comment ユーザー名とパスワードを連結して取得1
     *
     * @return string
     */
    public function getNameAndEmail1Attribute(): string
    {
        return "{$this->name} {$this->email}";
    }

    /**
     * @comment ユーザー名とパスワードを連結して取得2
     *
     * @return \Illuminate\Database\Eloquent\Casts\Attribute
     */
    protected function nameAndEmail2(): Attribute
    {
        return Attribute::make(
            get: fn (): string => "{$this->name} {$this->email}",
        );
    }
}
```

- 現形式のアクセサにアクセスするためのプロパティ(`name_and_email2`)がPHPDocに生えることを確認！

### ModelsCommandの調査について
vendorディレクトリ以下に存在するcomposerでインストールしたプログラムに対して`info()`を埋め込み変数に何が入っているのかを調査することで解決できた

vendor/barryvdh/laravel-ide-helper/src/Console/ModelsCommand.php
```diff
~~~省略~~~
    public function getPropertiesFromMethods($model)
    {
~~~省略~~~
                    $name = Str::snake($method);
                    $types = $this->getAttributeReturnType($model, $reflection);
                    $comment = $this->getCommentFromDocBlock($reflection);
+                   info($name);
+                   info($types);
+                   info($comment);

                    if ($types->has('get')) {
                        $type = $this->getTypeInModel($model, $types['get']);
                        $this->setProperty($name, $type, true, null, $comment);
                    }
~~~省略~~~
```
↑の`info()`を埋め込んだ状態で`php artisan ide-helper:models --write --reset`を実行することで以下のログ出力を得られた
```
[2024-01-30 05:49:29] production.INFO: name_and_email2  
[2024-01-30 05:49:29] production.INFO: {"get":"string"}  
[2024-01-30 05:49:29] production.INFO: ユーザー名とパスワードを連結して取得2  
```

↓検証2の状態に戻して`php artisan ide-helper:models --write --reset`を実行した場合
```
[2024-01-30 05:52:48] production.INFO: name_and_email2  
[2024-01-30 05:52:48] production.INFO: []  
[2024-01-30 05:52:48] production.INFO: ユーザー名とパスワードを連結して取得2  
```

- 見てわかる通り2つ目の`info($types)`の出力が空になっている
- プログラムと照らし合わせると`if ($types->has('get')) {}`の部分がfalseとなり結果として`@property-read $name_and_email2`のPHPDocが生成されなかった

## 備考
- Laravel8.xの頃のアクセサの書き方をすれば良いのでは？
  - それでもいいけど別途[Laravel用のrector定義(driftingly/rector-laravel)](https://github.com/driftingly/rector-laravel)を導入している場合に[MigrateToSimplifiedAttributeRector](https://github.com/driftingly/rector-laravel/blob/main/docs/rector_rules_overview.md#migratetosimplifiedattributerector)のルールを有効化できなくなってしまう問題が起きてしまう（有効化するとLaravel8.xの形式のアクセサの記述について整形対象として挙がってしまう）。
- その他、laravel-ide-helperについて正式のリリースではないmasterの状態でインストールするかはプロジェクトのポリシー次第かなーと思った