# Vuex-Explanation (updating...)
Recently, we have put a lot of effort in migrating our front-end to Vue/Vuex. 
We had a lot of discussion and questions in how to use Vuex properly. 

* Should we use "getter" to retrieve every state in the store?  
* Should we use the store to hold all the data required by front-end?  

To answer these questions, I started to read the source code of Vuex.
## Entry File
Let's see the entry file *src/index* first.
```javascript
import { Store, install } from './store'
import { mapState, mapMutations, mapGetters, mapActions, createNamespacedHelpers } from './helpers'

export default {
  Store,
  install,
  version: '__VERSION__',
  mapState,
  mapMutations,
  mapGetters,
  mapActions,
  createNamespacedHelpers
}
```
It exports a few helper functions like *mapMutations*, *mapState*, *mapGetters*, *mapActions* 
and *createNamespacedHelpers*.  
We will address these functions at the end.   
Let's take a look at the *install* and *Store* first. Both functions come from *src/store*. 
If we ignore the actual code, the *src/store* just exports the *Store* class and *install* function.
```javascript
export class Store {
  ...
}
// A few function definitions
export function install (_Vue) {
  ...
}
```
Let's start with *install* first.
## Install the Vuex
Normally, we attache the Vuex to Vue by this way:
```javascript
Vue.use(Vuex)
```
You might be familiar with [Vue.use](https://vuejs.org/v2/guide/plugins.html#Using-a-Plugin) which is used for installing the Vue's plugins.  
Vue requires its plugin to expose the function "install". 
[How to write a Vue plugin](https://vuejs.org/v2/guide/plugins.html#Writing-a-Plugin).  
Here is the *install* function
```javascript
export function install (_Vue) {
  if (Vue && _Vue === Vue) {
    if (process.env.NODE_ENV !== 'production') {
      console.error(
        '[vuex] already installed. Vue.use(Vuex) should be called only once.'
      )
    }
    return
  }
  Vue = _Vue
  applyMixin(Vue)
}
```
According to Vue's document, the argument *_Vue* is a Vue constructor. Let's skip the if block and take look at the next code
```javascript
Vue = _Vue
```
Variable *Vue* is defined at the top of file *src/store* which is used for holding the reference to a Vue constructor.
Let's go back and see the condition in the if block
```javascript
if (Vue && _Vue === Vue) {
  ....
}
```
It basically checks if the *Vue* is already assigned a Vue constructor and it should be the same as the argument passed to function *install*.  
If the condition is fulfilled, it means the Vuex has been installed twice on the same Vue constructor. Under the development environment, it
prints out a error message.  
At the end of function *install*, it calls function *applyMixin* with Vue constructor
```javascript
applyMixin(Vue)
```
The function *applyMixin* is defined in the file *src/mixin*. This file only exports one anonymous function.
```javascript
export default function (Vue) {
  const version = Number(Vue.version.split('.')[0])

  if (version >= 2) {
    Vue.mixin({ beforeCreate: vuexInit })
  } else {
    // override init and inject vuex init procedure
    // for 1.x backwards compatibility.
    const _init = Vue.prototype._init
    Vue.prototype._init = function (options = {}) {
      options.init = options.init
        ? [vuexInit].concat(options.init)
        : vuexInit
      _init.call(this, options)
    }
  }

  /**
   * Vuex init hook, injected into each instances init hooks list.
   */

  function vuexInit () {
    const options = this.$options
    // store injection
    if (options.store) {
      this.$store = typeof options.store === 'function'
        ? options.store()
        : options.store
    } else if (options.parent && options.parent.$store) {
      this.$store = options.parent.$store
    }
  }
}
```
This anonymous function is passed with a Vue constructor as well.  
The first line of code simple gets the major version of Vue.  
Next is a "if...else..." block.
```javascript
  if (version >= 2) {
    Vue.mixin({ beforeCreate: vuexInit })
  } else {
    // override init and inject vuex init procedure
    // for 1.x backwards compatibility.
    const _init = Vue.prototype._init
    Vue.prototype._init = function (options = {}) {
      options.init = options.init
        ? [vuexInit].concat(options.init)
        : vuexInit
      _init.call(this, options)
    }
  }
```
If the Vuex is installed to a Vue 2 (>=2), then mix the function *vuexInit* into lifecycle *beforeCreate* of Vue.  
Otherwise, if you want to use Vuex on Vue 1, it needs to have some special treatment.
```javascript
// override init and inject vuex init procedure
// for 1.x backwards compatibility.
const _init = Vue.prototype._init
Vue.prototype._init = function (options = {}) {
  options.init = options.init
    ? [vuexInit].concat(options.init)
    : vuexInit
  _init.call(this, options)
}
```
It declares a constant *_init* to save the reference to *Vue.prototype._init*. Then, it decorates *Vue.prototype._init* by
overwriting the option "init".
```javascript
options.init = options.init
  ? [vuexInit].concat(options.init)
  : vuexInit
```
If developer already define the option "init", it concatenates the function *vuexInit* to *options.init*. Otherwise, use *vuexInit* directly.
Finally, pass this "options" to original *_init*.
```javascript
_init.call(this, options)
```
The reason it has special treatment on Vue 1.x is because there is no "beforeCreated" lifecycle hook on Vue 1.x   
**You may wonder why must it be done in the "beforeCreated" lifecycle hook?**  
To answer this question, we need to look at what function *vuexInit* does.
```javascript
/**
* Vuex init hook, injected into each instances init hooks list.
*/

function vuexInit () {
 const options = this.$options
 // store injection
 if (options.store) {
    this.$store = typeof options.store === 'function'
      ? options.store()
      : options.store
 } else if (options.parent && options.parent.$store) {
    this.$store = options.parent.$store
 }
}
```
Firstly, it saves the reference to *[this.$options](https://vuejs.org/v2/api/#vm-options)*. Then, it comes to a "if...else..." block.  
If the *options.store* exists, it uses a ternary expression to set *this.$store*. If you ever used the Vue/Vuex, you should already be familiar with *$store*.
Vue expose this variable to offer you a convenient way to access the store. It allows developer to pass the store definition either as a function or
as an object.  
If developer does not define any store, then it goes to "else if"
```javascript
 else if (options.parent && options.parent.$store) {
    this.$store = options.parent.$store
 }
```
What is the *options.parent*?  
From the name, it is not hard to figure that is the *options* of parent component. This property *parent* is set by function *createComponentInstanceForVnode* in the Vue.
If you are interested, you can search the file *src/vdom/create-component*.  
So, the above code means if the current component is passed with store and its parent component has *$store*, then use the *$store* from parent component.

## Create a store
As Vuex documents suggests, the easiest way of creating a store is the following:
```javascript
const store = new Vuex.Store({
  state: {
    count: 0
  },
  mutations: {
    increment (state) {
      state.count++
    }
  }
})
```
From above code, we can know *Vuex.Store* is a class. Its constructor is defined in the file *src/store*.
```javascript
export class Store {
  constructor (options = {}) {
    // Auto install if it is not done yet and `window` has `Vue`.
    // To allow users to avoid auto-installation in some cases,
    // this code should be placed here. See #731
    if (!Vue && typeof window !== 'undefined' && window.Vue) {
      install(window.Vue)
    }

    if (process.env.NODE_ENV !== 'production') {
      assert(Vue, `must call Vue.use(Vuex) before creating a store instance.`)
      assert(typeof Promise !== 'undefined', `vuex requires a Promise polyfill in this browser.`)
      assert(this instanceof Store, `store must be called with the new operator.`)
    }

    const {
      plugins = [],
      strict = false
    } = options

    // store internal state
    this._committing = false
    this._actions = Object.create(null)
    this._actionSubscribers = []
    this._mutations = Object.create(null)
    this._wrappedGetters = Object.create(null)
    this._modules = new ModuleCollection(options)
    this._modulesNamespaceMap = Object.create(null)
    this._subscribers = []
    this._watcherVM = new Vue()

    // bind commit and dispatch to self
    const store = this
    const { dispatch, commit } = this
    this.dispatch = function boundDispatch (type, payload) {
      return dispatch.call(store, type, payload)
    }
    this.commit = function boundCommit (type, payload, options) {
      return commit.call(store, type, payload, options)
    }

    // strict mode
    this.strict = strict

    const state = this._modules.root.state

    // init root module.
    // this also recursively registers all sub-modules
    // and collects all module getters inside this._wrappedGetters
    installModule(this, state, [], this._modules.root)

    // initialize the store vm, which is responsible for the reactivity
    // (also registers _wrappedGetters as computed properties)
    resetStoreVM(this, state)

    // apply plugins
    plugins.forEach(plugin => plugin(this))

    const useDevtools = options.devtools !== undefined ? options.devtools : Vue.config.devtools
    if (useDevtools) {
      devtoolPlugin(this)
    }
  }
  ...
}
``` 
It is a little bit long, but don't worry. Let's review it line by line.
Firstly, Let's see this portion:
```javascript
    // Auto install if it is not done yet and `window` has `Vue`.
    // To allow users to avoid auto-installation in some cases,
    // this code should be placed here. See #731
    if (!Vue && typeof window !== 'undefined' && window.Vue) {
      install(window.Vue)
    }
```
If you are interested, you can take a look at issue #731. Before the fix, if there is a global vue defined on "window.Vue", since the Vuex is installed
when it is imported, there is no way that we can bind the Vuex with another Vue (different version). Now, the Vuex is installed during initializing the store,
we can install the Vuex to Vue before creating the store. Then, it works if two different version of Vuex coexists.
Let's move on.
```javascript
if (process.env.NODE_ENV !== 'production') {
  assert(Vue, `must call Vue.use(Vuex) before creating a store instance.`)
  assert(typeof Promise !== 'undefined', `vuex requires a Promise polyfill in this browser.`)
  assert(this instanceof Store, `store must be called with the new operator.`)
}
```
This piece of code prints out some warning message when the environment is development environment.
The assert function is defined in *src/util*.
```javascript
export function assert (condition, msg) {
  if (!condition) throw new Error(`[vuex] ${msg}`)
}
```
Simple. If the condition is evaluated to false, then throws an error with the message passed in.
Back to *constructor*, the three messages are:
* If the *Vue* is not defined, then developer should install the Vuex first.
* If the global Promise is not defined, then developer needs to define a Promise Polyfill first.
* If *this* is not an instance of *Store*, then warn developer to initialize a store with *new* operator.

Next, set default values for a bunch of variables. Let's take a look. 
```javascript
const {
  plugins = [],
  strict = false
} = options
```
The default value of [plugins](https://vuex.vuejs.org/guide/plugins.html) is an empty array.  
The [strict mode](https://vuex.vuejs.org/guide/strict.html) is disabled initially.  
```javascript
// store internal state
this._committing = false
this._actions = Object.create(null)
this._actionSubscribers = []
this._mutations = Object.create(null)
this._wrappedGetters = Object.create(null)
this._modules = new ModuleCollection(options)
this._modulesNamespaceMap = Object.create(null)
this._subscribers = []
this._watcherVM = new Vue()
```
Then, it sets some internal states. *_actions*, *_mutations*, *_wrappedGetters* and *_modulesNamespaceMap* are set to a [null object](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/create#Custom_and_Null_objects).
The null object a like a plain object ({}) without the methods on the prototype.  
*_subscribers* and *_actionSubscribers* are set to an empty array.  
*_watcherVM* is set to an instance of Vue.
The most important code here is following:
```javascript
this._modules = new ModuleCollection(options)
```
The *_modules* is the instance of *ModuleCollection* and it is initialize with the options of store constructor.
The *ModuleCollection* is defined in *src/module/module-collection*. Let's look at its constructor first.
```javascript
constructor (rawRootModule) {
  // register root module (Vuex.Store options)
  this.register([], rawRootModule, false)
}
```
The *constructor* has one parameter *rawRootModule* and it calls the method *register* with three parameters.
```javascript
  register (path, rawModule, runtime = true) {
    if (process.env.NODE_ENV !== 'production') {
      assertRawModule(path, rawModule)
    }

    const newModule = new Module(rawModule, runtime)
    if (path.length === 0) {
      this.root = newModule
    } else {
      const parent = this.get(path.slice(0, -1))
      parent.addChild(path[path.length - 1], newModule)
    }

    // register nested modules
    if (rawModule.modules) {
      forEachValue(rawModule.modules, (rawChildModule, key) => {
        this.register(path.concat(key), rawChildModule, runtime)
      })
    }
  }
```
In the *constructor* of *Module Collection*, the path is an empty array. The *rawModule* is the options of *Store Constructor*.  
The default value of *runtime* is true. But, in the *constructor*, it is set to *false*. So, there must be somewhere else calls the function
*register* without setting *runtime*. We will discuss it later.  
After explaining all the parameters, let's get to the body of *register*.  
```javascript
if (process.env.NODE_ENV !== 'production') {
  assertRawModule(path, rawModule)
}
```
Under the development environment, it calls the function *assertRawModule* and this function is defined in the same file.  
```javascript
function assertRawModule (path, rawModule) {
  Object.keys(assertTypes).forEach(key => {
    if (!rawModule[key]) return

    const assertOptions = assertTypes[key]

    forEachValue(rawModule[key], (value, type) => {
      assert(
        assertOptions.assert(value),
        makeAssertionMessage(path, key, type, value, assertOptions.expected)
      )
    })
  })
}
```
If we ignore the code inside of *forEach*, then *assertModule* becomes:
```javascript
function assertRawModule (path, rawModule) {
  Object.keys(assertTypes).forEach(key => {
    ...
  })
}
```
It simply goes through each key of *assertTypes* object. The *assertTypes* is defined right above function *assertRawModule*.
```javascript
const assertTypes = {
  getters: functionAssert,
  mutations: functionAssert,
  actions: objectAssert
}
```
So, the value of *key* are *getters*, *mutations* and *actions*. On each type, *assertRawModule* does
```javascript
if (!rawModule[key]) return

const assertOptions = assertTypes[key]

forEachValue(rawModule[key], (value, type) => {
  assert(
    assertOptions.assert(value),
    makeAssertionMessage(path, key, type, value, assertOptions.expected)
  )
})
```
Let's look at the first statement.  
```javascript
if (!rawModule[key]) return
```
We already know the *rawModule* is the options passed to Vuex constructor. So, assuming we initialize the store as the following:
```javascript
const store = new Vuex.store({
  getters: {}
});
```
Then, it directly returns when processing *mutations* and *actions*. If any of *assert types* is passed in, then it goes to the second statement:
```javascript
const assertOptions = assertTypes[key]
```
It gets value from *assertTypes* and assign it to variable *assertOptions*. According to the definition of *assertTypes*, we know the value is 
*functionAssert* for both *getters* and *mutations*. But, for *actions*, the value is *objectAssert*.  
Let's look at what is the *functionAssert* first.  It is defined in the same file.
```javascript
const functionAssert = {
  assert: value => typeof value === 'function',
  expected: 'function'
}
```
The *functionAssert* is an object with two properties. The property *assert* is a function which check if the passed value is a function.
The value of property *expected* is a string "function".  
The *objectAssert* is defined right below the *functionAssert*.   
```javascript
const objectAssert = {
  assert: value => typeof value === 'function' ||
    (typeof value === 'object' && typeof value.handler === 'function'),
  expected: 'function or object with "handler" function'
}
```
The *assert* function here checks if the value is a function or an object with property *handler*.
Back to *forEach* body in *assertRawModule*, variable *assertOptions* could be one of *functionAssert* and *objectAssert*.
Let's see the next line.
```javascript
forEachValue(rawModule[key], (value, type) => {
  assert(
    assertOptions.assert(value),
    makeAssertionMessage(path, key, type, value, assertOptions.expected)
  )
})
```
Another loop on each item in *rawModule[key]*. As explained before, *rawModule[key]* is each "action", "mutation" and "getter" developer defines
and passes to *Vuex.Store*. So, on each "action", "mutation" and "getter", calls function *assert*. We have explained the function *assert*.
It prints out the second parameter if the first parameter is evaluated to false.  
Thus, for each "mutation" or "getter", its value must be a function. For each "action", its value could be either a function or an object with
property *handler*. Otherwise, it calls function *makeAssertionMessage*.
```javascript
function makeAssertionMessage (path, key, type, value, expected) {
  let buf = `${key} should be ${expected} but "${key}.${type}"`
  if (path.length > 0) {
    buf += ` in module "${path.join('.')}"`
  }
  buf += ` is ${JSON.stringify(value)}.`
  return buf
}
```
It returns a string which is the message printed in the console when the assertion fails.
So, the function *assertRawModule* is used for validating the *actions*, *mutations* and *getters*.
Let's go back to *register*. The next line of code is
```javascript
const newModule = new Module(rawModule, runtime)
```
It creates an instance of class *Module*. The *rawModule* here is the options of *constructor of store*. And *runtime* is false.
The class *Module* is defined in the file *src/module/module*.
```javascript
export default class Module {
    constructor (rawModule, runtime) {
      this.runtime = runtime
      // Store some children item
      this._children = Object.create(null)
      // Store the origin module object which passed by programmer
      this._rawModule = rawModule
      const rawState = rawModule.state
  
      // Store the origin module's state
      this.state = (typeof rawState === 'function' ? rawState() : rawState) || {}
    }
    ...
}
```
In the constructor, it sets a bunch of initial states. Assuming we create a store:
```javascript
const store = new Vuex.Store({
  state: {
    open: false
  }
});
```
So, the *_rawModule* is
```javascript
{
  state: {
    open: false
  }
}
```
And, the *rawState* is
```javascript
{
  open: false
}
```
From the last line we can know, the *state* here can be defined as a function or an object. 
```javascript
this.state = (typeof rawState === 'function' ? rawState() : rawState) || {}
```
After creating an instance of *Module*, the *register* function goes to
```javascript
if (path.length === 0) {
  this.root = newModule
} else {
  const parent = this.get(path.slice(0, -1))
  parent.addChild(path[path.length - 1], newModule)
}
```
It is a *if...else*. The first condition is
```javascript
path.length === 0
```
When the path is an empty array which is the case in the constructor of store, then it must be the root module.  
So, it saves the root module to *this.root*.
Otherwise, the module must be a children module.
```javascript
  const parent = this.get(path.slice(0, -1))
  parent.addChild(path[path.length - 1], newModule)
```
The *path.slice(0, -1)* gets the path to the parent module. So, function *get* is called with the path of parent module.
```javascript
  get (path) {
    return path.reduce((module, key) => {
      return module.getChild(key)
    }, this.root)
  }
```
The function *get* is actually a reducer. The accumulator is *this.root*. Since the *this.root* points to the root module, so *module* is the
instance of the *Module* class. Here is the method defined on *Module* class.
```javascript
getChild (key) {
  return this._children[key]
}
```
Simple. It just gets the value from *Module._children*. From the constructor of *Module*, we can know that *_children* is an object which is
used for storing all the children modules.
Once it gets the parent module, the current module is added to parent module.
```javascript
parent.addChild(path[path.length - 1], newModule)
```
*path[path.length - 1]* is the last item of the path which is the path of current module. Let's look at the method *addChild*.
```javascript
addChild (key, module) {
  this._children[key] = module
}
```
It adds the module to *_children* of parent module with the *key*.
Assuming we have a user module and here is its path:
```javascript
path = ['app', 'system', 'user']
```
So, the parent module of module "user" is system. In the module "system", it is
```javascript
{
  _children: {
    "user": user_module
  }
}
```
Then, the last part of the *register* is
```javascript
// register nested modules
if (rawModule.modules) {
  forEachValue(rawModule.modules, (rawChildModule, key) => {
    this.register(path.concat(key), rawChildModule, runtime)
  })
}
```
If developer defines the *modules* in the options of store constructor, then it registers all these modules as children modules by recursively
calling the method *register*.  
The first parameter of *register* become *path.cancat(key)* which is no longer an empty array. So, the module is added to its parent module.