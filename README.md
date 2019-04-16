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
### How modules are initialized?
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
*forEachValue* is defined in the *util*  
If developer defines the *modules* in the options of store constructor, then it registers all these modules as children modules by recursively
calling the method *register*.  
The first parameter of *register* become *path.cancat(key)* which is no longer an empty array. So, the module is added to its parent module.
Let's take the example in the official document.
```javascript
const moduleA = {
  state: { ... },
  mutations: { ... },
  actions: { ... },
  getters: { ... }
}

const moduleB = {
  state: { ... },
  mutations: { ... },
  actions: { ... }
}

const store = new Vuex.Store({
  modules: {
    a: moduleA,
    b: moduleB
  }
})
```
The module collection will become:
```javascript
{
  root: {
    _children: {
      a: moduleA,
      b: moduleB
    }
  }
}
```
### How the modules are installed?
Let's see the next important function in the *store.constructor* - installModule.  
```javascript
// init root module.
// this also recursively registers all sub-modules
// and collects all module getters inside this._wrappedGetters
installModule(this, state, [], this._modules.root)
```
It is called with three parameters. The first parameter is the instance of the store. The
second parameter is *state* which is defined right above the *installModule*.
```javascript
const state = this._modules.root.state
```
The variable state points to the state of the root module.  
The third parameter is an empty array. And, the final parameter is root module.  
Let's take a look at the method *installModule*. It is defined in the file *src/store* as well.
```javascript
function installModule (store, rootState, path, module, hot) {
  const isRoot = !path.length
  const namespace = store._modules.getNamespace(path)

  // register in namespace map
  if (module.namespaced) {
    store._modulesNamespaceMap[namespace] = module
  }

  // set state
  if (!isRoot && !hot) {
    const parentState = getNestedState(rootState, path.slice(0, -1))
    const moduleName = path[path.length - 1]
    store._withCommit(() => {
      Vue.set(parentState, moduleName, module.state)
    })
  }

  const local = module.context = makeLocalContext(store, namespace, path)

  module.forEachMutation((mutation, key) => {
    const namespacedType = namespace + key
    registerMutation(store, namespacedType, mutation, local)
  })

  module.forEachAction((action, key) => {
    const type = action.root ? key : namespace + key
    const handler = action.handler || action
    registerAction(store, type, handler, local)
  })

  module.forEachGetter((getter, key) => {
    const namespacedType = namespace + key
    registerGetter(store, namespacedType, getter, local)
  })

  module.forEachChild((child, key) => {
    installModule(store, rootState, path.concat(key), child, hot)
  })
}
```
The first line defines a constant *isRoot*. If the path is empty, then *isRoot* is true.
In the case of the *store.constructor*, the *installModule* is called with third parameter
*[]*, so *isRoot* is true.  
The second line is
```javascript
const namespace = store._modules.getNamespace(path)
```
We explained before, the *store._modules* is a module collection. So, let's see how method
*getNamespace* is defined on the module collection. 
```javascript
getNamespace (path) {
  let module = this.root
  return path.reduce((namespace, key) => {
    module = module.getChild(key)
    return namespace + (module.namespaced ? key + '/' : '')
  }, '')
}
```
If the module need to be namespaced, then it only need to set *namespaced* to true.
The variable *module* points to the root module. Then, it reduces the *path* to a string.
Every time, it get the child module by the key. If the module is namespaced, then it adds
the key to the accumulator.  

