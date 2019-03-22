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
Here is its code:
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
