# Laravel AWS Cognito Auth

A simple authentication package for Laravel 5 for authenticating users in Amazon Cognito User Pools.

This is package works with Laravel's native authentication system and allows the authentication of users that are already registered in Amazon Cognito User Pools. It does not provide functionality for user management, i.e., registering user's into User Pools, password resets, etc.


### Contents

- [Installation and Setup](#installation-and-setup)
    - [Install](#install)
    - [Configure](#configure)
    - [Users Table](#users-table)
- [Usage](#usage)
    - [Authenticating](#authenticating)
    - [Handling Failed Authentication](#handling-failed-authentication)
        - [Methods](#methods)
            - [No Error Handling](#no-error-handling)
            - [Throw Exception](#throw-exception)
            - [Return Attempt Instance](#return-attempt-instance)
            - [Using a Closure](#using-a-closure)
            - [Using a Custom Class](#using-a-custom-class)
        - [Default Error Handler](#default-error-handler)
        - [About AuthAttemptException](#about-authattemptexception)

## Installation and Setup

This package makes use of the  [aws-sdk-php-laravel](https://github.com/aws/aws-sdk-php-laravel) package. As well as setting up and configuring this package you'll also need to configure [aws-sdk-php-laravel](https://github.com/aws/aws-sdk-php-laravel) for the authentication to work. Instructions on how to do this are below. If you've already installed, set up and configured [aws-sdk-php-laravel](https://github.com/aws/aws-sdk-php-laravel) you can skip the parts where it's mentioned below.

### Install

Add `pallant/laravel-aws-cognito-auth` to `composer.json` and run `composer update` to pull down the latest version:

```
"pallant/laravel-aws-cognito-auth": "~1.0"
```

Or use `composer require`:

```
composer require pallant/laravel-aws-cognito-auth
```

Add the service provider and the [aws-sdk-php-laravel](https://github.com/aws/aws-sdk-php-laravel) service provider to the `providers` array in `config/app.php`.

```php
'providers' => [
    ...
    Aws\Laravel\AwsServiceProvider::class,
    Pallant\LaravelAwsCognitoAuth\ServiceProvider::class,
    ...
]
````

Open `app/Http/Kernel.php` and replace the default `\Illuminate\Session\Middleware\AuthenticateSession::class` middleware with `\Pallant\LaravelAwsCognitoAuth\AuthenticateSession::class`.

```php
protected $middlewareGroups = [
    'web' => [
        ...
        \Pallant\LaravelAwsCognitoAuth\AuthenticateSession::class,
        ...
    ],
];
```

Publish the config file as well as the [aws-sdk-php-laravel](https://github.com/aws/aws-sdk-php-laravel) config file to your `config` directory by running:

```
php artisan vendor:publish --provider="Pallant\LaravelAwsCognitoAuth\ServiceProvider"

php artisan vendor:publish --provider="Aws\Laravel\AwsServiceProvider"
```

### Configure

Open `config/auth.php` and set your default guard's driver to `aws-cognito`. Out of the box the default guard is `web` so your `config/auth.php` would look something like:

```php
'defaults' => [
    'guard' => 'web',
    'passwords' => 'users',
],

...

'guards' => [
    'web' => [
        'driver' => 'aws-cognito',
        'provider' => 'users',
    ],
]

```

Open `config/aws-cognito-auth.php` and add your AWS Cognito User Pool's id, and User Pool App's `client-id`.

```php
'pool-id' => '<xxx-xxxxx>',

...

'apps' => [
    'default' => [
        'client-id' => '<xxxxxxxxxx>',
        'refresh-token-expiration' => 30,
    ],
]
```

When creating an App for your User Pool the default Refresh Token Expiration time is 30 days. If you've set a different expiration time for your App then make sure you update the `refresh-token-expiration` value in the config file accordingly.

```php
'apps' => [
    'default' => [
        'client-id' => '<xxxxxxxxxx>',
        'refresh-token-expiration' => <num-of-days>,
    ],
]
```

Open the `config/aws.php` file and set the `region` value to whatever region your User Pool is in. The default `config/aws.php` file that is created when using the `php artisan vendor:publish --provider="Aws\Laravel\AwsServiceProvider"` command doesn't include the IAM credential properties so you'll need to add them manually. Add the following to the `config/aws.php` file where `key` is an IAM user Access Key Id and `secret` is the corresponding Secret key:

```php
'credentials' => [
    'key' => <xxxxxxxxxx>,
    'secret' => <xxxxxxxxxx>,
]
```

Your final `config/aws.php` should look something like this:

```php
'credentials' => [
    'key' => <xxxxxxxxxx>,
    'secret' => <xxxxxxxxxx>,
],
'region' => <xx-xxxx-x>,
'version' => 'latest',
'ua_append' => [
    'L5MOD/' . AwsServiceProvider::VERSION,
],
```

### Users Table

Cognito is not treated as a "database of users", and is only used to authorise the users. A request to Cognito is not made unless the user already exists in the app's database. This means you'll still need a `users` table populated with the users that you want to authenticate. At a minimum this table will need an `id`, `email` and `remember_token` field.

In Cognito every user has a `username`. When authenticating with Cognito this package will need one of the user's attributes to match the user's Congnito username. By default it uses the user's `email` attribute.

If you want to use a different attribute to store the user's Cognito username then you can do so by first adding a new field to your `users` table, for example `cognito_username`, and then setting the `username-attribute` in the `config/aws-cognito-auth.php` file to be the name of that field.

## Usage

Once installed and configured authentication works the same as it doesn natively in Laravel. See Laravel's [documentation](https://laravel.com/docs/5.4/authentication) for full details.

### Authenticating

**Authenticate:**

```php
Auth::attempt([
    'email' => 'xxxxx@xxxxx.xx',
    'password' => 'xxxxxxxxxx',
]);
```

**Authenticate and remember:**

```php
Auth::attempt([
    'email' => 'xxxxx@xxxxx.xx',
    'password' => 'xxxxxxxxxx',
], true);
```

**Get the authenticated user:**

```php
Auth::user();
```

**Logout:**

```php
Auth::logout();
```

As well as the default functionality some extra methods are made available for accessing the user's Cognito access token, id token, etc:

```php
Auth::getCognitoAccessToken();
```

```php
Auth::getCognitoIdToken();
```

```php
Auth::getCognitoRefreshToken();
```

```php
Auth::getCognitoTokensExpiryTime();
```

```php
Auth::getCognitoRefreshTokenExpiryTime();
```

### Handling Failed Authentication

AWS Cognito may fail to authenticate for a number of reasons, from simply entering the wrong credentials, or because additional checks or actions are required before the user can be successfully authenticated.

So that you can deal with failed attempts appropriately several options are available to you within the package that dictate how failed attempts should be handled.

#### Methods

You can specify how failed attempts should be handled by passing an additional `$errorHandler` argument when calling the `Auth::attempt()` and `Auth::validate()` methods.

```php
Auth::attempt(array $credentials, [bool $remember], [$errorHandler]);

Auth::validate(array $credentials, [$errorHandler]);
```

##### No Error Handling

If an `$errorHandler` isn't passed then all failed authentication attempts will be handled and suppressed internally, and both the `Auth::attempt()` and `Auth::validate()` methods will simply return `true` or `false` as to whether the authentication attempt was successful.

##### Throw Exception

To have the `Auth::attempt()` and `Auth::validate()` methods throw an exception pass `AWS_COGNITO_AUTH_THROW_EXCEPTION` as the `$errorHandler` argument.

```php
Auth::attempt([
    'email' => 'xxxxx@xxxxx.xx',
    'password' => 'xxxxxxxxxx',
], false, AWS_COGNITO_AUTH_THROW_EXCEPTION);

Auth::validate([
    'email' => 'xxxxx@xxxxx.xx',
    'password' => 'xxxxxxxxxx',
], AWS_COGNITO_AUTH_THROW_EXCEPTION);
```

If the authentication fails then a `\Pallant\LaravelAwsCognitoAuth\AuthAttemptException` exception will be thrown, which can be used to access the underlying error by calling the exception's `getResponse()` method. [About AuthAttemptException](#about-authattemptexception).

```php
try {
    Auth::attempt([
        'email' => 'xxxxx@xxxxx.xx',
        'password' => 'xxxxxxxxxx',
    ], false, AWS_COGNITO_AUTH_THROW_EXCEPTION);
} catch (\Pallant\LaravelAwsCognitoAuth\AuthAttemptException $e) {
    $response = $e->getResponse();
    // Handle error...
}
```

##### Return Attempt Instance

To have the `Auth::attempt()` and `Auth::validate()` methods return an attempt object pass `AWS_COGNITO_AUTH_RETURN_ATTEMPT` as the `$errorHandler` argument.

```php
Auth::attempt([
    'email' => 'xxxxx@xxxxx.xx',
    'password' => 'xxxxxxxxxx',
], false, AWS_COGNITO_AUTH_RETURN_ATTEMPT);

Auth::validate([
    'email' => 'xxxxx@xxxxx.xx',
    'password' => 'xxxxxxxxxx',
], AWS_COGNITO_AUTH_RETURN_ATTEMPT);
```

When using `AWS_COGNITO_AUTH_RETURN_ATTEMPT` both methods will return an instance of `\Pallant\LaravelAwsCognitoAuth\AuthAttempt`, which can be used to check if the authentication attempt was successful or not.

```php
$attempt = Auth::attempt([
    'email' => 'xxxxx@xxxxx.xx',
    'password' => 'xxxxxxxxxx',
], false, AWS_COGNITO_AUTH_RETURN_ATTEMPT);

if ($attempt->successful()) {
    // Do something...
} else {
    $response = $attempt->getResponse();
    // Handle error...
}
```

For unsuccessful authentication attempts the attempt instance's `getResponse()` method can be used to access the underlying error. This method will return an array of data that will contain different values depending on the reason for the failed attempt.

In events where the AWS Cognito API has thrown an exception, such as when invalid credentials are used, the array that is returned will contain the original exception.

```php
[
    'exception' => CognitoIdentityProviderException {...},
]
```

In events where the AWS Cognito API has failed to authenticate for some other reason, for example because a challenge must be passed, then the array that is returned will contain the details of the error.

```php
[
    'ChallengeName' => 'NEW_PASSWORD_REQUIRED',
    'Session' => '...',
    'ChallengeParameters' => [...],
]
```

##### Using a Closure

To handle failed authentication attempts with a closure pass one as the `Auth::attempt()` and `Auth::validate()` methods' `$errorHandler` argument.

```php
Auth::attempt([
    'email' => 'xxxxx@xxxxx.xx',
    'password' => 'xxxxxxxxxx',
], false, function (\Pallant\LaravelAwsCognitoAuth\AuthAttemptException $e) {
    $response = $e->getResponse();
    // Handle error...
});

Auth::validate([
    'email' => 'xxxxx@xxxxx.xx',
    'password' => 'xxxxxxxxxx',
], function (\Pallant\LaravelAwsCognitoAuth\AuthAttemptException $e) {
    $response = $e->getResponse();
    // Handle error...
};
```

If the authentication fails then the closure will be run and will be passed an `\Pallant\LaravelAwsCognitoAuth\AuthAttemptException` instance, which can be used to access the underlying error by calling the exception's `getResponse()` method. [About AuthAttemptException](#about-authattemptexception).


##### Using a Custom Class

To handle failed authentication attempts with a custom class pass the classes name as the `Auth::attempt()` and `Auth::validate()` methods' `$errorHandler` argument.

```php
Auth::attempt([
    'email' => 'xxxxx@xxxxx.xx',
    'password' => 'xxxxxxxxxx',
], false, \App\MyCustomErrorHandler::class);

Auth::validate([
    'email' => 'xxxxx@xxxxx.xx',
    'password' => 'xxxxxxxxxx',
], \App\MyCustomErrorHandler::class);
```

The error handler class should have a `handle()` method, which will be called when an authentication attempt fails. The `handle()` method will be passed an `\Pallant\LaravelAwsCognitoAuth\AuthAttemptException` instance, which can be used to access the underlying error by calling the exception's `getResponse()` method. [About AuthAttemptException](#about-authattemptexception).

```php
<?php

namespace App;

use Pallant\LaravelAwsCognitoAuth\AuthAttemptException;

class MyCustomErrorHandler
{
    public function handle(AuthAttemptException $e)
    {
        $response = $e->getResponse();
        // Handle error...
    }
}

```

#### Default Error Handler

As well defining the error handler in line, you can also define a default error handler in the `config/aws-cognito-auth.php` file. The same error handling methods are available as detailed above. When using `AWS_COGNITO_AUTH_THROW_EXCEPTION` or `AWS_COGNITO_AUTH_RETURN_ATTEMPT` set the value as a string, do not use the constant.

**Throw Exception:**

```php
'errors' => [
    'handler' => 'AWS_COGNITO_AUTH_THROW_EXCEPTION',
],
```

**Return Attempt:**

```php
'errors' => [
    'handler' => 'AWS_COGNITO_AUTH_RETURN_ATTEMPT',
],
```

**Use a Closure:**

```php
'errors' => [
    'handler' => function (\Pallant\LaravelAwsCognitoAuth\AuthAttemptException $e) {
        $e->getResponse();
        // Do something...
    },
],
```

**Use a Custom Class:**

```php
'errors' => [
    'handler' => \App\MyCustomErrorHandler::class,
],
```

#### About AuthAttemptException

An `\Pallant\LaravelAwsCognitoAuth\AuthAttemptException` exception will be thrown when using the `AWS_COGNITO_AUTH_THROW_EXCEPTION` error handler, or will be passed as an argument to a closure when using the `Clousre` method of error handling.

The exception's `getResponse()` method will return an array of data that will contain different values depending on the reason for the failed attempt.

In events where the AWS Cognito API has thrown an exception, such as when invalid credentials are used, the array that is returned will contain the original exception.

```php
[
    'exception' => CognitoIdentityProviderException {...},
]
```

In events where the AWS Cognito API has failed to authenticate for some other reason, for example because a challenge must be passed, the array that is returned will contain the details of the error.

```php
[
    'ChallengeName' => 'NEW_PASSWORD_REQUIRED',
    'Session' => '...',
    'ChallengeParameters' => [...],
]
```