Let's say we have the following modules defined:
```javascript
const store = new Vuex.Store({
  modules: {
    account: {
      namespaced: true,
      modules: {
        profile: {
          namespaced: true,
          ...
        },
        permission: {
          namespaced: true,
          ...
        }
      },
      ...
    }
  }
});
```
If the path is *["account", "profile"]*, then *getNamespace* returns "account.profile".
In the *store.constructor*, *installModule* is called with *path=[]*, so the constant
*namespace* is an empty string.
```javascript
const namespace = store._modules.getNamespace(path)
```
Let's move on.
```javascript
// register in namespace map
if (module.namespaced) {
  store._modulesNamespaceMap[namespace] = module
}
```
So, all the children modules are registered in the *_modulesNamespaceMap*.
```javascript
store._modulesNamespaceMap = {
  'account.profile': profileModule,
  'account.permission': permissionModule
}
```
To make it easier to understand, I will skip the next if block. It will be explained later.
Then, it sets the variable *local* and *module.context*
```javascript
const local = module.context = makeLocalContext(store, namespace, path)
```
The definition of method "makeLocalContext" is
```javascript
/**
 * make localized dispatch, commit, getters and state
 * if there is no namespace, just use root ones
 */
function makeLocalContext (store, namespace, path) {
  const noNamespace = namespace === ''

  const local = {
    dispatch: noNamespace ? store.dispatch : (_type, _payload, _options) => {
      const args = unifyObjectStyle(_type, _payload, _options)
      const { payload, options } = args
      let { type } = args

      if (!options || !options.root) {
        type = namespace + type
        if (process.env.NODE_ENV !== 'production' && !store._actions[type]) {
          console.error(`[vuex] unknown local action type: ${args.type}, global type: ${type}`)
          return
        }
      }

      return store.dispatch(type, payload)
    },

    commit: noNamespace ? store.commit : (_type, _payload, _options) => {
      const args = unifyObjectStyle(_type, _payload, _options)
      const { payload, options } = args
      let { type } = args

      if (!options || !options.root) {
        type = namespace + type
        if (process.env.NODE_ENV !== 'production' && !store._mutations[type]) {
          console.error(`[vuex] unknown local mutation type: ${args.type}, global type: ${type}`)
          return
        }
      }

      store.commit(type, payload, options)
    }
  }

  // getters and state object must be gotten lazily
  // because they will be changed by vm update
  Object.defineProperties(local, {
    getters: {
      get: noNamespace
        ? () => store.getters
        : () => makeLocalGetters(store, namespace)
    },
    state: {
      get: () => getNestedState(store.state, path)
    }
  })

  return local
}
```
At first glance, this method might look complex. Let's simplify it a little bit.
```javascript
function makeLocalContext (store, namespace, path) {
  const local = {
    ...
  };
  
  Object.defineProperties(local, {});
  return local;
}
```
This method just defines a plain object and returns it.  
What exactly the variable *local* is defined? 
First of all, the *local* is initialized as a plain object with two properties *dispatch* and *commit*. The value of both properties
are a ternary operator. Let's see the *dispatch* first.  
```javascript
const local = {
  dispatch: noNamespace ? store.dispatch : (_type, _payload, _options) => {
    const args = unifyObjectStyle(_type, _payload, _options)
    const { payload, options } = args
    let { type } = args

    if (!options || !options.root) {
      type = namespace + type
      if (process.env.NODE_ENV !== 'production' && !store._actions[type]) {
        console.error(`[vuex] unknown local action type: ${args.type}, global type: ${type}`)
        return
      }
    }

    return store.dispatch(type, payload)
  }
}
```
The variable *noNamespace* is defined at the beginning of the method.  
```javascript
const noNamespace = namespace === ''
```
If the second parameter of method is not an empty string, then, it is defined in a name space.  
If there is no namespace defined, then it directly uses the dispatch defined in the root store. Otherwise, it would be a function.
The function has three parameters, *_type*, *_payload* and *_options*. Normally, we use the *dispatch* like this:
```javascript
store.dispatch('ADD_WIDGET', {widgetId: 1});
```
If we use the dispatch in a module, we can pass the third parameter.
```javascript
store.dispatch('ADD_WIDGET', {widgetId: 1}, {root: true});
```
So, let's look at what *dispatch* does internally if the module is namespaced.  
In the first line, all the three parameters are passed to a method called *unifyObjectStyle*, its definition is
```javascript
function unifyObjectStyle (type, payload, options) {
  if (isObject(type) && type.type) {
    options = payload
    payload = type
    type = type.type
  }

  if (process.env.NODE_ENV !== 'production') {
    assert(typeof type === 'string', `expects string as the type, but found ${typeof type}.`)
  }

  return { type, payload, options }
}
```
Firstly, it comes to a if block. It checks if the first parameter *type* is an object. If it is true, then check whether the object has property
*type*. If both conditions are met, then it reassign the variables.
The second parameter becomes the *options*.  
The first parameter becomes the *payload*.  
The type is assigned with *type.type*.
At the end of this method, it returns three parameters as a single object.
You might already figured out that it compresses the first and second parameter of the *dispatch* into one. This is called "object style"
dispatch in the document.  
Under the development mode, if the *type* is not a string, it warns you that the *type* must be a string.  
Then, in the *dispatch* function, it assigns a few variables and come to a if block.
```javascript
if (!options || !options.root) {
  type = namespace + type
  if (process.env.NODE_ENV !== 'production' && !store._actions[type]) {
    console.error(`[vuex] unknown local action type: ${args.type}, global type: ${type}`)
    return
  }
}
```
The condition means the *dispatch* doesn't call with *options* or setting *options.root* to false, then it must dispatch a local action.
Then, variable *type* is prefixed with *namespace*. When it is under the development environment and the namespaced the action is not defined,
then prints out an error message.  
Finally, it calls *store.dispatch* with *type* and *payload*. Then, it returns its result.  
The *commit* is pretty much the same. Except, at the end, it calls the *store.commit* with third parameter *options*. But, it doesn't return
the result.
```javascript
// getters and state object must be gotten lazily
// because they will be changed by vm update
Object.defineProperties(local, {
  getters: {
    get: noNamespace
      ? () => store.getters
      : () => makeLocalGetters(store, namespace)
  },
  state: {
    get: () => getNestedState(store.state, path)
  }
})
```
It adds two properties *getters* and *state*. Instead of defining what is the getters and state, it defines the *getter* of both properties.
For *getters*, if the namespace is not defined, then it calls the *makeLocalGetters*
```javascript
function makeLocalGetters (store, namespace) {
  const gettersProxy = {}

  const splitPos = namespace.length
  Object.keys(store.getters).forEach(type => {
    // skip if the target getter is not match this namespace
    if (type.slice(0, splitPos) !== namespace) return

    // extract local getter type
    const localType = type.slice(splitPos)

    // Add a port to the getters proxy.
    // Define as getter property because
    // we do not want to evaluate the getters in this time.
    Object.defineProperty(gettersProxy, localType, {
      get: () => store.getters[type],
      enumerable: true
    })
  })

  return gettersProxy
}
```
First of all, it defines the constant *gettersProxy* and return it at the end of the function. So, we could guess the code in between is just
filling the *gettersProxy*. Then, it assign the length of *namespace* to a constant *splitPos*.  
Next, it loops through the keys of all *store.getters*.
```javascript
type.slice(0, splitPos) !== namespace
```
*splitPos* is the length of the namespace. So, it extracts the namespace from key. If it fails to extract, then return directly.  
If it successfully extract the namespace, then it gets the local type from the *type*. After that, it defines the *localType* on *gettersProxy*.
The *getter* of *localType* points to *store.getters[type]* and it is enumerable.  
So, all the "local getters" actually proxy to the *store.getters*.
