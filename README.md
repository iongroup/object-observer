[![npm version](https://badge.fury.io/js/object-observer.svg)](https://badge.fury.io/js/object-observer)

# Summary

Observation of a changes performed on any arbitrary object (array being a subtype of it) is a **MUST HAVE** facility in JavaScript world.

Native facility would be the best solution for this, since it may provide non-intrusive observation wihtout 'touching' the original objects, but seems like ES is not yet mature enough for that.

Present library attempts to provide this functionality in a most clean and performant way. Main aspects:
- Implementation relies on __Proxy__ mechanism
- Observation is 'deep', yielding changes from a __sub-graphs__ too
- Changes delivered in a __synchronous__ way
- Changes delivered always as an __array__, in order to have unified callback API signature supporting future bulk changes delivery in a single call back
- Original objects are __cloned__ and clone/s are __instrumented__, thus not affecting the original objects yet requiring few basic steps in a consumption flow
  - first, create observable clone from the specified object
  - second, register observers on the observable (not on the original object)
- Arrays:
  - generic object-like mutations supported
  - intrinsic mutation methods supported: `push`, `pop`, `shift`, `unshift`, `reverse`, `sort`, `fill`, `splice` (see below for more info on changes delivery for these)
  - massive mutations delivered in a single callback, usually having an array of an atomic changes

#### Support matrix: ![CHROME](https://raw.githubusercontent.com/alrra/browser-logos/master/chrome/chrome_24x24.png) <sub>49+</sub>, ![FIREFOX](https://raw.githubusercontent.com/alrra/browser-logos/master/firefox/firefox_24x24.png) <sub>42+</sub>, ![EDGE](https://raw.githubusercontent.com/alrra/browser-logos/master/edge/edge_24x24.png) <sub>13+</sub>
Support matrix is mainly dependent on 2 advanced language features: `Proxy` and `Reflect`. The broader their adoption - the broader the support matrix of ObjectObserver.

#### Backlog:
 - Optimization for the cases of Array massive mutations
 - Add `readPath` and `writePath` utility methods in `DataPath` object (part of change data)?

#### Versions
- 0.1.0
  - First stable API release


# Loading the Library

You have 2 ways to load the library: into a 'window' global scope, or a custom scope provided by you.

* Simple a reference (script tag) to the object-oserver.js in your HTML will load it into the __global scope__:
```html
<script src="object-observer.js"></script>
<script>
	var person = { name: 'Uriya', age: 8 },
	    observablePerson;
	observablePerson = Observable.from(person);
</script>
```

* Following loader exemplifies how to load the library into a __custom scope__ (add error handling as appropriate):
```javascript
var customNamespace = {},
    person = { name: 'Nava', age: 6 },
    observablePerson;

fetch('object-observer.js').then(function (response) {
	if (response.status === 200) {
		response.text().then(function (code) {
			Function(code).call(customNamespace);
			
			//	the below code is an example of consumption, locate it in your app lifecycle/flow as appropriate
			observablePerson = customNamespace.Observable.from(person);
		});
	}
});
```

# APIs

##### `Observable` static properties

- __`from`__ - receives a _non-null object_ and returns __`Observable`__ interface:
	```javascript
	var person = { name: 'Aya', age: '1' },
		observablePerson;

	observablePerson = Observable.from(person);
	...
	```

##### `Observable` instance properties

- __`observe`__ - receives a _function_, which will be added to the list of observers subscribed for a changes of this observable:
	```javascript
	function personUIObserver(changes) {
		changes.forEach(change => {
			console.log(change.type);
			console.log(change.path);
			console.log(change.value);
			console.log(change.oldValue);
		});
	}
	...
	observablePerson = Observable.from(person);
	observablePerson.observe(personUIObserver);
	```
	
	Changes delivered always as an array. Changes MAY NOT be null. Changes MAY be an empty array.
	Each change is a defined, non-null object, having:
	- `type` - one of the following: 'insert', 'update', 'delete', 'shuffle' or 'reverse'
	- `path` - path to the changed property represented as an __Array__ of nodes (see examples below)
	- `value` - new value; not available in 'delete', 'shuffle' and 'reverse' changes
	- `oldValue` - old value; not available in 'insert', 'shuffle' or 'reverse' changes
	
- __`unobserve`__ - receives a _function/s_ which previously was/were registered as an observer/s and removes it/them. If _no arguments_ passed, all observers will be removed:
	```javascript
	...
	observablePerson.unobserve(personUIObserver);
	...
	observablePerson.unobserve();
	...
	```

# Examples

##### Objects
	```javascript
	var order = { type: 'book', pid: 102, ammount: 5, remark: 'remove me' },
		observableOrder;
	observableOrder = Observable.from(order);
	observableOrder.observe(changes => {
		changes.forEach(change => {
			console.log(change);
		});
	});
	observableOrder.ammount = 7;		// { type: 'update', path: ['ammount'], value: 7, oldValue: 5 }
	observableOrder.address = {
		street: 'Str 75',
		apt: 29
	};									// { type: "insert", path: ['address'], value: { ... } }
	observableOrder.address.apt = 30;	// { type: "update", path: ['address','apt'], value: 30, oldValue: 29 }
	delete observableOrder.remark;		// { type: "delete", path: ['remark'], oldValue: 'remove me' }
	```

##### Arrays
