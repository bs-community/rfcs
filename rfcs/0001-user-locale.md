- 仓库：blessing-skin-server
- 日期: 2020-06-29
- 作者：[Pig Fang](https://github.com/g-plane)
- RFC PR: [#0001](https://github.com/bs-community/rfcs/pull/1)
- 实现 PR：[blessing-skin-server#190](https://github.com/bs-community/blessing-skin-server/pull/190)

# 基于用户的语言设置

## 概要

Blessing Skin 的语言将基于每个用户（若用户已登录）而设置，而不是通过浏览器来判断。但基于浏览器的判断机制不会被取消，因为要考虑未登录的用户。

## 初衷

目前，Blessing Skin 的文本语言是通过这样流程来判断的：先检查请求中是否包含 `lang` 参数，如果有，则设置当前语言为 `lang` 参数中所指定的语言；如果没有，则检查 Cookie 中的 `locale` 对应的值，如果有则设置为 `locale` Cookie 中指定的语言；最后，通过 HTTP 请求头部的 `Accept-Language` 字段来判断。完整的代码可在 [DetectLanguagePrefer 中间件](https://github.com/bs-community/blessing-skin-server/blob/e018ced5d8f09f83740762b89a8d5fdcdfec63bd/app/Http/Middleware/DetectLanguagePrefer.php) 找到。

这样的机制能满足大多数场景的需要，但有缺陷：

- API 如 Yggdrasil API 中因为没有 Cookie 而且请求中不会（也不太可能）包含 `lang` 参数，因此往往会 fallback 到英文，这不利于用户体验。虽然在 Yggdrasil API 中我们通过提建议或发起 PR 的方式让启动器在发送 Yggdrasil API 请求中带有 `Accept-Language` 头部，但这是有限的，因为我们不可能让所有启动器都能实现这一点。

- 发送邮件或通知。由于无法知道目标用户所使用的语言，因此只能同时发送中文和英文。

而如果能记录用户使用的语言，那么上面的问题将可以被解决。

## 详细设计

### 数据库

为了能「记住」用户使用的语言，必须想办法存储用户的语言信息。目前可以通过在 users 表中添加一个 `locale` 字段来保存。

该 `locale` 字段为可空（nullable）的 `varchar` 字段，放置在 `nickname` 字段后。

设置可空是为了保证向后的兼容性。

### 记录用户使用的语言

`locale` 字段默认为空值，我们需要选择一个合适的时机来更新这个字段。

#### 用户未登录

因为用户没有登录，所有没有办法去操作 users 表，因此不用继续考虑接下的流程。

#### 用户已登录

Blessing Skin 的页面中提供了一个下拉菜单，该菜单用于更改页面 UI 文本所使用的语言。该菜单的功能是通过在当前 URL 中附加一个 `lang` 参数并刷新页面来实现更改语言的。我们可以在 `DetectLanguagePrefer` 中间件中判断当前请求中是否包含 `lang` 参数，如果有，则用这个 `lang` 参数的值去更新当前用户的 `locale` 字段的值。而且这个过程充分地体现了用户的意图：我希望更改语言设置。

但由于 `DetectLanguagePrefer` 中间件的执行时机比 Session 和 Auth 等中间件早，因此即使用户的确登录了，在该中间件执行 `auth()->user()` 得到的值一定为 `null`。

我们可以监听 `Illuminate\Auth\Events\Authenticated` 事件，无论用户刚刚登录还是已经登录，只要用户目前处于「已登录」状态，该事件都会被触发。在这个事件中，我们可以获取到当前用户的 `User` 模型实例，此时我们可以更新 `locale` 字段的值。

### 获取用户的语言并更新 Blessing Skin 运行时的语言

前面提到，`DetectLanguagePrefer` 中间件在运行的时候无法获取当前的认证相关信息，因此这一部分的流程只能在 `Illuminate\Auth\Events\Authenticated` 的事件监听器中进行。

不管用户有没有登录，先在 `DetectLanguagePrefer` 中间件进行：判断请求中的 `lang` 参数，然后从 HTTP 头部 `Accept-Language` 中获取。以此来通过 `app()->setLocale()` 进行语言设置。

若用户已登录，但其 `locale` 值为空，则无需进行继续的操作。同时因为 `DetectLanguagePrefer` 中间件已运行过，因此此时的语言设置处于合理的状态。

若用户已登录，并且其 `locale` 有具体的值，则应该以此 `locale` 为准。即，再次调用 `app()->setLocale()` 来覆盖之前设置过的值。

### 应用用户所设置的语言

这一部分是针对 API 和邮件、通知的发送的。关于 UI 部分，前面已提到过。

#### Blessing Skin API

这一部分无需作出改动，因为 Blessing Skin API 中只是没有 Cookie 和 Session 功能而已。目前 Blessing Skin API 使用 JWT 和 OAuth2 作为认证方法，而这些方法对应的库可能不会触发 `Illuminate\Auth\Events\Authenticated` 事件，具体可参见「相关内容」部分。

#### Yggdrasil API

Yggdrasil API 有自己的一套认证机制，因此需要结合实际的代码逻辑来进行语言设置。（即 `setLocale` 调用）

#### 邮件发送

目前 Blessing Skin 内有「忘记密码」和「邮箱验证」两个场景需要发送邮件。其中「忘记密码」时用户未登录，因此不需要进行改动；「邮箱验证」中用户已登录，并且目标用户（即要给谁发送邮件）与当前用户为同一个用户，也不需要进行改动。

对于「正版验证」插件中，角色属主更改时发送邮件，由于目标用户（角色被取走的用户）与当前用户（获得角色的用户）不是同一个用户，这意味着这两处的语言很有可能不同，因此需要根据目标用户的语言设置来发送邮件。

Laravel 的 `trans` 函数包含三个参数，其中第三个参数是 `$locale`，也就是可以指定与当前语言设置不同的语种。由于此参数默认为 `null`，因此即使目标用户的 `locale` 字段值为空也没有问题。

使用方法大概如下：

```php
trans('xxx', [], $user->locale);
```

#### 通知发送

通知发送的情况与前面的「正版验证」插件中发送「角色属主更改」邮件的情况类似，只不过当前的用户是管理员，因此可以采用同样的策略。

## 缺点

暂无。

## 备选方案

暂无。

## FAQ

- 为什么不直接通过用户当前的浏览器设置来更新 `locale` 字段的值？

  这是为了保证一定的向后兼容性。特别是，在皮肤站的代码被更新而数据库迁移没有被执行时，如果此时通过浏览器的设置来更新的话，将会导致数据库找不到字段而出现错误。

## 未能解决的问题

暂无。

## 相关内容

- [laravel/passport#398: Auth events not fired](https://github.com/laravel/passport/issues/398)
- [`trans` 函数签名](https://github.com/laravel/framework/blob/07ee3e820b34df5e422fb868886fd190880dfc7f/src/Illuminate/Foundation/helpers.php#L858)

## 其它

### 允许更改 fallback locale

我们可以考虑允许 fallback locale 被自定义：在 `.env` 文件中添加一条 `APP_FALLBACK_LOCALE` 配置项。

### 重构「临时性的语言设置存储」

这里主要是针对未登录的用户的。因为用户没有登录，所以不可能更改 users 表。但对于这一类用户，他们也有可能会切换语言。目前的做法是使用一项未加密的名为 `locale` 的 Cookie。

我们也许可以重构这一部分功能，使其通过 Session 来现实。这样也就可以将原来的 `App\Http\Middlewares\EncryptCookies` 中间件替换为 Laravel 自带的 `Illuminate\Cookie\Middleware\EncryptCookies` 中间件。