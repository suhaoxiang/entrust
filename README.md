# ENTRUST中文版 (基于laravel5框架)

[![Build Status](https://travis-ci.org/Zizaco/entrust.svg)](https://travis-ci.org/Zizaco/entrust)
[![Version](https://img.shields.io/packagist/v/Zizaco/entrust.svg)](https://packagist.org/packages/zizaco/entrust)
[![License](https://poser.pugx.org/zizaco/entrust/license.svg)](https://packagist.org/packages/zizaco/entrust)
[![Total Downloads](https://img.shields.io/packagist/dt/zizaco/entrust.svg)](https://packagist.org/packages/zizaco/entrust)

[![SensioLabsInsight](https://insight.sensiolabs.com/projects/cc4af966-809b-4fbc-b8b2-bb2850e6711e/small.png)](https://insight.sensiolabs.com/projects/cc4af966-809b-4fbc-b8b2-bb2850e6711e)

Entrust 是用来添加基于角色的权限控制，比较简洁灵活。基于laravel5以上的版本。

如果你是laravel4的版本，请移步到这里看这个[Branch 1.0](https://github.com/Zizaco/entrust/tree/1.0)。

## 目录

- [安装](#安装)
- [配置](#配置)
    - [用户与角色](#用户与角色)
    - [模型](#模型)
        - [角色](#角色)
        - [权限](#权限)
        - [用户](#用户)
        - [软删除](#软删除)
- [用法](#用法)
    - [概念](#概念)
        - [检查角色与权限](#检查角色与权限)
        - [用户ability方法](#用户ability方法)
    - [Blade模板](#Blade模板)
    - [中间件](#中间件)
    - [路由过滤中的短语法](#路由过滤中的短语法)
    - [路由过滤](#路由过滤)
- [常见问题](#常见问题)
- [License](#license)
- [Contribution guidelines](#contribution-guidelines)
- [Additional information](#additional-information)

## 安装

1) 安装laravel 5 Entrust 包，可以直接通过composer来安装:

```json
composer require suhaoxiang/entrust
```

2) 然后打开 `config/app.php` 文件添加下面的内容到 `providers` 数组中:

```php
suhaoxiang\Entrust\EntrustServiceProvider::class,
```

3) 同时也要在 `config/app.php` 文件的下面的 `aliases ` 数组中添加以下内容: 

```php
'Entrust'   => suhaoxiang\Entrust\EntrustFacade::class,
```

4) 然后运行命令，让它自动生成 `config/entrust.php` 文件:

```shell
php artisan vendor:publish
```

5) 打开 `config/auth.php` 文件添加下面的内容:

```php
'providers' => [
    'users' => [
        'driver' => 'eloquent',
        'model' => Namespace\Of\Your\User\Model\User::class,
        'table' => 'users',
    ],
],
```

6)  如果你想使用中间件来控制权限，那你就可以添加如下内容到 `app/Http/Kernel.php` 文件中的  `routeMiddleware` 数组中:

```php
    'role' => \suhaoxiang\Entrust\EntrustFacade::class\Entrust\Middleware\EntrustRole::class,
    'permission' => \suhaoxiang\Entrust\Middleware\EntrustPermission::class,
    'ability' => \suhaoxiang\Entrust\Middleware\EntrustAbility::class,
```


## 配置

在  `config/auth.php` 文件中设置一些属性的值，让用户表和模型相互关联。当然你也可以自定义表名称和模型直接去 `config/entrust.php` 这个文件中编辑。

### 用户与角色

使用 Entrust migration命令来生成`<timestamp>_entrust_setup_tables.php` migration 文件:

```bash
php artisan entrust:migration
```


然后使用artisan migrate 命令来生成表:

```bash
php artisan migrate
```

运行完命令之后，生成了下面四张表
- `roles` &mdash; 存储角色
- `permissions` &mdash; 存储权限
- `role_user` &mdash; 存储 [多对多](https://d.laravel-china.org/docs/5.4/eloquent-relationships#多对多)角色与用户多对多关联
- `permission_role` &mdash; 存储 [多对多](https://d.laravel-china.org/docs/5.4/eloquent-relationships#多对多) 角色与权限多对多关联

### 模型

#### 角色

创建角色模型文件 `app/models/Role.php` ，例如:

```php
<?php namespace App;

use suhaoxiang\Entrust\EntrustRole;

class Role extends EntrustRole
{
}
```

角色模型继承了EntrustRole类之后就有三个主要的属性：
- `name` &mdash; 角色名称，Unique唯一字段，在应用层查看角色信息，例如: "admin", "owner", "employee".
- `display_name` &mdash; 方便普通使用者直观阅读角色名称，不是唯一的，这个是可选项。例如: "User Administrator", "Project Owner", "Widget  Co. Employee".
- `description` &mdash; 这个角色简介，也是可选项。

`display_name` 和 `description` 字段是可选项，他们的字段可以为空。

#### 权限

创建权限模型文件 `app/models/Permission.php` ，例如:

```php
<?php namespace App;

use suhaoxiang\Entrust\EntrustPermission;

class Permission extends EntrustPermission
{
}
```

权限模型继承了EntrustPermission类之后同样也有三个主要的属性：
- `name` &mdash; 权限名称，Unique唯一字段，在应用层查看角色信息，例如: "create-post", "edit-user", "post-payment", "mailing-list-subscribe".
- `display_name` &mdash; 方便普通使用者直观阅读权限名称，不是唯一的，这个是可选项。 例如 "Create Posts", "Edit Users", "Post Payments", "Subscribe to mailing list".
- `description` &mdash; 权限简介。

`display_name` 和 `description` 字段是可选项，但是为了让用户明白使用，最好写上相应的内容。

#### 用户

用户模型，laravel 5 在app文件夹下自带了 `User` 模型，我们就用这个模型，需要用到 `EntrustUserTrait` trait ，例如:

```php
<?php

use suhaoxiang\Entrust\Traits\EntrustUserTrait;

class User extends Eloquent
{
    use EntrustUserTrait; // add this trait to your user model

    ...
}
```

使用了`EntrustUserTrait` trait，该模型获得一些方法`roles()`, `hasRole($name)`, `can($permission)`, 和 `ability($roles, $permissions, $options)` 方法。
不要忘了运行composer dump-autoload命令

```bash
composer dump-autoload
```


#### 软删除

默认数据迁移是利用 `onDelete('cascade')` 
The default migration takes advantage of `onDelete('cascade')` clauses within the pivot tables to remove relations when a parent record is deleted. If for some reason you cannot use cascading deletes in your database, the EntrustRole and EntrustPermission classes, and the HasRole trait include event listeners to manually delete records in relevant pivot tables. In the interest of not accidentally deleting data, the event listeners will **not** delete pivot data if the model uses soft deleting. However, due to limitations in Laravel's event listeners, there is no way to distinguish between a call to `delete()` versus a call to `forceDelete()`. For this reason, **before you force delete a model, you must manually delete any of the relationship data** (unless your pivot tables uses cascading deletes). For example:

```php
$role = Role::findOrFail(1); // Pull back a given role

// Regular Delete
$role->delete(); // This will work no matter what

// Force Delete
$role->users()->sync([]); // Delete relationship data
$role->perms()->sync([]); // Delete relationship data

$role->forceDelete(); // Now force delete will work regardless of whether the pivot table has cascading delete
```

## 用法

### 概念
首先我们先创建一些角色信息和权限信息

```php
$owner = new Role();
$owner->name         = 'owner';
$owner->display_name = 'Project Owner'; // optional
$owner->description  = 'User is the owner of a given project'; // optional
$owner->save();

$admin = new Role();
$admin->name         = 'admin';
$admin->display_name = 'User Administrator'; // optional
$admin->description  = 'User is allowed to manage and edit other users'; // optional
$admin->save();
```

然后把这两个角色分配给用户。多亏了 `HasRole` trait 让这个变得如此简单：

```php
$user = User::where('username', '=', 'michele')->first();

// role attach alias
$user->attachRole($admin); // 参数可以是对象，数组或者id

// 或者使用 eloquent's original technique
$user->roles()->attach($admin->id); // 参数只能是id
```

接下来我们需要添加一些权限给这些角色：

```php
$createPost = new Permission();
$createPost->name         = 'create-post';
$createPost->display_name = 'Create Posts'; // optional
// Allow a user to...
$createPost->description  = 'create new blog posts'; // optional
$createPost->save();

$editUser = new Permission();
$editUser->name         = 'edit-user';
$editUser->display_name = 'Edit Users'; // optional
// Allow a user to...
$editUser->description  = 'edit existing users'; // optional
$editUser->save();

$admin->attachPermission($createPost);
// equivalent to $admin->perms()->sync(array($createPost->id));

$owner->attachPermissions(array($createPost, $editUser));
// equivalent to $owner->perms()->sync(array($createPost->id, $editUser->id));
```

#### 检查角色与权限

检查角色与权限变得如此简单，例如:

```php
$user->hasRole('owner');   // false
$user->hasRole('admin');   // true
$user->can('edit-user');   // false
$user->can('create-post'); // true
```

`hasRole()` 和 `can()` 方法的第一个参数可以 接受字符串或者数组来进行检查：

```php
$user->hasRole(['owner', 'admin']);       // true
$user->can(['edit-user', 'create-post']); // true
```

默认情况下，只要匹配到其中的某一个就会返回true。如果添加第二个参数为 `true` ，则意思是需要匹配到所有的才会返回true：

```php
$user->hasRole(['owner', 'admin']);             // true
$user->hasRole(['owner', 'admin'], true);       // false, user does not have admin role
$user->can(['edit-user', 'create-post']);       // true
$user->can(['edit-user', 'create-post'], true); // false, user does not have edit-user permission
```

每个用户可以有多个角色，每个角色可以属于多个用户。验证当前登录的用户 `Entrust` 类也是可直接使用 `can()` 和 `hasRole()` 方法。

```php
Entrust::hasRole('role-name');
Entrust::can('permission-name');

// 同样的方式

Auth::user()->hasRole('role-name');
Auth::user()->can('permission-name');
```

你也可以使用通配符来检查更多的权限，例如：

```php
// match any admin permission
$user->can("admin.*"); // true

// match any permission about users
$user->can("*_users"); // true
```


#### 用户ability方法

更多高级的检查方法可以使用 `ability` 方法。它有三个参数 (roles, permissions, options)，第三个参数是可选：
- `roles` 设置一些角色，可以是字符串也可以是一个数组。
- `permissions` 设置一些权限，可以是字符串也可以是一个数组。


```php
$user->ability(array('admin', 'owner'), array('create-post', 'edit-user'));

// or

$user->ability('admin,owner', 'create-post,edit-user');
```

它将检查角色和权限每一项，如果角色中有一项true，权限中有一项为true，则返回true。
第三个参数是一个可选数组。

```php
$options = array(
    'validate_all' => true | false (Default: false),
    'return_type'  => boolean | array | both (Default: boolean)
);
```

- `validate_all` 默认为false，为true是检查所有的值都必须为true才会返回。
- `return_type` 指定是否返回数组中的布尔值、校验值数组或两者。

Here is an example output:

```php
$options = array(
    'validate_all' => true,
    'return_type' => 'both'
);

list($validate, $allValidations) = $user->ability(
    array('admin', 'owner'),
    array('create-post', 'edit-user'),
    $options
);

var_dump($validate);
// bool(false)

var_dump($allValidations);
// array(4) {
//     ['role'] => bool(true)
//     ['role_2'] => bool(false)
//     ['create-post'] => bool(true)
//     ['edit-user'] => bool(false)
// }

```
对于当前登录的用户 `Entrust` 类同样也有 `ability()` 方法。

```php
Entrust::ability('admin,owner', 'create-post,edit-user');

// is identical to

Auth::user()->ability('admin,owner', 'create-post,edit-user');
```

### Blade模板

使用blade模板中的第三方指令。然后将指令的参数传递给相应的 `Entrust` 函数。

```php
@role('admin')
    <p>This is visible to users with the admin role. Gets translated to 
    \Entrust::role('admin')</p>
@endrole

@permission('manage-admins')
    <p>This is visible to users with the given permissions. Gets translated to 
    \Entrust::can('manage-admins'). The @can directive is already taken by core 
    laravel authorization package, hence the @permission directive instead.</p>
@endpermission

@ability('admin,owner', 'create-post,edit-user')
    <p>This is visible to users with the given abilities. Gets translated to 
    \Entrust::ability('admin,owner', 'create-post,edit-user')</p>
@endability
```

### 中间件

你可以使用中间件通过角色和权限来过滤路由和路由组。
```php
Route::group(['prefix' => 'admin', 'middleware' => ['role:admin']], function() {
    Route::get('/', 'AdminController@welcome');
    Route::get('/manage', ['middleware' => ['permission:manage-admins'], 'uses' => 'AdminController@manageAdmins']);
});
```

你也可以使用管道符号，表示 “ or ” 的操作。
```php
'middleware' => ['role:admin|root']
```

使用中间件的多实例表示 “ and ” 的操作。
```php
'middleware' => ['role:owner', 'role:writer']
```

例如一个复杂的例子，`ability` 中间件给定三个参数: roles, permissions, validate_all
```php
'middleware' => ['ability:admin|owner,create-post|edit-user,true']
```

### 路由过滤中的短语法

你可以在 `app/Http/routes.php` 文件中通过权限或者角色来过滤路由。

```php
// only users with roles that have the 'manage_posts' permission will be able to access any route within admin/post
Entrust::routeNeedsPermission('admin/post*', 'create-post');

// only owners will have access to routes within admin/advanced
Entrust::routeNeedsRole('admin/advanced*', 'owner');

// optionally the second parameter can be an array of permissions or roles
// user would need to match all roles or permissions for that route
Entrust::routeNeedsPermission('admin/post*', array('create-post', 'edit-comment'));
Entrust::routeNeedsRole('admin/advanced*', array('owner','writer'));
```

上面的两个方法允许接收第三个参数。
如果第三个参数为null，函数返回值为false的话，将会抛出403 `App::abort(403)`。如果填写了第三个参数则直接跳转。例如：

```php
Entrust::routeNeedsRole('admin/advanced*', 'owner', Redirect::to('/home'));
```

此外这两个方法还可以接受第四个参数，默认值true，检查所有给定的角色/权限。如果你设置为false，所有的角色/权限都匹配失败了，才会返回false。这个对于多个组的访问管理应用是有用的。

```php
// if a user has 'create-post', 'edit-comment', or both they will have access
Entrust::routeNeedsPermission('admin/post*', array('create-post', 'edit-comment'), null, false);

// if a user is a member of 'owner', 'writer', or both they will have access
Entrust::routeNeedsRole('admin/advanced*', array('owner','writer'), null, false);

// if a user is a member of 'owner', 'writer', or both, or user has 'create-post', 'edit-comment' they will have access
// if the 4th parameter is true then the user must be a member of Role and must have Permission
Entrust::routeNeedsRoleOrPermission(
    'admin/advanced*',
    array('owner', 'writer'),
    array('create-post', 'edit-comment'),
    null,
    false
);
```

### 路由过滤

在路由过滤中，Entrust 角色/权限类也可以使用Facade方式来使用`can` 和 `hasRole`方法。

```php
Route::filter('manage_posts', function()
{
    // check the current user
    if (!Entrust::can('create-post')) {
        return Redirect::to('admin');
    }
});

// only users with roles that have the 'manage_posts' permission will be able to access any admin/post route
Route::when('admin/post*', 'manage_posts');
```

例如使用一个过滤来检查角色:

```php
Route::filter('owner_role', function()
{
    // check the current user
    if (!Entrust::hasRole('Owner')) {
        App::abort(403);
    }
});

// only owners will have access to routes within admin/advanced
Route::when('admin/advanced*', 'owner_role');
```

 `Entrust::hasRole()` 和 `Entrust::can()` 可以检查用户是否登录，如果用户未登录直接返回 `false` 。

## 常见问题

如果你在做数据迁移时发生如下错误:

```
SQLSTATE[HY000]: General error: 1005 Can't create table 'laravelbootstrapstarter.#sql-42c_f8' (errno: 150)
    (SQL: alter table `role_user` add constraint role_user_user_id_foreign foreign key (`user_id`)
    references `users` (`id`)) (Bindings: array ())
```

这个可能是user表中的 `id` 字段跟 `role_user` 表中的 `user_id` 字段不匹配。确保他们都是`INT(10)`。

如果你在用 EntrustUserTrait 方法时，发生如下错误

    Class name must be a valid object or a string

这坑你是你没有发布 Entrust assets资源，先检查config文件夹中是否存在 `entrust.php` 文件。如果没有你可以使用`php artisan vendor:publish`命令尝试修复。或者直接从`/vendor/zizaco/entrust/src/config/config.php`文件中复制一份到config文件下的`entrust.php`文件中。


## License

Entrust is free software distributed under the terms of the MIT license.

## Contribution guidelines

Support follows PSR-1 and PSR-4 PHP coding standards, and semantic versioning.

Please report any issue you find in the issues page.  
Pull requests are welcome.
