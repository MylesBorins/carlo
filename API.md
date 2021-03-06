## API

#### launch([options])
- `options` <[Object]>  Set of configurable options to set on the app. Can have the following fields:
  - `width` <[number]> app window width in pixels.
  - `height` <[number]> app window height in pixels.
  - `bgcolor` <[string]> background color using hex notation, defaults to `#ffffff`.
  - `userDataDir` <[string]> Path to a [User Data Directory](https://chromium.googlesource.com/chromium/src/+/master/docs/user_data_dir.md). This folder is created upon the first app launch and contains user settings and Web storage data. Defaults to `.profile`.
  - `executablePath` <[string]> Path to a Chromium or Chrome executable to run instead of the automatically located Chrome. If `executablePath` is a relative path, then it is resolved relative to [current working directory](https://nodejs.org/api/process.html#process_process_cwd). Carlo is only guaranteed to work with the latest Chrome stable version.
- returns: <[Promise]<App>> Promise which resolves to the app instance.

Launches the browser.

#### App.serveFolder(folder, prefix)
- `folder, prefix` <[string]> Folder with web content to make available to Chrome.
- `prefix` <[string]> Prefix of the URL path to serve from the given folder.

Makes the content of the given folder available to the Chrome web app.

An example of adding a local `www` folder along with the `node_modules`:

`main.js`
```js
const carlo = require('carlo');
const crypto = require('crypto');

carlo.launch().then(async app => {
  app.on('exit', () => process.exit());
  app.serveFolder(__dirname + '/www');
  app.serveFolder(__dirname + '/node_modules', 'node_modules');
  await app.load('index.html');
});
```
`www/index.html`
```html
<script src="node_modules/..."></script> 
```

#### App.load(uri)
- `uri` <[string]> Path to the resource relative to the folder passed into `serveFolder`.

Navigates the Chrome web app to the given `uri`.

#### App.exposeFunction(name, carloFunction)
- `name` <[string]> Name of the function on the window object
- `carloFunction` <[function]> Callback function which will be called in Carlo's context.
- returns: <[Promise]>

The method adds a function called `name` on the page's `window` object.
When called, the function executes `carloFunction` in node.js and returns a [Promise] which resolves to the return value of `carloFunction`.

If the `carloFunction` returns a [Promise], it will be awaited.

> **NOTE** Functions installed via `App.exposeFunction` survive navigations.

An example of adding an `md5` function into the page:
```js
const carlo = require('carlo');
const crypto = require('crypto');

carlo.launch().then(async app => {
  app.on('exit', () => process.exit());
  app.serveFolder(__dirname);
  await app.exposeFunction('md5', text =>  // <-- expose function
    crypto.createHash('md5').update(text).digest('hex')
  );
  await app.load('index.html');
});
```

#### App.exposeObject(name, object)
- `name` <[string]> Name of the object
- `object` <[Object]> Object to make available to the web surface.

The method makes the given object available to Chrome.

> **NOTE** Communication between Chrome and Node takes place over the wired protocol, so the actual object Chrome is getting is a handle to the original object with the original methods available on that handle via RPC.

An example of adding a `world` object into the page:

`main.js`
```js
const carlo = require('carlo');

carlo.launch().then(async app => {
  app.serveFolder(__dirname);
  app.on('exit', () => process.exit());
  const world = new World();  // <-- create object
  world.on('happy', console.log);  // <-- subscribe to events
  await app.exposeObject('world', world);  // <-- expose it to the Web side
  await app.load('index.html');
});

class World extends EventEmitter {
  hello(name) {
    this.emit('happy', 'happy event');  // <-- emit event that is handled on the Web side.
    return 'Hello ' + name;  // <-- return value to the Web side.
  }
}
```

`index.html`
```html
<script>
async function start() {
  const world = await rpc.lookup('world');  // <- lookup service by name.
  world.on('happy', console.log);  // <-- remote objects can emit events.
  console.log(await world.hello());
}
</script> 
<body onload="start()"></body>
```

#### App.exposeFactory(factoryConstructor)
- `factoryConstructor` <[function]> Factory of the objects to make available to Chrome.

The method makes the given object factory available to Chrome.

An example of adding a `world` object into the page:

`main.js`
```js
const carlo = require('carlo');

carlo.launch().then(async app => {
  app.serveFolder(__dirname);
  app.on('exit', () => process.exit());
  await app.exposeFactory(World);  // <-- expose factory to the Web side
  await app.load('index.html');
});

class World {
  hello(name) {
    return 'Hello ' + name;  // <-- return value to the web side.
  }
}
```

`index.html`
```html
<script>
async function start() {
  const world = await rpc.create('World');  // <- create remote instance.
  console.log(await world.hello());  // <-- remote call.
  world.dispose();  // <-- release handle.
}
</script> 
<body onload="start()"></body>
```


#### App.evaluate(pageFunction, ...args)
- `pageFunction` <[function]|[string]> Function to be evaluated in the page context
- `...args` <...[Serializable]> Arguments to pass to `pageFunction`
- returns: <[Promise]<[Serializable]>> Promise which resolves to the return value of `pageFunction`

If the function passed to the `page.evaluate` returns a [Promise], then `page.evaluate` would wait for the promise to resolve and return its value.

If the function passed to the `App.evaluate` returns a non-[Serializable] value, then `page.evaluate` resolves to `undefined`.

Passing arguments to `pageFunction`:
```js
const result = await app.evaluate(x => {
  return Promise.resolve(8 * x);
}, 7);
console.log(result);  // prints "56"
```

A string can also be passed in instead of a function:
```js
console.log(await page.evaluate('1 + 2'));  // prints "3"
const x = 10;
console.log(await page.evaluate(`1 + ${x}`));  // prints "11"
```

#### App.exit()
- returns: <[Promise]>

Closes the browser window.


[Object]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object "Object"
[Promise]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise "Promise"
[Array]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array "Array"
[boolean]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures#Boolean_type "Boolean"
[Buffer]: https://nodejs.org/api/buffer.html#buffer_class_buffer "Buffer"
[function]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function "Function"
[number]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures#Number_type "Number"
[origin]: https://developer.mozilla.org/en-US/docs/Glossary/Origin "Origin"
[Promise]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise "Promise"
[string]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures#String_type "String"
[Serializable]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/stringify#Description "Serializable"
