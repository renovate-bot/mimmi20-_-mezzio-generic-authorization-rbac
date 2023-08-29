# mezzio-generic-authorization-rbac

[![Latest Stable Version](https://poser.pugx.org/mimmi20/mezzio-generic-authorization-rbac/v/stable?format=flat-square)](https://packagist.org/packages/mimmi20/mezzio-generic-authorization-rbac)
[![Latest Unstable Version](https://poser.pugx.org/mimmi20/mezzio-generic-authorization-rbac/v/unstable?format=flat-square)](https://packagist.org/packages/mimmi20/mezzio-generic-authorization-rbac)
[![License](https://poser.pugx.org/mimmi20/mezzio-generic-authorization-rbac/license?format=flat-square)](https://packagist.org/packages/mimmi20/mezzio-generic-authorization-rbac)

## Code Status

[![codecov](https://codecov.io/gh/mimmi20/mezzio-generic-authorization-rbac/branch/master/graph/badge.svg)](https://codecov.io/gh/mimmi20/mezzio-generic-authorization-rbac)
[![Average time to resolve an issue](https://isitmaintained.com/badge/resolution/mimmi20/mezzio-generic-authorization-rbac.svg)](https://isitmaintained.com/project/mimmi20/mezzio-generic-authorization-rbac "Average time to resolve an issue")
[![Percentage of issues still open](https://isitmaintained.com/badge/open/mimmi20/mezzio-generic-authorization-rbac.svg)](https://isitmaintained.com/project/mimmi20/mezzio-generic-authorization-rbac "Percentage of issues still open")

This library provides a laminas-rbac adapter for mezzio-generic-authorization.

## Installation

You can install the mezzio-generic-authorization-rbac library with
[Composer](https://getcomposer.org):

```shell
composer require mimmi20/mezzio-generic-authorization-rbac
```

# Introduction

This component provides [Role-Based Access Control](https://en.wikipedia.org/wiki/Role-based_access_control)
(RBAC) authorization abstraction for the [mezzio-generic-authorization](https://github.com/mimmi20/mezzio-generic-authorization)
library.

RBAC is based on the idea of **roles**. In a web application, users have an
**identity** (e.g. username, email, etc). Each identified user then has one or
more roles (e.g. admin, editor, guest). Each role has a **permission** to
perform one or more actions (e.g. access an URL, execute specific web API
calls).

In a typical RBAC system:

- An **identity** has one or more roles.
- A **role** requests access to a permission.
- A **permission** is given to a role.

Thus, RBAC has the following model:

- Many-to-many relationship between identities and roles.
- Many-to-many relationship between roles and permissions.
- Roles can have a parent role.

The first requirement for an RBAC system is **identities**. In our scenario, the
users are generated by an authentication system, provided by
[mezzio-authentication](https://github.com/mezzio/mezzio-authentication).
That library provides a PSR-7 request attribute named
`Mezzio\Authentication\UserInterface` when a user is authenticated.
The RBAC system uses this instance to get information about the user's identity.

## Configure an RBAC system

You can configure your RBAC using a configuration file, as follows:

```php
// config/autoload/authorization.local.php
return [
    // ...
    'mezzio-authorization-rbac' => [
        'roles' => [
            'administrator' => [],
            'editor'        => ['administrator'],
            'contributor'   => ['editor'],
        ],
        'permissions' => [
            'contributor' => [
                'admin.dashboard',
                'admin.posts',
            ],
            'editor' => [
                'admin.publish',
            ],
            'administrator' => [
                'admin.settings',
            ],
        ],
    ],
];
```

In the above example, we designed an RBAC system with 3 roles: `administator`,
`editor`, and `contributor`. We defined a hierarchy of roles as follows:

- `administrator` has no parent role.
- `editor` has `administrator` as a parent. That means `administrator` inherits
  the permissions of the `editor`.
- `contributor` has `editor` as a parent. That means `editor` inherits the
  permissions of `contributor`, and following the chain, `administator` inherits
  the permissions of `contributor`.

For each role, we specified an array of permissions. As you can notice, a
permission is just a string; it can represent anything. In our implementation,
this string represents a route name.  That means the `contributor` role can
access the routes `admin.dashboard` and `admin.posts` but cannot access the
routes `admin.publish` (assigned to `editor` role) and `admin.settings`
(assigned to `administrator`).

If you want to change the authorization logic for each permission, you can write
your own `Mimmi20\Mezzio\GenericAuthorization\AuthorizationInterface` implementation.
That interface defines the following method:

```php
public function isGranted(string $role, string $resource, ?string $privilege = null, ?\Psr\Http\Message\ServerRequestInterface\ServerRequestInterface $request = null): bool;
```

where `$role` is the role, `$resource` is the resource, `$privilege` is an privilege and `$request` is the PSR-7 HTTP request to authorize.

> This library uses the [laminas/laminas-permissions-rbac](https://docs.laminas.dev/laminas-permissions-rbac/)
> library to implement the RBAC system. Privileges are not supported in this RBAC implementation. If you want to know more about the usage
> of this library, read the blog post 
> [Manage permissions with laminas-permissions-rbac](https://framework.zend.com/blog/2017-04-27-zend-permissions-rbac.html).

# Dynamic Assertion

In some cases you will need to authorize a role based on a specific HTTP request.
For instance, imagine that you have an "editor" role that can add/update/delete
a page in a Content Management System (CMS). We want to prevent an "editor" from
modifying pages they have not created.

These types of authorization are called [dynamic assertions](https://docs.laminas.dev/laminas-permissions-rbac/examples/#dynamic-assertions)
and are implemented via the `Laminas\Permissions\Rbac\AssertionInterface` of
[laminas-permissions-rbac](https://github.com/laminas/laminas-permissions-rbac).

In order to use it, this package provides `LaminasRbacAssertionInterface`,
which extends `Laminas\Permissions\Rbac\AssertionInterface`:

```php
namespace Mezzio\Authorization\Rbac;

use Psr\Http\Message\ServerRequestInterface;
use Laminas\Permissions\Rbac\AssertionInterface;

interface LaminasRbacAssertionInterface extends AssertionInterface
{
    public function setRequest(ServerRequestInterface $request) : void;
}
```

The `Laminas\Permissions\Rbac\AssertionInterface` defines the following:

```php
namespace Laminas\Permissions\Rbac;

interface AssertionInterface
{
    public function assert(Rbac $rbac, RoleInterface $role, string $permission) : bool;
}
```

Going back to our use case, we can build a class to manage the "editor"
authorization requirements, as follows:

```php
use Mimmi20\Mezzio\GenericAuthorization\Rbac\LaminasRbacAssertionInterface;
use App\Service\Article;
use Laminas\Permissions\Rbac\Rbac;
use Laminas\Permissions\Rbac\RoleInterface;
use Psr\Http\Message\ServerRequestInterface;

class EditorAuth implements LaminasRbacAssertionInterface
{
    public function __construct(Article $article)
    {
        $this->article = $article;
    }

    public function setRequest(ServerRequestInterface $request): void
    {
        $this->request = $request;
    }

    public function assert(Rbac $rbac, RoleInterface $role, string $permission): bool
    {
        $user = $this->request->getAttribute(UserInterface::class, false);
        return $this->article->isUserOwner($user->getIdentity(), $this->request);
    }
}
```

Where `Article` is a class that checks if the identified user is the owner of
the article referenced in the HTTP request.

If you manage articles using a SQL database, the implementation of
`isUserOwner()` might look like the following:

```php
public function isUserOwner(string $identity, ServerRequestInterface $request): bool
{
    // get the article {article_id} attribute specified in the route
    $url = $request->getAttribute('article_id', false);
    if (! $url) {
        return false;
    }
    $sth = $this->pdo->prepare(
        'SELECT * FROM article WHERE url = :url AND owner = :identity'
    );
    $sth->bindParam(':url', $url);
    $sth->bindParam(':identity', $identity);
    if (! $sth->execute()) {
        return false;
    }
    $row = $sth->fetch();
    return ! empty($row);
}
```

To pass the `Article` dependency to your assertion, you can use a Factory class
that generates the `EditorAuth` class instance, as follows:

```php
use App\Service\Article;

class EditorAuthFactory
{
    public function __invoke(ContainerInterface $container) : EditorAuth
    {
        return new EditorAuth(
            $container->get(Article::class)
        );
    }
}
```

And configure the service container to use `EditorAuthFactory` to point to
`EditorAuth`, using the following configuration:

```php
return [    
    'dependencies' => [
        'factories' => [
            // ...
            EditorAuth::class => EditorAuthFactory::class
        ]
    ]
];
```


## License

This package is licensed using the MIT License.

Please have a look at [`LICENSE.md`](LICENSE.md).
