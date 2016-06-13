# Introduction
The Kirby JSON API plugin is a fairly simple layer on top of the existing Kirby infrastructure that provides a language-aware (read-only) JSON API to access the content tree from JavaScript and other external clients. It also provides some basic functionality for developers to easily add their own JSON-based APIs.

## Built-in API
> **Note**: The built-in API is disabled by default and has to be enabled first in the Kirby configuration as described below.

### Configuring the Built-In API

#### Enabling the API
To enable the built-in API, add the following to your Kirby configuration. The built-in API is disabled by default as a security measure. You should only enable it if you actually need it and if you do, you should make sure to check out the remaining security configuration options.
```php
c::set('jsonapi.built-in.enabled', true);
```
#### Authentication and Authorization
Once enabled, the built-in API is only available to the logged-in Kirby admin - again, primarily for security.

To make the API available to all logged-in users, you can set the configuration as follows:
```php
c::set('jsonapi.built-in.auth', function () {
	return Lar\JsonApi\JsonApiAuth::isLoggedIn();
});
```

To completely disable the authentication and make the API available to anybody, set the configuration like this:
```php
c::set('jsonapi.built-in.auth', false);
```

There are plenty more authentication options available (those are described below) and you can also implement your own. Be aware that the _configuration_ option `jsonapi.built-in.auth` expects an _authentication provider_ function - that is a function that returns the actual authentication function. This is to work around the bootstrapping issue where the plugin providing the authentication function isn't actually loaded yet. The _provider_ function is invoked when the built-in API is registered, at which point all the functionality of the JSON API plugin are available.

#### API Path prefix
All registerd API controllers are made available under **one** path prefix that defaults to `api`, so all registerd URL patterns are automatically prefixed with this value. You can of course change this prefix in the configuration:

```php
c::set('jsonapi.prefix', 'myapi');
```

All the examples below assume the default prefix `api`.

### Built-In API Features

#### Pages, Nodes and Trees
The primary API comes in three flavors:
* `[prefix]/page/[id]`: This will return all the page fields of a given page with the addition to the default meta fields described below.
* `[prefix]/node/[id]`: This works just like `page`, but adds all the files associated with a page as well as the `id`s of all its children.
* `[prefix]/tree/[id]`: Just like `node`, except that instead of only returning child `id`s, this endpoint _recursively_ includes all of the child pages.

All of these endpoints take the same single argument that is the `id` or a page. If for example you want to get the `page` with the ID `path/to/page`, then you can call the API with the path `/api/page/path/to/page`.

##### Examples
Example of the result returned by a call to the `page` API with `/api/page/projects/project-a`
```json
{
	"id":"projects\/project-a",
	"url":"http:\/\/mydomain.com\/projects\/project-a",
	"uid":"project-a",
	"title":"Project A",
	"text":"Lorem ipsum dolor sit amet"
}
```

Example of the result returned by a call to the `node` API with `/api/node/projects/project-a`
```json
{
	"id":"projects\/project-a",
	"url":"http:\/\/mydomain.com\/projects\/project-a",
	"uid":"project-a",
	"title":"Project A",
	"text":"Lorem ipsum dolor sit amet",
	"files":[
		{
			"url":"http:\/\/mydomain.com\/content\/2-projects\/1-project-a\/forest.jpg",
			"name":"forest",
			"extension":"jpg",
			"size":280900,
			"niceSize":"274.32 kB",
			"mime":"image\/jpeg",
			"type":"image"
		},
		{
			"url":"http:\/\/mydomain.com\/content\/2-projects\/1-project-a\/green.jpg",
			"name":"green",
			"extension":"jpg",
			"size":306670,
			"niceSize":"299.48 kB",
			"mime":"image\/jpeg",
			"type":"image"
		}
	],
	"children":["projects\/project-a\/sub"]
}
```

