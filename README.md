# seneca-perm

### Node.js Seneca permissions module

This module is a plugin for the Seneca framework. It provides a
permissions system for actions. It's also a good example of how you can compose actions by layering plugins onto each other.

This plugin works by wrapping existing actions with a permission checking action. If the permission test passes, the parent
action can proceed. If not, a permission error is generated.

The possible permission checks are:

   * _allow_: simple yes/no
   * _act_: allow only specific actions to pass
   * _entity_: allow only specific actions on entities
   * _own_: allow on specific actions on entities that are 'owned' by given users 

This plugin also understands when it is used in a web server context, and will add a permission specification to 
the req.seneca object if it exists.

A full example, in the context of the seneca data editor, is provided
in the [seneca examples repository](https://github.com/rjrodger/seneca-examples). TODO!


### Support

If you're using this module, feel free to contact me on twitter if you
have any questions! :) [@rjrodger](http://twitter.com/rjrodger)

Current Version: 0.1.0

Tested on: node 0.8.16, seneca 0.5.6


### Quick example

You use this plugin mainly by adding a _perm_ object to user entities:

```JavaScript
var userent = seneca.make('sys','user')

userent.find({email:'alice@example.com'}, function(err,alice){
  alice.perm = {
    act:[ 
      {foo:'bar', perm$:true},
      {qaz:'lol', perm$:false},
    ]
  }
  alice.save$()
})
```

This permission specification allows the user alice to execute _foo:bar_ actions, but not _qaz:lol_ ones.
The perm$ metadata property specifies the permissions for each action pattern in the list.

Here's another example, this time for entities:

```JavaScript
var userent = seneca.make('sys','user')

userent.find({email:'alice@example.com'}, function(err,alice){
  alice.perm = {
    entity:[ 
      {base:'shop', name:'cart', perm$:'r'},
      {base:'qaz', perm$:'crudq'},
    ]
  }
  alice.save$()
})
```

In this case, alice can only read ('r') from -/shop/cart entities, but can do anything to -/qaz/- entities.
There are more details on the [seneca data entity model](http://senecajs.org/data-entities.html) here.

Of course, this example code does not actually work, as all of the setup and configuration is missing.
Here is some working code that provides a minimal example by setting up some of the values manually.

### Working Code

This code applies a permissions test to the _echo_ plugin. The _echo_
plugin has one action _role:echo_, that just gives you back the same
arguments you put it. The example code adds a permission check to this
action, and shows you how to trigger it manually, using the _perm$_ argument.


```JavaScript
var seneca  = require('seneca')()
seneca.use('echo')

seneca.use( 'perm', {act:[
  {role:'echo'},
]})

seneca.act({role:'perm', cmd:'init'})

seneca.act({role:'echo', foo:'bar', perm$:{allow:true}},function(err,out){
  console.log('foo='+out.foo)
})

seneca.act({role:'echo', foo:'bar', perm$:{allow:false}},function(err,out){
  console.log(err)
})
```


The _perm$_ metadata argument is recognised by the _perm_ plugin and contains the permission specification to test against.
Normally, you won't set this manually, but generate it using the _role:perm, cmd:makeperm_ action. In a web context, this
plugin also does this for you, based on the perm property of the current user.


### Web App Code

Here's an outline of how you would use it in a
[connect](http://www.senchalabs.org/connect) app, using the _user_ and
_auth_ plugins to provide user accounts:

```JavaScript
var seneca  = require('seneca')()
var connect = require('connect')

seneca.use('user')
seneca.use('auth')

seneca.use('perm',{
  act:[{role:'echo'}]
})

seneca.use('echo')

seneca.act({role:'perm', cmd:'init'})

var app = connect()
app.use( seneca.service() )
app.listen(3000)
```

The perm plugin does not wrap other actions immediately when it is registered. Rather, you call the _role:perm, cmd:init_ action
when you're ready. First you need to add any other plugins and actions that you want to apply permissions checking to.



## Install

```sh
npm install seneca
npm install seneca-perm
```


## Usage

This plugin has two elements. First, the options define the set of actions that permission checks will apply to. Second, permission
checks only occur if there is a _perm$_ metadata argument, containing a permissions specification. 

### Permission options

   * _act_: array of action pins
   * _entity_: array of entity type specifications
   * _own_: array of entity type specifications

These specify the actions to wrap. For example:

```JavaScript
seneca.use( 'perm', {act:[
  {role:'echo'}
]})
```

This wraps any action with a _role:echo_ argument, which means that it will have a permission check applied.

You need to specify explicitly the actions to which you wish to apply permission checks.

As a convenience, you can apply permission checks to entities by simply specifying their zone, base and name (all optional):

```JavaScript
seneca.use( 'perm', {entity:[
  {base:'sys'},
  {base:'foo',name:'bar'}
]})
```

The above code specifies that actions on any entity of type -/sys/- or -/foo/bar will have an permission check applied. 

The _entity_ option saves you from having to specify a permission check for all the entity actions, otherwise you would have to do this:

```JavaScript
seneca.use( 'perm', {act:[
  {role:'entity',cmd:'save',base:'sys'},
  {role:'entity',cmd:'load',base:'sys'},
  {role:'entity',cmd:'list',base:'sys'},
  ...
]})
```

The allowed entity operations (create, read, update, delete, query)
are specified in the perm$ metadata argument to the entity actions
(see below).

The _own_ option works in much the same way as the _entity_ option,
except that the user must be the owner of the entity in question.
Entities should have an _owner_ property containing the identifier of the -/sys/user entity for this to work.

```JavaScript
seneca.use( 'perm', {own:[
  {base:'foo'}
]})
```


### Permission specifications

To trigger a permissions check, an action must contain a perm$ metadata argument. This is an object contains one or more of the
following properties:

   * allow: boolean, true if action is permitted.
   * act: an action router object; only matching actions can be executed
   * entity: an entity type router object, matching the entity action, and specifying the operations permitted
   * own: an entity type router object, matching the entity action, and specifying the operations permitted, if also owner of the entity

You do not normally construct the perm$ object directly, but instead use the _role:perm, cmd:makeperm_ action to create one from a 
literal definition (you can store this in the -/sys/user entity, for example). If you store the definition in a _perm_ property on
-/sys/user, and use the perm plugin in a web context, then this is done for you automatically.


#### allow

To store as a literal, use this structure in the _perm_ property:

```JavaScript
    {allow:true|false}
```

This will converted to the perm$ metadata argument, by the _role:perm, cmd:makeperm_ action:

```JavaScript
{allow:true|false}
```

In general, this check is not particularly useful for individual
users, serving rather to provide a global block on certain
actions. You would do this by adding an allow property to any _perm$_
you generate.


#### act

To store as a literal, use this structure in the _perm_ property:

```JavaScript
{act:[
  { role:'...', cmd:'...', perm$:true },
  { role:'...', perm$:false },
  { foo:'bar', perm$:true },
  ...
]}
```

The _act_ sub property is an array of action pins. Each pin specifies
the argument properties to match against. The perm$ value indicates if
the action is allowed or not.

This will converted to the perm$ metadata argument:

```JavaScript
{act:router}
```

where _router_ is a [seneca router](http://senecajs.org/routing.html) TODO! (the same thing that routes action arguments to plugin functions). 
The router matches a given set of action arguments to the permission specification.

You could construct it manually, like so:

```JavaScript
var router = seneca.util.router()
router.add( {role:'...',cmd:'...'}, true )
router.add( {role:'...'}, false )
router.add( {foo:'bar'}, true )
```

Note that as with all routers, the action arguments are matched in alphabetical order.


#### entity

To store as a literal, use this structure in the _perm_ property:

```JavaScript
{act:[
  { base:'sys', perm$:'' },
  { base:'foo', perm$:'rq' },
  { base:'foo': name:'bar', perm$:'crudq' },
  ...
]}
```

This specification is similar to the act specification above, except
that the entity type is matched against. The permission value encodes the allowed operations. There are:

   * create (r): create new entities
   * read (r): load an entity by identifier
   * update (r): modify an entity by identifier
   * delete (r): remove an entity by identifier
   * query (r): perform queries to list multiple entites

The query (q) permission also allows you to perform queries for the
read, update and delete operations, if you have permission for those
too.

In the example specification above, the user has no permissions on
-/sys/- entities, can only read and query -/foo/- entities, and
can perform any operations on -/foo/bar entities.

The perm$ metadata argument form is:

```JavaScript
{entity:router}
```

where the router is constructed in the same way as the _act_
permission, except using the entity zone, base and name.
 The data values in the router are the _crudq_ operation
specifications.


#### entity

To store as a literal, use this structure in the _perm_ property:

```JavaScript
{act:[
  { base:'sys', perm$:'' },
  { base:'foo', perm$:'rq' },
  { base:'foo': name:'bar', perm$:'crudq' },
  ...
]}
```

This is the same as the _entity_ permission. However, the permissions only apply if the entity has an _owner_
property that matches the identifier of the user executing the action.

The perm plugin handles all the set up for you when used in a web context. See the _test/perm.app.js_ example code.

The perm$ metadata argument form is:

```JavaScript
{own:{
  entity:router
  owner:'...'
}}
```

Where the entity property is a router on the entity zone, base and
name, has data values of the form 'crudq', and the owner is the
identifier of the user.
 

## Test

```bash
cd test
mocha perm.test.js --seneca.log.print
```