# laravel-ide-helper-use-memo2
PHP8.2 + Laravel 10.x + Postgresql 16.x の開発環境において、[laravel-ide-helper](https://github.com/barryvdh/laravel-ide-helper)を使ったモデルクラスへのPHPDocの補完を行う際に詰まった内容についてまとめる2

## 問題点
- Laravel10.xにおいて現形式のアクセサの記述を行った場合、PHPDocに期待する@property-readのプロパティが生えない
- Laravel8.xの頃のget〇〇Attributeの記述を行った場合、PHPDocに期待する@property-readのプロパティが生える

## 2種類のアクセサについて
まずモデルクラスに記述するアクセサについておさらい

---
以下はLaravel8.xの頃の記述
```
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
```
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
```
$ composer require --dev barryvdh/laravel-ide-helper:^2.13
```

このとき `php artisan ide-helper:models --write --reset` によってUserモデルのPHPDocを補完を行ったところ以下のようになった

```php:User.php
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