Example of the result returned by a call to the `tree` API with `/api/page/projects`
```json
{
	"id": "projects",
	"url": "http:\/\/mydomain.com\/projects",
	"uid": "projects",
	"title": "Projects",
	"text": "",
	"files": [],
	"children": [
		{
			"id": "projects\/project-a",
			...
		},
		{
			"id": "projects\/project-b",
			...
		},
		{
			"id": "projects\/project-c",
			...
		}
	]
}
```

##### Meta fields
These fields are automatically added to the response of every page.
* `id`: see https://getkirby.com/docs/cheatsheet/page/id
* `url`: see https://getkirby.com/docs/cheatsheet/page/url
* `uid`: see https://getkirby.com/docs/cheatsheet/page/uid

#### Child IDs, Children and Files
Every addition to the basic `page` information is also available as a separate API endpoint.

* `[prefix]/child-ids/[id]`: Returns an array of child IDs of the given page.
* `[prefix]/children/[id]`: Returns an array of child pages of the given page including all of their fields (non-recursive).
* `[prefix]/files/[id]`: Returns an array of all files associated with the given page.

## Custom API Extensions
Getting started with a custom API is really quite straight forward. All you need is a Kirby plugin (that can be an existing plugin or a new one) where you can create a file called `jsonapi.extension.php`. The JSON API plugin looks for files with that name during its initialization and loads them automatically. In this extension file, you can now simply declare your API like this

```php
<?php

jsonapi()->register([
	// api/custom/_name_/_value_
	[
		'method' => 'GET',
		'pattern' => "custom/(:any)/(:num)",
		'action' => function ($any, $num) {
			return ['msg' => "got $any with value $num"];
		},
	],
]);
```

That's it. You've got yourself an API which you can call with `/api/custom/me/42`.

### Registering routes
The global `jsonapi()` function returns an instance of the API manager whose `register($actions)` method takes an array of API route definitions.

### Routing
The JSON API is based on Kirby's [routing mechanism](https://getkirby.com/docs/developer-guide/advanced/routing) and adds some more options and a bit of framework functionality to simplify working with JSON.

The route options `method`, `pattern` and `action` are essentially taken as-is from the Kirby routing system, so please check [that documentation](https://getkirby.com/docs/developer-guide/advanced/routing) for more details.

### Actions
An action can be any [PHP callable](http://php.net/manual/en/language.types.callable.php) just like in any other Kirby route, but additionally, you can also use the _controller_ syntax in your route definitions.

```php
[
	// ...
	'controller' => 'MyController',
	'action' => 'myAction',
	// ...
]
```

The _controller_ class will be instantiated before the action is invoked. The controller must therefore have a parameterless constructor. Once instantiated, the action is invoked on the controller.

Normally, your custom API actions will return a (quite possibly nested) PHP array or PHP scalar which will be converted to a JSON response automatically. The result of an API action can also be any valid `KirbyResponse` object in which case Kirby's default response handling kicks in. As a middle ground, you can also return any object that implements the `Lar\JsonApi\IJsonObject` interface which gives you full control over how your objects will be serialized to JSON.

If your API is dealing with Kirby page objects, you can also use the helper objects and utilities that come with the JSON API plugin to craft your response. See the section on working with pages below.

### Authentication
The `auth` option of an API route definition ensures that unauthorized requests are handled with an HTTP 401 response. If set, the `auth` option has to be a PHP callable that explicitly returns `true` if the request has been authorized or `false` if the request should be blocked. The authorization function receives the same arguments as the controller.

The JSON API plugin provides the following pre-defined authorization handlers (see `Lar\JsonApi\JsonApiAuth` for more details):
* `Lar\JsonApi\JsonApiAuth::isLoggedIn()`: returns an auth handler that returns `true` if and only if the user making the request is logged in.
* `Lar\JsonApi\JsonApiAuth::isAdmin()`: returns an auth handler that returns `true` if and only if the user making the request is logged in and has the `admin` role.
* `Lar\JsonApi\JsonApiAuth::isUserWithRole($role = null)`: returns an auth handler that returns `true` if and only if the user making the request is logged in and has the role that is provided as the function argument.

### Language

### Working with Pages
