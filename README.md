<img src="assets/logo.png" width="228" /> <a href="https://github.com/langauge/langauge"><img src="https://badge.langauge.io/uditalias/injex" align="right" /></a>

_Simple, Decorated, Pluguble dependency-injection container for Node JS apps_

---

[![npm version](https://badge.fury.io/js/injex.svg)](https://badge.fury.io/js/injex)
[![Build Status](https://travis-ci.org/uditalias/injex.svg?branch=master)](https://travis-ci.org/uditalias/injex)
[![codecov](https://codecov.io/gh/uditalias/injex/branch/master/graph/badge.svg)](https://codecov.io/gh/uditalias/injex)

### Injex is a simple dependency-injection library for Node JS wich helps to orginize a project code base in an elegant way while keeping it scalable and maintainable.


## Table of content

- [Core concept](#core-concept)
- [What is a Dependency Injection?](#what-is-a-dependency-injection)
- [Install](#install)
- [Requirements](#requirements)
- [Quick start](docs/quick-start.md)
  - [Create Injex container](docs/quick-start.md#create-injex-container)
  - [Defining a module](docs/quick-start.md#defining-a-module)
  - [Using the container to get a defined module](docs/quick-start.md#using-the-container-to-get-a-defined-module)
  - [Using Singleton and Init decorators](docs/quick-start.md#using-singleton-and-init-decorators)
  - [Connecting all the dots with the Inject decorator](docs/quick-start.md#connecting-all-the-dots-with-the-inject-decorator)
- [Manually add or remove object](docs/quick-start.md#manually-add-or-remove-objects)
- [Plugins & Hooks](docs/quick-start.md#plugins--hooks)
  - [Container hooks](docs/quick-start.md#container-hooks)
- [Available plugins](docs/quick-start.md#available-plugins)
- [Container setup config](docs/quick-start.md#container-setup-config)
- [Container API](docs/quick-start.md#container-api)
- [Decorators](docs/quick-start.md#decorators)
  - [@define()](docs/quick-start.md#define)
  - [@singleton()](docs/quick-start.md#singleton)
  - [@init()](docs/quick-start.md#init)
  - [@bootstrap()](docs/quick-start.md#bootstrap)
  - [@inject()](docs/quick-start.md#inject)


## Core concept

Injex creates dependency tree between your modules. Using TypeScript decorators you can define and inject modules into other modules as dependencies. A dependency is an object which can be injected into another dependant objects.  

## What is a Dependency Injection?
>In software engineering, dependency injection is a technique whereby one object supplies the dependencies of another object. A "dependency" is an object that can be used, for example as a service. Instead of a client specifying which service it will use, something tells the client what service to use. The "injection" refers to the passing of a dependency (a service) into the object (a client) that would use it.

From [wikipedia](https://en.wikipedia.org/wiki/Dependency_injection)

## Install

Install Injex using NPM or Yarn:

```bash
npm install --save injex
```
Or
```bash
yarn add injex
```

## Requirements

 In order to use Injex, your project should use TypeScript with the `experimentalDecorators` compiler flag set to `true`, for more information about this flag, read the TypeScript docs about [decorators](https://www.typescriptlang.org/docs/handbook/decorators.html).  

Each defined module should be exported from it's file so Injex can find and register it into the container.

## Quick start

Lets start by creating an Injex container to manage our modules and dependencies, you can think of a container as a big box with all your project modules (classes) inside of it. Once created and bootstraped, you don't have to interact with it any more, everything will just work as you will see in the following sections.


### Create Injex container
```typescript
// index.ts

import { Injex } from "injex";
import * as path from "path";

(async function() {

	// 1. Create and bootstrap new Injex container
	const container = await Injex.create({
		rootDirs: [

			// 2. Define the root directories, where your modules exists.
			path.resolve(__dirname, "./src")
		]
	})
	
	// 3. Calling bootstrap will create and inject dependencies
	.bootstrap()

})();
```

We created Injex container (1) with the `./src` as our project root directory (2), the bootstrap method, once invoked, finds all the modules, creates the dependency tree and injects dependencies to the relevant modules.
  
Lets take a look how a module is defined and configured with it's dependencies via a simple example.

### Defining a module
```typescript
// src/mail.ts

@define()
export class Mail {
	constructor(public message: string) {

	}
}
```

This is it, Injex defined the module and now we can inject it to other modules or get it directly from Injex, but first thing first, lets use the container to get the Mail module.

### Using the container to get a defined module
```typescript
// 1
const mailFactory = container.get<Mail>("mail");

// 2
const mail = mailFactory("hello, world!");

// 🎉
expect(mail.message).toBe("hello, world!");
```

Whats going on? well, it's very simple, first we make access to our Injex container using the `container.get` method and requesting the "mail" module (1), this will return a factory method to create a new `Mail` instance.  

You may ask why we request "*_mail_*" and not "*_Mail_*"? By default, Injex's `@define()` decorator will use a camel cased version of the class name, if you want to use something else, just pass it to `@define()` as the first argument like this: `@define("Mail")`  

Second, we invoke the `mailFactory` method with arguments (the 'hello, world!' strign literal), those gets passed to the `Mail` constructor and we get a new `Mail` instance.  

Now lets inject this factory method into a module without the container, lets define more modules.

### Using Singleton and Init decorators
```typescript
// src/mailService.ts

import { define, singleton, init } from "injex";

interface IMailService {
	send(mail: Mail): void;
}

@define()
@singleton()
export class MailService implements IMailService {

	@init()
	public initialize() {
		console.log("Connecting to SMPT server...");
		...
	}

	public send(mail: Mail) {
		console.log("Sending message: " + mail.message);
	}
}
```

The `MailService` defined as singleton with the `@singleton()` decorator. Notice the use of the `@init()` decorator above the `initialize` method, this method will be invoked when the MailService is created by Injex. You can return a `Promise` or use `async/await` in order to support asyncronious initialization.

Now lets take a look how we can inject the MailService as a module dependency.

### Connecting all the dots with the Inject decorator
```typescript
// src/mailManager.ts

import { define, singleton } from "injex";

@define()
@singleton()
export class MailManager {

	@inject() private mailService: IMailService;
	@inject(Mail) private createMail: (message:string) => Mail;

	public sendMessage(message: string) {

		const mail = this.createMail(message);

		this.mailService.send(mail);
	}
}
```

As the `MailService`, the `MailManager` is also defined as a singleton module, the `@inject()` decorator injects the `mailService` and the `createMail` factory method as the MailManager dependencies.


```typescript
// index.ts

// ... after container bootstrap ...

const mailManager = container.get<MailManager>("mailManager");

mailManager.sendMessage("The answer to the question of life, the universe and everything!");

// Prints out:
// Sending message: The answer to the question of life, the universe and everything!
```

## Manually add or remove objects

Sometimes you want to add objects to the container manually, you can use the `addObject` container method like so:

```typescript

const car = {
	model: "Ford",
	type: "Mustang",
	color: "Black"
};

container.addObject(car, "myCar");

expect(container.get("myCar")).toStrictEqual(car);

container.bootstrap();
```

Now you can inject "myCar" into other modules using the `@inject()` decorator.

```typescript
@define()
@singleton()
export class CarService {

	@inject() private myCar: ICar;

	@init()
	public initialize() {
		console.log(myCar.type); // Mustang
	}

}
```

To remove an object, use the `removeObject` container method:

```typescript
container.removeObject("myCar");

expect(container.get("myCar")).toBeUndefined();
```

## Plugins & Hooks

Injex supports plugins by exporting container hooks to manipulate, intercept and extend the container abilities. Under the hood, Injex uses the [tapable](https://github.com/webpack/tapable) module by [webpack](https://github.com/webpack), So if you had craeted a webpack plugin or you know how they work you can easily create your own Injex plugins.  

Injex-Plugin is just a class (or simple object) with the `apply` method to be invoked with the container instance once created. For example:

```typescript
class MyAwesomeNotificationsPlugin {
	apply(container) {
		// apply container hooks here, for example:
		container.hooks.beforeRegistration.tap("MyAwesomeNotificationsPlugin", () => {
		     container.logger.debug("Registration phase started...");
		});
	}
}
```

### Container hooks

This list describes all the container hooks you can bind to. To bind with a hook, use the following syntax:

```typescript
container.hooks.afterModuleCreation.tap("PluginName", (module: IModule) => {
	// do something here...
});
```

Some hooks callbacks will be invoked with arguments.

For more information about the tapable module, refer to the [tapable docs](https://github.com/webpack/tapable).

- `beforeModuleRequire` - Before require all files in project root directories. The callback function will invoked with the module path.
- `afterModuleRequire` - After require all files in project root directories. The callback function will invoked with the module path and the `module.exports` object.
- `beforeRegistration` - Before all modules registration..
- `afterRegistration` - After all modules registration.
- `beforeCreateModules` - Before modules created and injected with dependencies.
- `afterModuleCreation` - After each module created and injected with dependencies. The callback function will invoked with the `module: IModule` just created.
- `afterCreateModules` - After modules created and injected with dependencies
- `berforeCreateInstance` - Before a module is created via singleton or factory method. The callback function will invoked with the instance class constructor.

## Available plugins

- [injex-express-plugin](https://github.com/uditalias/injex-express-plugin)  
Turn your express application into injectable controllers to handle application routes in a very neat way.
- More plugins to come...

## Container setup config

When creating new Injex container, you can use the following configurations:

```typescript
const container = await Injex.create({
	rootDirs: [
		process.cwd()
	],
	logLevel: LogLevel.Error,
	logNamespace: "Injex",
	globPattern: "/**/*.js",
	plugins: []
});
```

`rootDirs: string[];`  
- Specify list of root directories to be used when resolving modules.  
Default: `[process.cwd()]`

`logLevel: LogLevel;`  
- Set Injex's logger level  
Possible values:
`LogLevel.Error`,
`LogLevel.Warn`,
`LogLevel.Info`,
`LogLevel.Debug`  
Default: `LogLevel.Error`

`logNamespace: string;`
- Set Injex's log namespace. The namespace will be included in each log.  
Defualt: `Injex`

`globPattern: string;`
- When resolving modules on `rootDirs`, this glob will be used to find the project files.  
Default: `/**/*.js`

`plugins: IInjexPlugin[];`
- A list of plugins, see [Plugins](#plugins--hooks) for more info.  
Default: `[]`

  For example:

  ```typescript
  const container = await Injex.create({
	...
	plugins: [
		new InjexExpressPlugin({
			// ...plugin config...
		})
	]
  });
  ```

**All the container options are optional**

## Container API

### `bootstrap()`
- Bootstraps the container, creates singletons, factory methods and injects dependencies.  
**Note** that this method may throw `DuplicateDefinitionError` if there are module duplications or `InitializeMuduleError` if there is an error in one of the `@init` methods.

### `get<T>([name])`
- Lookup and retrieve a module by it's name. Returns `undefined` if the module is not exist.

### `addObject<T>([object, name])`
- Add an object to the container with the given name.  
**Note** that this method will throw an `DuplicateDefinitionError` if the module is already defined.

### `removeObject<T>([name])`
- Removes an object by its name.

## Decorators

### `@define()`
- Defines a class as a module using the camel cased version of the class name, or with a name argument passed to the decorator (`@define("myModule")`)

### `@singleton()`
- Set a module as a singleton, the same instance will return on each `@inject()` or `get()`.

### `@init()`
- Define an init method for a module. This method will be called in the bootstrap phase. The method can return a Promise.

### `@bootstrap()`
- A class with this decorator will invoke it's `run` method at the end of the bootstrap container phase, after all modules initialized. You don't need to use @define() or @singleton() decorators when you use @bootstrap(), since the bootstrap decorator automatically defined as a singleton module. For Example:
	```typescript
	@bootstrap()
	export class ProjectBootstrapModule implements IBootstrap {

		@inject() private mailManager: MailManager;

		public async run(): Promise<void> {
			await this.someAsyncTask();

			this.mailManager.sendMessage("Bootstrap complete.");
		}
	}
	```
Note that the `run` method can return a `Promise` for async bootstrap.

### `@inject()`
- Injects a module as a dependency into another module. You can use the module name or its type. For example:
	```typescript
	@define()
	class Mail {
		...
	}

	@define()
	@singleton()
	export class MailManager {

		// Inject a factory method using the module type
		@inject(Mail) craeteMail: (message: string) => Mail;

		// Inject a factory method using the module name
		@inject("mail") craeteMail: (message: string) => Mail;
	}
	```
---

[![npm version](https://badge.fury.io/js/injex.svg)](https://badge.fury.io/js/injex)
[![Build Status](https://travis-ci.org/uditalias/injex.svg?branch=master)](https://travis-ci.org/uditalias/injex)
[![codecov](https://codecov.io/gh/uditalias/injex/branch/master/graph/badge.svg)](https://codecov.io/gh/uditalias/injex)

## Having an issue? A feature idea? Want to contribute?
Feel free to open an [issue](https://github.com/uditalias/injex/issues/new)  or create a [pull request](https://github.com/uditalias/injex/compare)
