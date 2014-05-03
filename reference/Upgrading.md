# Upgrading
Sails v0.10 comes with some big changes.  The sections below provide a high level overview of what's changed, major bug fixes, enhancements and new features, as well as a basic tutorial on how to upgrade your v0.9.x Sails app to v0.10.

> If you run across any upgrade issues that are not addressed here, please let us know and send a pull request to [this file on Github](https://github.com/balderdashy/sails-docs/edit/master/reference/Upgrading.md).  Thanks!
>
> ~Mike

========================================

### Contents

|     | Jump to...        |
|-----|-------------------------|
| I   | [Blueprints](#blueprints) |
| II  | [Policies](#policies) |
| III | [Associations](#associations) |
| IV  | [Pubsub](#pubsub) |
| V   | [Generators](#generators) |
| VI  | [Command-Line Tool](#command-line-tool) |
| VII | [Custom Server Responses](#custom-server-responses) |
| VIII| [Legacy Data in `sails-disk`](#legacy-data-stored-in-the-temporary-sails-disk-database) |
| IX  | [Validations](#validations-upgrade-to-validator-3x) |
| X   | [Adapter/Connections Configuration](#adapterdatabase-configuration) |
| XI  | [Blueprints/Controllers Configuration](#controller-configuration) |
========================================


### Blueprints

A new blueprint action (`findOne`) has been added.  For instance, if you have a `FooController` and `Foo` model, then send a request to `/foo/5`, the `findOne` action in your `FooController` will run.  If you don't have a `findOne` action, the `findOne` blueprint action will be used in its stead.  Requests sent to `/foo` will still run the `find` controller/blueprint action.





### Policies.
Policies work exactly as they did in v0.9- however there is a new consideration you should take into account:  Due to the introduction of the more specific `findOne()` blueprint action mentioned above, you will want to make sure you're handling it explicitly in your policy mapping configuration.

For example, let's say you have a v0.9 app whose `policies.js` configuration prevents access to the `find` action in your `DoveController`:

```javascript
module.exports.policies = {
  '*': true,
  DoveController: {
    find: false
  }
};
```

Assuming `rest` blueprint routes are enabled, this would prevent access to requests like both `/dove` and `/dove/14`.  But now in v0.10, since `/dove/14` will actually run the `findOne` action, we must handle it explicitly:

```javascript
module.exports.policies = {
  '*': true,
  DoveController: {
    find: false,
    findOne: false
  }
};
```


### Pubsub

The biggest change to pubsub is that Socket.io events are emitted under the name of the model emitting them.  Previously, your client listened for the `message` event and then had to determine which model it came from based on the included data:

```
socket.on('message', function(data) {
   if (data.model == 'user') ...
}
```

Now, you subscribe to the identity of the model:

```
socket.on('user', function(data) {...}
```

This helps to structure your front end code.

The way you subscribe clients to models has also changed.  Previously, you specified whether you were subscribing to the model class (class room) or one or more model instances based on the parameters that you passed to `Model.subscribe`.  It was effectively one method to do two very different things.

Now, you use `Model.subscribe()` to subscribe only to model instances (records).  You can also specify event "contexts", or types, that you'd like to hear about.  For example, if you only wanted to get messages about updates to an instance, you would call `User.subscribe(req, myUser, 'update')`.  If no context is given in a call to `.subscribe()`, then all contexts specified by the model class's [autosubscribe property](./#!documentation/reference/ModelProperties) will be used.

To subscribe to model creation events, you can now use `Model.watch()`.  Upon subscription, your clients will receive messages every time a new record is created on that model using the blueprint routes, and will automatically be subscribed to the new instance as well.

Remember, when working with blueprints, clients are no longer auto subscribed to the class room.  This must be done manually.

Finally, if you want to see all pubsub messages from all models, you can access the `firehose`, a development-only tool that broadcasts messages about *everything* that happens to your models.  You can subscribe to the firehose using `sails.sockets.subscribeToFirehose(socket)`, or on the front end by making a socket request to `/firehose`.  The firehose will broadcast a `firehose` event whenever a model is created, updated, destroyed, added to, removed from or messaged.  This effectively replaces the `message` event used in previous Sails versions.

To see examples of the new pubsub methods in action, see [SailsChat](https://github.com/balderdashy/sailschat).


### `.done()` vs. `.exec()`

**The old (/confusing?) meaning of `.done()` has been deprecated.**

In Sails <= v0.8, the syntax for executing an ORM query was `Model. [ … ] .done( cb )`.  In v0.9, when promise support was added, the  `Model. [ … ] .exec( cb )` became the recommended replacement, since `.done()` has a special meaning in the promise spec.  However, the original usage of `.done()` was left untouched to make upgrading from v0.8 to v0.9 easier.

But as of Sails/Weterline v0.10, the original meaning of `.done()` has been officially deprecated to allow for a more robust promise implementation going forward, and pluggable promise library support (e.g. choose `Q` or `Bluebird` etc.).


### Associations

Sails v0.10 introduces associations between data models.  Since the work we've done on associations is largely additive, your existing models should still just work.  That said, this is a powerful new feature that allows you to write less code and makes your app more maintainable, so we suggest taking advantage of it!  To learn about how to use associations in Sails, [check out the docs](./#!documentation/reference/ModelAssociations).

Associations (or "relations") are really just special attributes.  Instead of `string` or `integer` values, you can specify an instance of a model or a collection of model instances.  You can think about this kind of like an object (`{...}`) or an array (`[{...}, {...}]`) you might store as JSON in a NoSQL database.  The difference is, in Sails, this works with any of the supported databases, and even allows you to `populate` (i.e. join) across different databases and types of databases.




### Generators


Sails has had support for generating code for a while now (e.g. `sails generate controller foo`) but in v0.10, we wanted to make this feature more extensible, open, and accessible to everybody in the Sails community.  With that in mind, v0.10 comes with a complete rewrite of the command-line tool, and pluggable generators.  Want to be able to run `sails generate blog foo` to make a new blog built on Sails?  Create a `blog` generator (run `sails generate generator blog`), add your templates, and configure the generator to copy the new templates over.  Then you can release it to the community by publishing an npm module called `sails-generate-blog`.  Compatibility with Yeoman generators is also in our roadmap.


For a complete guide to what generators are and how they work, [check out the in-progress docs on the subject.](https://github.com/balderdashy/sails-docs/blob/master/Guide:%20Using%20Generators.md)





### Command-Line Tool

The big change here is how you create a new api.  In the past you called `sails generate new_api`.  This would generate a new controller and model called `new_api` in the appropriate places.  This is now done using `sails generate api new_api`

You can still generate models and controllers seperately using the same [CLI Commands](./#!documentation/reference/CommandLine/)

Also, `--linker` switch is no longer available. In previos version, if `--linker` switch was provided, it created a `myApp/assets/linker` folder, with `js`, `styles` and `templates` folders inside. In this new version, the `myApp/assets/linker` folder is not created. Compiling CoffeeScript and Less is the default behavior now, right from the `myApp/assets/js` and `myApp/assets/scripts` folders.




### Custom Server Responses

In v0.10, you can now generate your own custom server responses.  [See here to learn how](https://github.com/uncletammy/sails-generate-serverResponse).

Like before, there are a few that we automatically create for you.  Instead of generating `myApp/config/500.js` and other .js responses in the config directory, they are now generated in `myApp/api/responses/`.

To migrate, you will need to create a new v0.10 project and copy the `myApp/api/responses` directory into your existing app.  You will then modify the appropriate .js file to reflect any customization you made in your response logic files (500.js,etc).



### Legacy data stored in the temporary `sails-disk` database

`sails-disk`, used by default in new Sails projects, now stores data a bit differently.  If you have some temporary data stored in a 0.9.x project, you'll want to wipe it out and start fresh.  To do this:

From your project's root directory:
```
$ rm .tmp/disk.db
```


### Validations: Upgrade to validator 3.x

Validator 3.x removed support for the `regex` validation, and consequently it no longer works in Sails/Waterline models.  There is an [open feature request](https://github.com/balderdashy/anchor/issues/41) awaiting a PR to bring it back.



### Adapter/Database Configuration

`config.adapters` (in `myApp/config/adapters.js`) is now `config.connections` (in new projects, this is generated in `myApp/config/connections.js`). Also, `config.model` is now `config.models`.

Your app's default `connection` (i.e. database) should now be configured as a string `config.models.connection` used by default for model.  New projects are generated with a `/config/models.js` file that includes the default connection.

To configure a model to use specific adapters, you must now specify them in the `connection` key instead of `adapters`.

####### For example
```javascript

module.exports = {

    connection: ['someMongoDatabase'],

	attributes: {
		name:{
			type     : 'string',
			required : true
		}
	}
};

```



### Controller configuration

The object literal describing controller configuration overrides for controller blueprints should change from
```javascript
...
_config: {
  blueprints: {
    rest: true,
    ...
  }
}
```
to
```javascript
...
_config: {
    rest: true,
    ...
}
```




### Did we miss something?

If you notice something we've missed, please visit the `sails-docs` repo on GitHub and [edit the page](https://github.com/balderdashy/sails-docs/blob/master/Migration-Guide.md) to submit a pull request.

If you're having migration issues that we havn't mentioned here, please open an issue on the `sails-docs` repo.  If you've already figured out how to resolve the issue, give us a hand by submitting a PR to this file describing the problem and how you fixed it. Thanks!