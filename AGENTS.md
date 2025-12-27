---
title: Storage Manager
---

# Storage Manager

The Storage Manager is a built-in module that allows the persistence of your project data.

::: warning
This guide requires GrapesJS v0.19.\* or higher
:::

::: tip
Need more powerful and customizable storage options? [The Grapes Studio SDK has you covered.](https://app.grapesjs.com/docs-sdk/configuration/projects?utm_source=grapesjs-docs&utm_medium=tip#storage)
:::

[[toc]]

## Configuration

To change the default configurations you have to pass the `storageManager` property with the main configuration object.

```js
const editor = grapesjs.init({
  ...
  // Default configurations
  storageManager: {
    type: 'local', // Storage type. Available: local | remote
    autosave: true, // Store data automatically
    autoload: true, // Autoload stored data on init
    stepsBeforeSave: 1, // If autosave is enabled, indicates how many changes are necessary before the store method is triggered
    // ...
    // Default storage options
    options: {
      local: {/* ... */},
      remote: {/* ... */},
    }
  },
});
```

In case you don't need any persistence, you can disable the module in this way:

```js
const editor = grapesjs.init({
  ...
  storageManager: false,
});
```

Check the full list of available options here: [Storage Manager Config](https://github.com/GrapesJS/grapesjs/blob/master/src/storage_manager/config/config.ts)

## Project data

The project data is a JSON object containing all the necessary information (styles, pages, etc.) about your project in the editor and is the one used in the storage manager methods in order to store and load your project (locally or remotely in your DB/file).

::: tip
You can get the current state of the data and load it manually in this way:

```js
// Get current project data
const projectData = editor.getProjectData();
// ...
// Load project data
editor.loadProjectData(projectData);
```

:::

::: danger
You should only rely on the JSON project data in order to load your project properly in the editor.

The editor is able to parse and use HTML/CSS code, you can use it as part of your project initialization but never rely on it as a persitance layer in the load of projects as many information could be stripped off.
:::

<!-- If necessary, the JSON can be also enriched with your data of choice, but as the data schema might differ in time we highly recommend to store them in your domain specific keys-->

## Storage strategy

Project data are automatically stored every time the amount of changes (`editor.getDirtyCount()`) reaches the number of steps before save (`editor.Storage.getStepsBeforeSave()`). On any successful store of the data, the counter of changes is reset (`editor.clearDirtyCount()`).

::: tip
When necessary, you can always trigger store/load manually.

```js
// Store data
const storedProjectData = await editor.store();

// Load data
const loadedProjectData = await editor.load();
```

:::

## Setup local storage

By default, GrapesJS saves the data locally by using the built-in `local` storage which leverages [localStorage API].

The only option you might probably care for the local storage is the `key` used to store the data. If the user loads different projects in your application, you might probably need to differentiate the local storage by the ID of the project (the ID here is intended to be part of your application domain).

```js
// Get your project ID (eg. taken from the route)
const projectId = getProjectId();

const editor = grapesjs.init({
  ...
  storageManager: {
    type: 'local',
    options: {
      local: { key: `gjsProject-${projectId}` }
    }
  },
});
```

## Setup remote storage

Most commonly the data of the project might be saved remotely on your server (DB, file, etc.) therefore you need to setup your server-side API calls in order to store/load project data.

For a sake of simplicity we can setup a fake REST API server by relying on [json-server].

```sh
mkdir my-server
cd my-server
npm init
npm i json-server
echo '{"projects": [ {"id": 1, "data": {"assets": [], "styles": [], "pages": [{"component": "<div>Initial content</div>"}]} } ]}' > db.json
npx json-server --watch db.json
```

This will start up a local server with one single project available on `http://localhost:3000/projects/1`. The data will be updated on the `db.json` file.

Here below an example of how you would configure a `remote` storage in GrapesJS.

```js
const projectID = 1;
const projectEndpoint = `http://localhost:3000/projects/${projectID}`;

const editor = grapesjs.init({
  ...
  storageManager: {
    type: 'remote',
    stepsBeforeSave: 3,
    options: {
      remote: {
        urlLoad: projectEndpoint,
        urlStore: projectEndpoint,
        // The `remote` storage uses the POST method when stores data but
        // the json-server API requires PATCH.
        fetchOptions: opts => (opts.method === 'POST' ?  { method: 'PATCH' } : {}),
        // As the API stores projects in this format `{id: 1, data: projectData }`,
        // we have to properly update the body before the store and extract the
        // project data from the response result.
        onStore: data => ({ id: projectID, data }),
        onLoad: result => result.data,
      }
    }
  }
});
```

::: danger
Be sure to configure properly [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) on your server API. The [json-server] is not intended to be used in production and therefore enables all of them automatically.
:::

<div id="setup-the-server"></div>

### Server setup

Server configuration might differ case to case so usually, it's up to you to know how to configure it properly.
The default remote storage follows a simple REST API approach with project data exchanged as a JSON (`Content-Type: application/json`).

- On **load** (`GET` method), the JSON project data are expected to be returned directly in the response. As from example above, you can use `options.remote.onLoad` to extract the project data if the response contains other metadata.
- On **store** (`POST` method), the editor doesn't expect any particular result but only a valid response from the server (status code `200`).

<!-- ## Store and load templates

Even without a fully working endpoint, you can see what is sent from the editor by triggering the store and looking in the network panel of the inspector. GrapesJS sends mainly 4 types of parameters and it prefixes them with the `gjs-` key (you can disable it via `storageManager.id`). From the parameters, you will get the final result in 'gjs-html' and 'gjs-css' and this is what actually your end-users will gonna see on the final template/page. The other two, 'gjs-components' and 'gjs-styles', are a JSON representation of your template and therefore those should be used for the template editing. **So be careful**, GrapesJS is able to start from any HTML/CSS but use this approach only for importing already existent HTML templates, once the user starts editing, rely always on JSON objects because the HTML doesn't contain information about your components. You can achieve it in a pretty straightforward way and if you load your page by server-side you don't even need to load asynchronously your data (so you can turn off the `autoload`).

```js
// Lets say, for instance, you start with your already defined HTML template and you'd like to
// import it on fly for the user
const LandingPage = {
  html: `<div>...</div>`,
  css: null,
  components: null,
  style: null,
};
// ...
const editor = grapesjs.init({
  ...
  // If set to true, then the content within the wrapper element overrides the following config,
  fromElement: false,
  // The `components` accepts HTML string or a JSON of components
  // Here, at first, we check and use components if are already defined, otherwise
  // the HTML string gonna be used
  components: LandingPage.components || LandingPage.html,
  // We might want to make the same check for styles
  style: LandingPage.style || LandingPage.css,
  // As we already initialize the editor with the template we can skip the `autoload`
  storageManager: {
    ...
    autoload: false,
  },
});
``` -->

## Storage API

The Storage Manager module has also its own [set of APIs](/api/storage_manager.html) that allows you to extend and add new functionalities.

### Define new storage

Defining a new storage is a matter of passing of two asyncronous methods to the `editor.Storage.add` API. For a sake of simplicity, the example below illustrates the API usage for defining the `session` storage by using [sessionStorage API](https://developer.mozilla.org/en-US/docs/Web/API/Window/sessionStorage).

```js
const sessionStoragePlugin = (editor) => {
  // As sessionStorage is not an asynchronous API,
  // the `async` keyword could be skipped
  editor.Storage.add('session', {
    async load(options = {}) {
      return JSON.parse(sessionStorage.getItem(options.key));
    },

    async store(data, options = {}) {
      sessionStorage.setItem(options.key, JSON.stringify(data));
    }
  });
};

const editor = grapesjs.init({
  ...
  plugins: [sessionStoragePlugin],
  storageManager: {
    type: 'session',
    options: {
      session: { key: 'myKey' }
    }
  },
});
```

### Extend storage

Among other needs, you might need to use existing storages to combine them in a more complex use case.
For example, let's say we would like to mix the local and remote storages inside another one. This is how it would look like:

```js
const { Storage } = editor;

Storage.add('remote-local', {
  async store(data) {
    const remoteStorage = Storage.get('remote');

    try {
      await remoteStorage.store(data, Storage.getStorageOptions('remote'));
    } catch (err) {
      // On remote error, store data locally
      const localStorage = Storage.get('local');
      await localStorage.store(data, Storage.getStorageOptions('local'));
    }
  },

  async load() {
    // ...
  },
});
```

### Replace storage

You can also replace already defined storages with other implementations by passing the same storage type in the `Storage.add` method. You can switch, for example, the default `local`, which relies on [localStorage API], with something more scalable like [IndexedDB API].

It might also be possible that you're already using some HTTP client library (eg. [axios](https://github.com/axios/axios)) which handles for you all the necessary HTTP headers in your application (CSRF token, session data, etc.), so you can simply replace the default `remote` storage with your implementation of choice without caring about the default configurations.

```js
editor.Storage.add('remote', {
  async load() {
    return await axios.get(`projects/${projectId}`);
  },

  async store(data) {
    return await axios.patch(`projects/${projectId}`, { data });
  },
});
```

<!-- ### Examples

Here you can find some of the plugins extending the Storage Manager

* [grapesjs-indexeddb] - Storage wrapper for IndexedDB
* [grapesjs-firestore] - Storage wrapper for [Cloud Firestore](https://firebase.google.com/docs/firestore) -->

## Common use cases

### Skip initial load

In case you're using the `remote` storage, you might probably want to skip the initial remote call by loading the project instantly. In that case, you can specify the `projectData` on initialization.

```js
// Get the data before initializing the editor (eg. printed on server-side).
const projectData = {...};
// ...
grapesjs.init({
  // ...
  // If projectData is not defined we might want to load some initial data for the project.
  projectData: projectData || {
    pages: [
        {
          component: `
            <div class="test">Initial content</div>
            <style>.test { color: red }</style>
          `
        }
    ]
  },
  storageManager: {
    type: 'remote',
    // ...
  },
})
```

In case `projectData` is defined, the initial storage load will be automatically skipped.

### HTML code with project data

The project data doesn't contain HTML/CSS of your pages as its main purpose is to collect only the strictly necessary information.
In case you have a strict requirement to execute also other logic connected to the store of your project data (eg. deploy HTML/CSS result to the stage environment) you can enrich your remote calls by using the `onStore` option in the remote configuration.

```js
grapesjs.init({
  // ...
  storageManager: {
    type: 'remote',
    options: {
      remote: {
        // Enrich the store call
        onStore: (data, editor) => {
          const pagesHtml = editor.Pages.getAll().map((page) => {
            const component = page.getMainComponent();
            return {
              html: editor.getHtml({ component }),
              css: editor.getCss({ component }),
            };
          });
          return { id: projectID, data, pagesHtml };
        },
        // If on load, you're returning the same JSON from above...
        onLoad: (result) => result.data,
      },
    },
  },
});
```

### Inline project data

In might be a case where the editor is not connected to any storage but simply read/write the data in inputs placed in a form. For such a case you can create an inline storage.

```html
<form id="my-form">
  <input id="project-html" type="hidden" />
  <input id="project-data" type="hidden" value='{"pages": [{"component": "<div>Initial content</div>"}]}' />
  <div id="gjs"></div>
  <button type="submit">Submit</button>
</form>

<script>
  // Show data on submit
  document.getElementById('my-form').addEventListener('submit', (event) => {
    event.preventDefault();
    const projectDataEl = document.getElementById('project-data');
    const projectHtmlEl = document.getElementById('project-html');
    alert(`HTML: ${projectHtmlEl.value}\n------\nDATA: ${projectDataEl.value}`);
  });

  // Inline storage
  const inlineStorage = (editor) => {
    const projectDataEl = document.getElementById('project-data');
    const projectHtmlEl = document.getElementById('project-html');

    editor.Storage.add('inline', {
      load() {
        return JSON.parse(projectDataEl.value || '{}');
      },
      store(data) {
        const component = editor.Pages.getSelected().getMainComponent();
        projectDataEl.value = JSON.stringify(data);
        projectHtmlEl.value = `<html>
          <head>
            <style>${editor.getCss({ component })}</style>
          </head>
          ${editor.getHtml({ component })}
        <html>`;
      },
    });
  };

  // Init editor
  grapesjs.init({
    container: '#gjs',
    height: '500px',
    plugins: [inlineStorage],
    storageManager: { type: 'inline' },
  });
</script>
```

In the example above we're relying on two hidden inputs, one for containing the project data and the another one for the HTML/CSS.

## Events

For a complete list of available events, you can check it [here](/api/storage_manager.html#available-events).

[grapesjs-indexeddb]: https://github.com/GrapesJS/storage-indexeddb
[grapesjs-firestore]: https://github.com/GrapesJS/storage-firestore
[localStorage API]: https://developer.mozilla.org/it/docs/Web/API/Window/localStorage
[IndexedDB API]: https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API
[json-server]: https://github.com/typicode/json-server

---
title: Asset Manager
---

# Asset Manager

<p align="center"><img src="http://grapesjs.com/img/sc-grapesjs-assets-1.jpg" alt="GrapesJS - Asset Manager" align="center"/></p>

In this section, you will see how to setup and take the full advantage of built-in Asset Manager in GrapesJS. The Asset Manager is lightweight and implements just an `image` in its core, but as you'll see next it's easy to extend and create your own asset types.

::: tip
Want an asset manager that looks great out of the box? [Try the Grapes Studio SDK!](https://app.grapesjs.com/docs-sdk/configuration/assets/overview?utm_source=grapesjs-docs&utm_medium=tip)
:::

[[toc]]

## Configuration

To change default configurations you'd need to pass the `assetManager` property with the main configuration object

```js
const editor = grapesjs.init({
  ...
  assetManager: {
    assets: [...],
    ...
  }
});
```

You can update most of them later by using `getConfig` inside of the module

```js
const amConfig = editor.AssetManager.getConfig();
```

Check the full list of available options here: [Asset Manager Config](https://github.com/GrapesJS/grapesjs/blob/master/src/asset_manager/config/config.ts)

## Initialization

The Asset Manager is ready to work by default, so pass few URLs to see them loaded

```js
const editor = grapesjs.init({
  ...
  assetManager: {
    assets: [
     'http://placehold.it/350x250/78c5d6/fff/image1.jpg',
     // Pass an object with your properties
     {
       type: 'image',
       src: 'http://placehold.it/350x250/459ba8/fff/image2.jpg',
       height: 350,
       width: 250,
       name: 'displayName'
     },
     {
       // As the 'image' is the base type of assets, omitting it will
       // be set as `image` by default
       src: 'http://placehold.it/350x250/79c267/fff/image3.jpg',
       height: 350,
       width: 250,
       name: 'displayName'
     },
    ],
  }
});
```

If you want a complete list of available properties check out the source [AssetImage Model](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/asset_manager/model/AssetImage.ts)

The built-in Asset Manager modal is implemented and is showing up when requested. By default, you can make it appear by dragging Image Components in canvas, double clicking on images and all other stuff related to images (eg. CSS styling)

<img :src="$withBase('/assets-builtin-modal.png')">

<!--
Making the modal appear is registered with a command, so you can make it appear with this

```js
// This command shows only assets with `image` type
editor.runCommand('open-assets');
```


Worth noting that by doing this you can't do much with assets (if you double click on them nothing happens) and this is because you've not indicated any target. Try just to select an image in your canvas and run this in console (you should first make the editor globally available `window.editor = editor;` in your script)

```js
editor.runCommand('open-assets', {
  target: editor.getSelected()
});
```

Now you should be able to change the image of the component.
-->

## Uploading assets

The default Asset Manager includes also an easy to use, drag-and-drop uploader with a few UI helpers. The default uploader is already visible when you open the Asset Manager.

<img :src="$withBase('/assets-uploader.png')">

You can click on the uploader to select your files or just drag them directly from your computer to trigger the uploader. Obviously, before it will work you have to setup your server to receive your assets and specify the upload endpoint in your configuration

```js
const editor = grapesjs.init({
  ...
  assetManager: {
    ...
    // Upload endpoint, set `false` to disable upload, default `false`
    upload: 'https://endpoint/upload/assets',

    // The name used in POST to pass uploaded files, default: `'files'`
    uploadName: 'files',
    ...
  },
  ...
});
```

### Listeners

If you want to execute an action before/after the uploading process (eg. loading animation) or even on response, you can make use of these listeners

```js
// The upload is started
editor.on('asset:upload:start', () => {
  startAnimation();
});

// The upload is ended (completed or not)
editor.on('asset:upload:end', () => {
  endAnimation();
});

// Error handling
editor.on('asset:upload:error', (err) => {
  notifyError(err);
});

// Do something on response
editor.on('asset:upload:response', (response) => {
  ...
});
```

### Response

When the uploading is over, by default (via config parameter `autoAdd: 1`), the editor expects to receive a JSON of uploaded assets in a `data` key as a response and tries to add them to the main collection. The JSON might look like this:

```js
{
  data: [
    'https://.../image.png',
    // ...
    {
      src: 'https://.../image2.png',
      type: 'image',
      height: 100,
      width: 200,
    },
    // ...
  ];
}
```

<!-- Deprecated
### Setup Dropzone

There is another helper which improves the uploading of assets: A full-width editor dropzone.

<img :src="$withBase('/assets-full-dropzone.gif')">


All you have to do is to activate it and possibly set a custom content (you might also want to hide the default uploader)

```js
const editor = grapesjs.init({
  ...
  assetManager: {
    ...,
    dropzone: 1,
    dropzoneContent: '<div class="dropzone-inner">Drop here your assets</div>'
  }
});
``` -->

## Programmatic usage

If you need to manage your assets programmatically you have to use its [APIs][API-Asset-Manager]

```js
// Get the Asset Manager module first
const am = editor.AssetManager;
```

First of all, it's worth noting that Asset Manager keeps 2 collections of assets:

- **global** - which is just the one with all available assets, you can get it with `am.getAll()`
- **visible** - this is the collection which is currently rendered by the Asset Manager, you get it with `am.getAllVisible()`

This allows you to decide which assets to show and when. Let's say we'd like to have a category switcher, first of all you gonna add to the **global** collection all your assets (which you may already defined at init by `config.assetManager.assets = [...]`)

```js
am.add([
  {
    // You can pass any custom property you want
    category: 'c1',
    src: 'http://placehold.it/350x250/78c5d6/fff/image1.jpg',
  },
  {
    category: 'c1',
    src: 'http://placehold.it/350x250/459ba8/fff/image2.jpg',
  },
  {
    category: 'c2',
    src: 'http://placehold.it/350x250/79c267/fff/image3.jpg',
  },
  // ...
]);
```

Now if you call the `render()`, without an argument, you will see all the assets rendered

```js
// without any argument
am.render();

am.getAll().length; // <- 3
am.getAllVisible().length; // <- 3
```

Ok, now let's show only assets from the first category

```js
const assets = am.getAll();

am.render(assets.filter((asset) => asset.get('category') == 'c1'));

am.getAll().length; // Still have 3 assets
am.getAllVisible().length; // but only 2 are shown
```

You can also mix arrays of assets

```js
am.render([...assets1, ...assets2, ...assets3]);
```

<!--
If you want to customize the asset manager container you can get its `HTMLElement`

```js
am.getContainer().insertAdjacentHTML('afterbegin', '<div><button type="button">Click</button></div>');
```
-->

In case you want to update or remove an asset, you can make use of this methods

```js
// Get the asset via its `src`
const asset = am.get('http://.../img.jpg');

// Update asset property
asset.set({ src: 'http://.../new-img.jpg' });

// Remove asset
am.remove(asset); // or via src, am.remove('http://.../new-img.jpg');
```

For more APIs methods check out the [API Reference][API-Asset-Manager].

### Custom select logic

::: warning
This section is referring to GrapesJS v0.17.26 or higher
:::

You can open the Asset Manager with your own select logic.

```js
am.open({
  types: ['image'], // This is the default option
  // Without select, nothing will happen on asset selection
  select(asset, complete) {
    const selected = editor.getSelected();

    if (selected && selected.is('image')) {
      selected.addAttributes({ src: asset.getSrc() });
      // The default AssetManager UI will trigger `select(asset, false)`
      // on asset click and `select(asset, true)` on double-click
      complete && am.close();
    }
  },
});
```

## Customization

The default Asset Manager UI is great for simple things, but except the possibility to tweak some CSS style, adding more complex things like a search input, filters, etc. requires a replace of the default UI.

All you have to do is to indicate the editor your intent to use a custom UI and then subscribe to the `asset:custom` event that will give you all the information on any requested change.

```js
const editor = grapesjs.init({
  // ...
  assetManager: {
    // ...
    custom: true,
  },
});

editor.on('asset:custom', (props) => {
  // The `props` will contain all the information you need in order to update your UI.
  // props.open (boolean) - Indicates if the Asset Manager is open
  // props.assets (Array<Asset>) - Array of all assets
  // props.types (Array<String>) - Array of asset types requested, eg. ['image'],
  // props.close (Function) - A callback to close the Asset Manager
  // props.remove (Function<Asset>) - A callback to remove an asset
  // props.select (Function<Asset, boolean>) - A callback to select an asset
  // props.container (HTMLElement) - The element where you should append your UI
  // Here you would put the logic to render/update your UI.
});
```

Here an example of using custom Asset Manager with a Vue component.

<demo-viewer value="wbj4tmqk" height="500" darkcode/>

The example above is the right way if you need to replace the default UI, but as you might notice we append the mounted element to the container `props.container.appendChild(this.$el);`.
This is required as the Asset Manager, by default, is placed in the [Modal](/modules/Modal.html).

How to approach the case when your Asset Manager is a completely independent/external module (eg. should be shown in its own custom modal)? Not a problem, you can bind the Asset Manager state via `assetManager.custom.open`.

```js
const editor = grapesjs.init({
  // ...
  assetManager: {
    // ...
    custom: {
      open(props) {
        // `props` are the same used in `asset:custom` event
        // ...
        // Init and open your external Asset Manager
        // ...
        // IMPORTANT:
        // When the external library is closed you have to communicate
        // this state back to the editor, otherwise GrapesJS will think
        // the Asset Manager is still open.
        // example: myAssetManager.on('close', () => props.close())
      },
      close(props) {
        // Close the external Asset Manager
      },
    },
  },
});
```

It's important to declare also the `close` function, the editor should be able to close the Asset Manager via `am.close()`.

<!--
### Define new Asset type

Generally speaking, images aren't the only asset you'll use, it could be a `video`, `svg-icon`, or any other kind of `document`. Each type of asset is applied in our templates/pages differently. If you need to change the image of the Component all you need is another `url` in `src` attribute. However In case of a `svg-icon`, it's not the same, you might want to replace the element with a new `<svg>` content. Besides this you also have to deal with the presentation/preview of the asset inside the panel/modal. For example, showing a thumbnail for big images or the possibility to preview videos.


Defining a new asset means we have to push on top of the 'Stack of Types' a new layer. This stack is iterated over by the editor at any addition of the asset and tries to associate the correct type.

```js
am.add('https://.../image.png');
// string, url, ends with '.png' -> it's an 'image' type

am.add('<svg ...');
// string and starts with '<svg...' -> 'svg' type

am.add({type: 'video', src: '...'});
// an object, has 'video' type key -> 'video' type
```

It's up to you tell the editor how to recognize your type and for this purpose you should to use `isType()` method.
Let's see now an example of how we'd start to defining a type like `svg-icon`


```js
am.addType('svg-icon', {
  // `value` is for example the argument passed in `am.add(VALUE);`
  isType(value) {
    // The condition is intentionally simple
    if (value.substring(0, 5) == '<svg ') {
      return {
        type: 'svg-icon',
        svgContent: value
      };
    }
    // Maybe you pass the `svg-icon` object already
    else if (typeof value == 'object' && value.type == 'svg-icon') {
      return value;
    }
  }
})
```

With this snippet you can already add SVGs, the asset manager will assign the appropriate type.

```js
// Add some random SVG
am.add(`<svg viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
  <path d="M22,9 C22,8.4 21.5,8 20.75,8 L3.25,8 C2.5,8 2,8.4 2,9 L2,15 C2,15.6 2.5,16 3.25,16 L20.75,16 C21.5,16 22,15.6 22,15 L22,9 Z M21,15 L3,15 L3,9 L21,9 L21,15 Z"></path>
  <polygon points="4 10 5 10 5 14 4 14"></polygon>
</svg>`);
```


The default `open-assets` command shows only `image` assets, so to render `svg-icon` run this

```js
am.render(am.getAll().filter(
  asset => asset.get('type') == 'svg-icon'
));
```


You should see something like this

<img :src="$withBase('/assets-empty-view.png')">


The SVG asset is not rendered correctly and this is because we haven't yet configured its view

```js
am.addType('svg-icon', {
  view: {
    // `getPreview()` and `getInfo()` are just few helpers, you can
    // override the entire template with `template()`
    // Check the base `template()` here:
    // https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/asset_manager/view/AssetView.ts
    getPreview() {
      return `<div style="text-align: center">${this.model.get('svgContent')}</div>`;
    },
    getInfo() {
      // You can use model's properties if you passed them:
      // am.add({
      //  type: 'svg-icon',
      //  svgContent: '<svg ...',
      //  name: 'Some name'
      //  })
      //  ... then
      //  this.model.get('name');
      return '<div>SVG description</div>';
    },
  },
  isType(value) {...}
})
```


This is the result

<img :src="$withBase('/assets-svg-view.png')">


Now we have to deal with how to assign our `svgContent` to the selected element


```js
am.addType('svg-icon', {
  view: {
    // In our case the target is the selected component
    updateTarget(target) {
      const svg = this.model.get('svgContent');

      // Just to make things bit interesting, if it's an image type
      // I put the svg as a data uri, content otherwise
      if (target.get('type') == 'image') {
        // Tip: you can also use `data:image/svg+xml;utf8,<svg ...` but you
        // have to escape few chars
        target.set('src', `data:mime/type;base64,${btoa(svg)}`);
      } else {
        target.set('content', svg);
      }
    },
    ...
  },
  isType(value) {...}
})
```


Our custom `svg-icon` asset is ready to use. You can also add a `model` to the `addType` definition to group the business logic of your asset, but usually it's optional.


```js
// Just an example of model use
am.addType('svg-icon', {
  model: {
    // With `default` you define model's default properties
    defaults: {
      type:  'svg-icon',
      svgContent: '',
      name: 'Default SVG Name',
    },

    // You can call model's methods inside views:
    // const name = this.model.getName();
    getName() {
      return this.get('name');
    }
  },
  view: {...},
  isType(value) {...}
})
```


### Extend Asset Types

Extending asset types is basically the same as adding them, you can choose what type to extend and how.

```js
// svgIconType will contain the definition (model, view, isType)
const svgIconType = am.getType('svg-icon');

// Add new type and extend another one
am.addType('svg-icon2', {
  view: svgIconType.view.extend({
    getInfo() {
      return '<div>SVG2 description</div>';
    },
  }),
  // The `isType` is important, but if you omit it the default one will be added
  // isType(value) {
  //  if (value && value.type == id) {
  //    return {type: value.type};
  //  }
  // };
})
```


You can also extend the already defined types (to be sure to load assets with the old type extended create a plugin for your definitions)

```js
// Extend the original `image` and add a confirm dialog before removing it
am.addType('image', {
  // As you adding on top of an already defined type you can avoid indicating
  // `am.getType('image').view.extend({...` the editor will do it by default
  // but you can eventually extend some other type
  view: {
    // If you want to see more methods to extend check out
    // https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/asset_manager/view/AssetImageView.ts
    onRemove(e) {
      e.stopPropagation();
      const model = this.model;

      if (confirm('Are you sure?')) {
        model.collection.remove(model);
      }
    }
  },
})
``` -->

## Events

For a complete list of available events, you can check it [here](/api/assets.html#available-events).

[API-Asset-Manager]: /api/assets.html

---
title: Pages
---

# Pages

The Pages module in GrapesJS allows you to create a project with multiple pages. By default, one page is always created under the hood, even if you don't need multi-page support. This allows keeping the API consistent and easier to extend if you need to add multiple pages later.

::: warning
This guide is referring to GrapesJS v0.21.1 or higher
:::

::: tip
Want pages to just work, with a polished UI? [See how the Grapes Studio SDK does it!](https://app.grapesjs.com/docs-sdk/configuration/pages?utm_source=grapesjs-docs&utm_medium=tip)
:::
[[toc]]

## Initialization

The default editor initialization doesn't require any knowledge of pages and this was mainly done to avoid introducing breaking changes when the Pages module was introduced.

This is how a typical editor initialization looks like:

```js
const editor = grapesjs.init({
  container: '#gjs',
  height: '100%',
  storageManager: false,
  // CSS or a JSON of styles
  style: '.my-el { color: red }',
  // HTML string or a JSON of components
  components: '<div class="my-el">Hello world!</div>',
  // ...other config options
});
```

What actually is happening is that this configuration is automatically migrated to the Page Manager.

```js
const editor = grapesjs.init({
  container: '#gjs',
  height: '100%',
  storageManager: false,
  pageManager: {
    pages: [
      {
        // without an explicit ID, a random one will be created
        id: 'my-first-page',
        // CSS or a JSON of styles
        styles: '.my-el { color: red }',
        // HTML string or a JSON of components
        component: '<div class="my-el">Hello world!</div>',
      },
    ],
  },
});
```

::: warning
Worth noting the previous keys are `style` and `components`, where in pages you should use `styles` and `component`.
:::

As you might guess, this is how initializing the editor with multiple pages would look like:

```js
const editor = grapesjs.init({
  // ...
  pageManager: {
    pages: [
      {
        id: 'my-first-page',
        styles: '.my-page1-el { color: red }',
        component: '<div class="my-page1-el">Page 1</div>',
      },
      {
        id: 'my-second-page',
        styles: '.my-page2-el { color: blue }',
        component: '<div class="my-page2-el">Page 2</div>',
      },
    ],
  },
});
```

GrapesJS doesn't provide any default UI for the Page Manager but you can easily built one by leveraging its [APIs][Pages API]. Check the [Customization](#customization) section for more details on how to create your own Page Manager UI.

## Programmatic usage

If you need to manage pages programmatically you can use its [APIs][Pages API].

Below are some commonly used methods:

```js
// Get the Pages module first
const pages = editor.Pages;

// Get an array of all pages
const allPages = pages.getAll();

// Get currently selected page
const selectedPage = pages.getSelected();

// Add a new Page
const newPage = pages.add({
  id: 'new-page-id',
  styles: '.my-class { color: red }',
  component: '<div class="my-class">My element</div>',
});

// Get the Page by ID
const page = pages.get('new-page-id');

// Select another page by ID
pages.select('new-page-id');
// or by passing the Page instance
pages.select(page);

// Get the HTML/CSS code from the page component
const component = page.getMainComponent();
const htmlPage = editor.getHtml({ component });
const cssPage = editor.getCss({ component });

// Remove the Page by ID (or by Page instance)
pages.remove('new-page-id');
```

## Customization

By using the [Pages API] it's easy to create your own Page Manager UI.

The simpliest way is to subscribe to the catch-all `page` event, which is triggered on any change related to the Page module (not related to page content like components or styles), and update your UI accordingly.

```js
const editor = grapesjs.init({
  // ...
});

editor.on('page', () => {
  // Update your UI
});
```

In the example below you can see an quick implementation of the Page Manager UI.

<demo-viewer value="1y6bgeo3" height="500" darkcode/>

<!-- Demo template, here for reference
<style>
  .app-wrap {
    height: 100%;
    width: 100%;
    display: flex;
  }
  .editor-wrap  {
    widtH: 100%;
    height: 100%;
  }
  .pages-wrp, .pages {
    display: flex;
    flex-direction: column
  }
  .pages-wrp {
      background: #333;
      padding: 5px;
  }
  .add-page {
    background: #444444;
    color: white;
    padding: 5px;
    border-radius: 2px;
    cursor: pointer;
    white-space: nowrap;
    margin-bottom: 10px;
  }
  .page {
    background-color: #444;
    color: white;
    padding: 5px;
    margin-bottom: 5px;
    border-radius: 2px;
    cursor: pointer;

    &.selected {
      background-color: #706f6f
    }
  }

  .page-close {
    opacity: 0.5;
    float: right;
    background-color: #2c2c2c;
    height: 20px;
    display: inline-block;
    width: 17px;
    text-align: center;
    border-radius: 3px;

    &:hover {
      opacity: 1;
    }
  }
</style>

<div style="height: 100%">
  <div class="app-wrap">
    <div class="pages-wrp">
        <div class="add-page" @click="addPage">Add new page</div>
        <div class="pages">
          <div v-for="page in pages" :key="page.id" :class="{page: 1, selected: isSelected(page) }" @click="selectPage(page.id)">
            {{ page.get('name') || page.id }} <span v-if="!isSelected(page)" @click="removePage(page.id)" class="page-close">&Cross;</span>
          </div>
        </div>
    </div>
    <div class="editor-wrap">
      <div id="gjs"></div>
    </div>
  </div>
</div>

<script>
const editor = grapesjs.init({
  container: '#gjs',
  height: '100%',
  storageManager: false,
  plugins: ['gjs-blocks-basic'],
  pageManager: {
    pages: [{
      id: 'page-1',
      name: 'Page 1',
      component: '<div id="comp1">Page 1</div>',
      styles: `#comp1 { color: red }`,
    }, {
      id: 'page-2',
      name: 'Page 2',
      component: '<div id="comp2">Page 2</div>',
      styles: `#comp2 { color: green }`,
    }, {
      id: 'page-3',
      name: 'Page 3',
      component: '<div id="comp3">Page 3</div>',
      styles: `#comp3 { color: blue }`,
    }]
  },
});

const pm = editor.Pages;

const app = new Vue({
  el: '.pages-wrp',
  data: { pages: [] },
  mounted() {
    this.setPages(pm.getAll());
    editor.on('page', () => {
      this.pages = [...pm.getAll()];
    });
  },
  methods: {
    setPages(pages) {
      this.pages = [...pages];
    },
    isSelected(page) {
      return pm.getSelected().id == page.id;
    },
    selectPage(pageId) {
      return pm.select(pageId);
    },
    removePage(pageId) {
      return pm.remove(pageId);
    },
    addPage() {
      const len = pm.getAll().length;
      pm.add({
        name: `Page ${len + 1}`,
        component: '<div>New page</div>',
      });
    },
  }
});
</script>
-->

## Events

For a complete list of available events, you can check it [here](/api/pages.html#available-events).

[Pages API]: /api/pages.html

---
title: Block Manager
---

# Block Manager

<p align="center"><img src="http://grapesjs.com/img/sc-grapesjs-blocks-prp.jpg" alt="GrapesJS - Block Manager" height="400" align="center"/></p>

A [Block] is a simple object which allows the end-user to reuse your [Components]. It can be connected to a single [Component] or to a complex composition of them. In this guide, you will see how to setup and take full advantage of the built-in Block Manager UI in GrapesJS.
The default UI is a lightweight component with built-in Drag & Drop support, but as you'll see next in this guide, it's easy to extend and create your own UI manager.

::: warning
To get a better understanding of the content in this guide, we recommend reading [Components] first
:::
::: warning
This guide is referring to GrapesJS v0.17.27 or higher
:::

::: tip
Need a sleek block UI that’s easy to extend and customize? [Explore the Grapes Studio SDK!](https://app.grapesjs.com/docs-sdk/configuration/blocks?utm_source=grapesjs-docs&utm_medium=tip)
:::

[[toc]]

## Configuration

To change the default configurations you have to pass the `blockManager` property with the main configuration object.

```js
const editor = grapesjs.init({
  ...
  blockManager: {
    blocks: [...],
    ...
  }
});
```

Check the full list of available options here: [Block Manager Config](https://github.com/GrapesJS/grapesjs/blob/master/src/block_manager/config/config.ts)

## Initialization

By default, Block Manager UI is considered a hidden component. Currently, the GrapesJS core, renders default panels and buttons that allow you to show them, but in the long term, this is something that might change. Here below you can see how to init the editor without default panels and immediately rendered Block Manager UI.

::: tip
Follow the [Getting Started] guide in order to setup properly the editor with custom panels.
:::

```js
const editor = grapesjs.init({
  container: '#gjs',
  height: '100%',
  storageManager: false,
  panels: { defaults: [] }, // Avoid default panels
  blockManager: {
    appendTo: '.myblocks',
    blocks: [
      {
        id: 'image',
        label: 'Image',
        media: `<svg style="width:24px;height:24px" viewBox="0 0 24 24">
            <path d="M8.5,13.5L11,16.5L14.5,12L19,18H5M21,19V5C21,3.89 20.1,3 19,3H5A2,2 0 0,0 3,5V19A2,2 0 0,0 5,21H19A2,2 0 0,0 21,19Z" />
        </svg>`,
        // Use `image` component
        content: { type: 'image' },
        // The component `image` is activatable (shows the Asset Manager).
        // We want to activate it once dropped in the canvas.
        activate: true,
        // select: true, // Default with `activate: true`
      },
    ],
  },
});
```

## Block content types

The key of connecting blocks to components is the `block.content` property and what we passed in the example above is the [Component Definition]. This is the component-oriented way to create blocks and this is how we highly recommend the creation of your blocks.

### Component-oriented

The `content` can accept different formats, like an HTML string (which will be parsed and converted to components), but the component-oriented approach is the most precise as you can keep the control of your each dropped block in the canvas. Another advice is to keep your blocks' [Component Definition] as light as possible, if you're defining a lot of redundant properties, probably it makes sense to create another dedicated component, this might reduce the size of your project JSON file. Here an example:

```js
// Your components
editor.Components.addType('my-cmp', {
  model: {
    defaults: {
      prop1: 'value1',
      prop2: 'value2',
    }
  }
});
// Your blocks
[
  { ..., content: { type: 'my-cmp', prop1: 'value1-EXT', prop2: 'value2-EXT' } }
  { ..., content: { type: 'my-cmp', prop1: 'value1-EXT', prop2: 'value2-EXT' } }
  { ..., content: { type: 'my-cmp', prop1: 'value1-EXT', prop2: 'value2-EXT' } }
]
```

Here we're reusing the same component multiple times with the same set of properties (just an example, makes more sense with composed content of components), this can be reduced to something like this.

```js
// Your components
editor.Components.addType('my-cmp', { ... });
editor.Components.addType('my-cmp-alt', {
  extend: 'my-cmp',
  model: {
    defaults: {
      prop1: 'value1-EXT',
      prop2: 'value2-EXT'
    }
  }
});
// Your blocks
[
  { ..., content: { type: 'my-cmp-alt' } }
  { ..., content: { type: 'my-cmp-alt' } }
  { ..., content: { type: 'my-cmp-alt' } }
]
```

### HTML strings

Using HTML strings as `content` is not wrong, in some cases you don't need the finest control over components and want to leave the user full freedom on template composition (eg. static site builder editor with HTML copy-pasted from a framework like [Tailwind Components](https://tailwindcomponents.com/))

```js
// Your block
{
  // ...
  content: `<div class="el-X">
    <div class="el-Y el-A">Element A</div>
    <div class="el-Y el-B">Element B</div>
    <div class="el-Y el-C">Element C</div>
  </div>`;
}
```

In such a case, all rendered elements will be converted to the best suited default component (eg. `.el-Y` elements will be treated like `text` components). The user will be able to style and drag them with no particular restrictions.

Thanks to Components' [isComponet](Components.html#iscomponent) feature (executed post parsing), you're still able to bind your rendered elements to components and enforce an extra logic. Here an example how you would enforce all `.el-Y` elements to be placed only inside `.el-X` one, without touching any part of the original HTML used in the `content`.

```js
// Your component
editor.Components.addType('cmp-Y', {
  // Detect '.el-Y' elements
  isComponent: (el) => el.classList?.contains('el-Y'),
  model: {
    defaults: {
      name: 'Component Y', // Simple custom name
      draggable: '.el-X', // Add `draggable` logic
    },
  },
});
```

Another alternative is to leverage `data-gjs-*` attributes to attach properties to components.

::: tip
You can use most of the available [Component properties](/api/component.html#properties).
:::

```js
// -- [Option 1]: Declare type in HTML strings --
{
  // ...
  content: `<div class="el-X">
    <div data-gjs-type="cmp-Y" class="el-Y el-A">Element A</div>
    <div data-gjs-type="cmp-Y" class="el-Y el-B">Element B</div>
    <div data-gjs-type="cmp-Y" class="el-Y el-C">Element C</div>
  </div>`;
}
// Component
editor.Components.addType('cmp-Y', {
  // You don't need `isComponent` anymore as you declare types already on elements
  model: {
    defaults: {
      name: 'Component Y', // Simple custom name
      draggable: '.el-X', // Add `draggable` logic
    },
  },
});

// -- [Option 2]: Declare properties in HTML strings (less recommended option) --
{
  // ...
  content: `<div class="el-X">
    <div data-gjs-name="Component Y" data-gjs-draggable=".el-X" class="el-Y el-A">Element A</div>
    <div data-gjs-name="Component Y" data-gjs-draggable=".el-X" class="el-Y el-B">Element B</div>
    <div data-gjs-name="Component Y" data-gjs-draggable=".el-X" class="el-Y el-C">Element C</div>
  </div>`;
}
// No need for a custom component.
// You're already defining properties of each element.
```

Here we showed all the possibilities you have with HTML strings, but we strongly advise against the abuse of the `Option 2` and to stick to a more component-oriented approach.
Without a proper component type, not only your HTML will be harder to read, but all those defined properties will be "hard-coded" to a generic component of those elements. So, if one day you decide to "upgrade" the logic of the component (eg. `draggable: '.el-X'` -> `draggable: '.el-X, .el-Z'`), you won't be able.

### Mixed

It's also possible to mix components with HTML strings by passing an array.

```js
{
  // ...
  // Options like `activate`/`select` will be triggered only on the first component.
  activate: true,
  content: [
    { type: 'image' },
    `<div>Extra</div>`
  ]
}
```

## Important caveats

::: danger Read carefully
&nbsp;
:::

### Avoid non serializable properties

Don't put non serializable properties, like functions, in your blocks, keep them only in your components.

```js
// Your block
{
  content: {
    type: 'my-cmp',
    script() {...},
  },
}
```

This will work, but if you try to save and reload a stored project, those will disappear.

### Avoid styles

Don't put styles in your blocks, keep them always in your components.

```js
// Your block
{
  content: [
    // BAD: You risk to create conflicting styles
    { type: 'my-cmp', styles: '.cmp { color: red }' },
    { type: 'my-cmp', styles: '.cmp { color: green }' },

    // REALLY BAD: In case all related components are removed,
    // there is no safe way for the editor to know how to connect
    // and clean your styles.
    `<div class="el">Element</div>
    <div class="el2">Element 2</div>
    <style>
      .el { color: blue }
      .el2 { color: violet }
    </style>`,
  ],
}
```

<!-- Styles imported via component definition, by using the `styles` property, are connected to that specific component type. This allows the editor to remove automatically those styles if all related components are deleted. -->

With the component-oriented approach, you put yourself in a risk of conflicting styles and having a lot of useless redundant styles definitions in your project JSON.

With the HTML string, if you remove all related elements, the editor is not able to clean those styles from the project JSON, as there is no safe way to connect them.

## Programmatic usage

If you need to manage your blocks programmatically you can use its [APIs][Blocks API].

::: warning
All Blocks API methods update mainly your Block Manager UI, it has nothing to do with Components already dropped in the canvas.
:::

Below an example of commonly used methods.

```js
// Get the BlockManager module first
const bm = editor.Blocks; // `Blocks` is an alias of `BlockManager`

// Add a new Block
const block = bm.add('BLOCK-ID', {
  // Your block properties...
  label: 'My block',
  content: '...',
});

// Get the Block
const block2 = bm.get('BLOCK-ID-2');

// Update the Block properties
block2.set({
  label: 'Updated block',
});

// Remove the Block
const removedBlock = bm.remove('BLOCK-ID-2');
```

To know more about the available block properties, check the [Block API Reference][Block].

## Customization

The default Block Manager UI is great for simple things, but except the possibility to tweak some CSS style, adding more complex elements requires a replace of the default UI.

All you have to do is to indicate the editor your intent to use a custom UI and then subscribe to the `block:custom` event that will give you all the information on any requested change.

```js
const editor = grapesjs.init({
  // ...
  blockManager: {
    // ...
    custom: true,
  },
});

editor.on('block:custom', (props) => {
  // The `props` will contain all the information you need in order to update your UI.
  // props.blocks (Array<Block>) - Array of all blocks
  // props.dragStart (Function<Block>) - A callback to trigger the start of block dragging.
  // props.dragStop (Function<Block>) - A callback to trigger the stop of block dragging.
  // props.container (HTMLElement) - The default element where you can append your UI
  // Here you would put the logic to render/update your UI.
});
```

Here an example of using custom Block Manager with a Vue component.

<demo-viewer value="xyofm1qr" height="500" darkcode/>

From the demo above you can also see how we decided to hide our custom Block Manager and append it to the default container, but that is up to your preferences.

## Events

For a complete list of available events, you can check it [here](/api/block_manager.html#available-events).

<!--
## Custom render <Badge text="0.14.55+"/>

If you need to customize the aspect of each block you can pass a `render` callback function in the block definition. Let's see how it works.

As a first option, you can return a simple HTML string, which will be used as a new inner content of the block. As an argument of the callback you will get an object containing the following properties:

* `model` - Block's model (so you can use any passed property to it)
* `el` - Current rendered HTMLElement of the block
* `className` - The base class name used for blocks (useful if you follow BEM, so you can create classes like `${className}__elem`)

```js
blockManager.add('some-block-id', {
  label: `<div>
      <img src="https://picsum.photos/70/70"/>
      <div class="my-label-block">Label block</div>
    </div>`,
  content: '<div>...</div>',
  render: ({ model, className }) => `<div class="${className}__my-wrap">
      Before label
      ${model.get('label')}
      After label
    </div>`,
});
```

<img :src="$withBase('/block-custom-render.jpg')">


Another option would be to avoid returning from the callback (in that case nothing will be replaced) and edit only the current `el` block element

```js
blockManager.add('some-block-id', {
  // ...
  render: ({ el }) => {
    const btn = document.createElement('button');
    btn.innerHTML = 'Click me';
    btn.addEventListener('click', () => alert('Do something'))
    el.appendChild(btn);
  },
});
```
<img :src="$withBase('/block-custom-render2.jpg')">
-->

[Block]: /api/block.html
[Component]: /api/component.html
[Components]: Components.html
[Getting Started]: /getting-started.html
[Blocks API]: /api/block_manager.html
[Component Definition]: Components.html#component-definition

---
title: Commands
---

# Commands

A basic command in GrapesJS it's a simple function, but you will see in this guide how powerful they can be. The main goal of the Command module is to centralize functions and be easily reused across the editor. Another big advantage of using commands is the ability to track them, extend or even interrupt beside some conditions.

::: warning
This guide is referring to GrapesJS v0.14.61 or higher
:::

[[toc]]

## Basic configuration

You can create your commands already from the initialization step by passing them in the `commands.defaults` options:

```js
const editor = grapesjs.init({
  ...
  commands: {
    defaults: [
      {
        // id and run are mandatory in this case
        id: 'my-command-id',
        run() {
          alert('This is my command');
        },
      }, {
        id: '...',
        // ...
      }
    ],
  }
});
```

For all other available options check directly the [configuration source file](https://github.com/GrapesJS/grapesjs/blob/master/src/commands/config/config.ts).

Most commonly commands are created dynamically post-initialization, in that case, you'll need to use the [Commands API](/api/commands.html) (eg. this is what you need if you create a plugin)

```js
const commands = editor.Commands;
commands.add('my-command-id', (editor) => {
  alert('This is my command');
});

// or it would be the same...
commands.add('my-command-id', {
  run(editor) {
    alert('This is my command');
  },
});
```

As you see the definition is quite easy, you just add an ID and the callback function. The [Editor](/api/editor.html) instance is passed as the first argument to the callback so you can access any other module or API method.

Now if you want to call that command you should just run this

```js
editor.runCommand('my-command-id');
```

::: tip
The method `editor.runCommand` is an alias of `editor.Commands.run`
:::

You could also pass options if you need

```js
editor.runCommand('my-command-id', { some: 'option' });
```

Then you can get the same object as a third argument of the callback.

```js
commands.add('my-command-id', (editor, sender, options = {}) => {
  alert(`This is my command ${options.some}`);
});
```

The second argument, `sender`, just indicates who requested the command, in our case will be always the `editor`

Until now there is nothing exciting except a common entry point for functions, but we'll see later its real advantages.

## Default commands

GrapesJS comes along with some default set of commands and you can get a list of all currently available commands via `editor.Commands.getAll()`. This will give you an object of all available commands, so, also those added later, like via plugins. You can recognize default commands by their namespace `core:*`, we also recommend to use namespaces in your own custom commands, but let's get a look more in detail here:

- [`core:canvas-clear`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/commands/view/CanvasClear.ts) - Clear all the content from the canvas (HTML and CSS)
- [`core:component-delete`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/commands/view/ComponentDelete.ts) - Delete a component
- [`core:component-enter`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/commands/view/ComponentEnter.ts) - Select the first children component of the selected one
- [`core:component-exit`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/commands/view/ComponentExit.ts) - Select the parent component of the current selected one
- [`core:component-next`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/commands/view/ComponentNext.ts) - Select the next sibling component
- [`core:component-prev`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/commands/view/ComponentPrev.ts) - Select the previous sibling component
- [`core:component-outline`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/commands/view/SwitchVisibility.ts) - Enable outline border on components
- [`core:component-offset`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/commands/view/ShowOffset.ts) - Enable components offset (margins, paddings)
- [`core:component-select`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/commands/view/SelectComponent.ts) - Enable the process of selecting components in the canvas
- [`core:copy`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/commands/view/CopyComponent.ts) - Copy the current selected component
- [`core:paste`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/commands/view/PasteComponent.ts) - Paste copied component
- [`core:preview`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/commands/view/Preview.ts) - Show the preview of the template in canvas
- [`core:fullscreen`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/commands/view/Fullscreen.ts) - Set the editor fullscreen
- [`core:open-code`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/commands/view/ExportTemplate.ts) - Open a default panel with the template code
- [`core:open-layers`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/commands/view/OpenLayers.ts) - Open a default panel with layers
- [`core:open-styles`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/commands/view/OpenStyleManager.ts) - Open a default panel with the style manager
- [`core:open-traits`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/commands/view/OpenTraitManager.ts) - Open a default panel with the trait manager
- [`core:open-blocks`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/commands/view/OpenBlocks.ts) - Open a default panel with the blocks
- [`core:open-assets`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/commands/view/OpenAssets.ts) - Open a default panel with the assets
- `core:undo` - Call undo operation
- `core:redo` - Call redo operation
  <!-- * `core:canvas-move` -->
  <!-- * `core:component-drag` -->
  <!-- * `core:component-style-clear` -->
  <!-- tlb-clone tlb-delete tlb-move -->

## Stateful commands

As we've already seen the command is just a function and once executed nothing is left behind, but in some cases, we'd like to keep a track of executed commands. GrapesJS can handle by default this case and to enable it you just need to declare a command as an object with the `run` and `stop` methods

```js
commands.add('my-command-state', {
  run(editor) {
    alert('This command is now active');
  },
  stop(editor) {
    alert('This command is disabled');
  },
});
```

So if we now run `editor.runCommand('my-command-state')` the command will be registered as active. To check the state of the command you can use `commands.isActive('my-command-state')` or you can even get the list of all active commands via `commands.getActive()`, in our case the result would be something like this

```js
{
  ...
  'my-command-state': undefined
}
```

The key of the result object tells you the active command, the value is the last return of the `run` command, in our case is `undefined` because we didn't return anything, but it's up to your implementation decide what to return and if you actually need it.

```js
// Let's return something
...
run(editor) {
    alert('This command is now active');
    return {
      activated: new Date(),
    }
},
...
// Now instead of the `undefined` you'll see the object from the run method
```

To disable the command use `editor.stopCommand` method, so in our case it'll be `editor.stopCommand('my-command-state')`. As for the `runCommand` you can pass an options object as a second argument and use them in your `stop` method.

Once the command is active, if you try to run `editor.runCommand('my-command-state')` again you'll notice that the `run` is not triggering. This behavior is useful to prevent executing multiple times the activation process which might lead to an inconsistent state (think about, for instance, having a counter, which should be increased on `run` and decreased on `stop`). If you need to run a command multiple times probably you're dealing with a not stateful command, so try to use it without the `stop` method, but in case you're aware of your application state you can actually force the execution with `editor.runCommand('my-command-state', { force: true })`. The same logic applies to the `stopCommand` method.

<br/>

::: danger WARNING
&nbsp;
:::

If you deal with UI in your stateful commands, be careful to keep the state coherent with your logic. Let's take, for example, the use of a modal as an indicator of the command state.

```js
commands.add('my-command-modal', {
  run(editor) {
    editor.Modal.open({
      title: 'Modal example',
      content: 'My content',
    });
  },
  stop(editor) {
    editor.Modal.close();
  },
});
```

If you run it, close the modal (eg. by clicking the 'x' on top) and then try to run it again you'll see that the modal is not opening anymore. This happens because the command is still active (you should see it in `commands.getActive()`) and to fix it you have to disable it once the modal is closed.

```js
...
  run(editor) {
    editor.Modal.open({
      title: 'Modal example',
      content: 'My content',
    }).onceClose(() => this.stopCommand());
  },
...
```

In the example above, we make use of few helper methods from the Modal module (`onceClose`) and the command itself (`stopCommand`) but obviously, the logic might be different due to your requirements and specific UI.

## Extending

Another big advantage of commands is the possibility to easily extend or override them with another command.
Let's take a simple example

```js
commands.add('my-command-1', (editor) => {
  alert('This is command 1');
});
```

If you need to overwrite this command with another one, just add it and keep the same id.

```js
commands.add('my-command-1', (editor) => {
  alert('This is command 1 overwritten');
});
```

Let's see now instead how can we extend one

```js
commands.add('my-command-2', {
  someFunction1() {
    alert('This is function 1');
  },
  someFunction2() {
    alert('This is function 2');
  },
  run() {
    this.someFunction1();
    this.someFunction2();
  },
});
```

to extend it just use `extend` method by passing the id

```js
commands.extend('my-command-2', {
  someFunction2() {
    alert('This is function 2 extended');
  },
});
```

## Events

The Commands module offers also a set of events that you can use to intercept the command flow for adding more functionality or even interrupting it.

### Intercept run and stop

By using our previously created `my-command-modal` command let's see which events we can listen to

```js
editor.on('command:run:my-command-modal', () => {
  console.log('After `my-command-modal` execution');
  // For example, you can add extra content to the modal
  const modalContent = editor.Modal.getContentEl();
  modalContent.insertAdjacentHTML('beforeEnd', '<div>Some content</div>');
});
editor.on('command:run:before:my-command-modal', () => {
  console.log('Before `my-command-modal` execution');
});
// for stateful commands
editor.on('command:stop:my-command-modal', () => {
  console.log('After `my-command-modal` is stopped');
});
editor.on('command:stop:before:my-command-modal:before', () => {
  console.log('Before `my-command-modal` is stopped');
});
```

If you need, you can also listen to all commands

```js
editor.on('command:run', (commandId) => {
  console.log('Run', commandId);
});

editor.on('command:stop', (commandId) => {
  console.log('Stop', commandId);
});
```

### Interrupt command flow

Sometimes you might need to interrupt the execution of an existant command due to some condition. In that case, you have to use `command:run:before:{COMMAND-ID}` event and set to `true` the abort option

```js
const condition = 1;

editor.on('command:run:before:my-command-modal', (options) => {
  if (condition) {
    options.abort = true;
    console.log('Prevent `my-command-modal` from execution');
  }
});
```

## Conclusion

The Commands module is quite simple but, at the same time, really powerful if used correctly. So, if you're creating a plugin for GrapesJS, use commands as much as possible, this will allow higher reusability and control over your logic.

---
title: Components & JS
---

# Components & JS

In this guide you'll see how to attach component related scripts and deal with external JavaScript libraries (eg. counters, galleries, slideshows, etc.)

::: warning
This guide is referring to GrapesJS v0.16.34 or higher.<br><br>
To get a better understanding of the content in this guide, we recommend reading [Components](Components.html) and [Traits] first
:::

::: tip
Prefer a modern UI that's production-ready? [Get started with the Grapes Studio SDK!](https://app.grapesjs.com/docs-sdk/configuration/components/overview?utm_source=grapesjs-docs&utm_medium=tip)
:::

[[toc]]

## Basic scripts

Let's see how to create a component with scripts.

```js
// This is our custom script (avoid using arrow functions)
const script = function () {
  alert('Hi');
  // `this` is bound to the component element
  console.log('the element', this);
};

// Define a new custom component
editor.Components.addType('comp-with-js', {
  model: {
    defaults: {
      script,
      // Add some style, just to make the component visible
      style: {
        width: '100px',
        height: '100px',
        background: 'red',
      },
    },
  },
});

// Create a block for the component, so we can drop it easily
editor.Blocks.add('test-block', {
  label: 'Test block',
  attributes: { class: 'fa fa-text' },
  content: { type: 'comp-with-js' },
});
```

Now if you drag the new block inside the canvas you'll see an alert and the message in console, as you might expect.
One thing worth noting is that `this` context is bound to the component's element, so, for example, if you need to change its property, you'd do `this.innerHTML = 'inner content'`.

One thing you should take into account is how the script is bound to component once rendered in the canvas or in your final template. If now you check the generated HTML code in the editor (via Export button or `editor.getHtml()`), you might see something like this:

```html
<div id="c764"></div>
<script>
  var items = document.querySelectorAll('#c764');
  for (var i = 0, len = items.length; i < len; i++) {
    (function () {
      // START component code
      alert('Hi');
      console.log('the element', this);
      // END component code
    }).bind(items[i])();
  }
</script>
```

As you see the editor attaches a unique ID to all components with scripts and retrieves them via `querySelectorAll`. Dragging another `test-block` will generate this:

```html
<div id="c764"></div>
<div id="c765"></div>
<script>
  var items = document.querySelectorAll('#c764, #c765');
  for (var i = 0, len = items.length; i < len; i++) {
    (function () {
      // START component code
      alert('Hi');
      console.log('the element', this);
      // END component code
    }).bind(items[i])();
  }
</script>
```

## Important caveat

::: danger
Read carefully
:::

Keep in mind that all component scripts are executed inside the iframe of the canvas (isolated, just like your **final template**), and therefore are NOT part of the current `document`. All your external libraries (eg. those you're loading along with the editor) are not there (you'll see later how to manage components with dependencies).

That means **you can't use stuff outside of the function scope**. Take a look at this scenario:

```js
const myVar = 'John';

const script = function () {
  alert('Hi ' + myVar);
  console.log('the element', this);
};
```

This won't work. You'll get an error of the undefined `myVar`. The final HTML shows the reason more clearly

```html
<div id="c764"></div>
<script>
  var items = document.querySelectorAll('#c764');
  for (var i = 0, len = items.length; i < len; i++) {
    (function () {
      alert('Hi ' + myVar); // <- ERROR: undefined myVar
      console.log('the element', this);
    }).bind(items[i])();
  }
</script>
```

## Passing properties to scripts

Let's say you need to make the script behave differently, based on some component property, maybe also changable via [Traits] (eg. you want to initiliaze some library with different options).
You can do it by using the `script-props` property on your component.

```js
// The `props` argument will contain only the properties you have declared in `script-props`
const script = function (props) {
  const myLibOpts = {
    prop1: props.myprop1,
    prop2: props.myprop2,
  };
  alert('My lib options: ' + JSON.stringify(myLibOpts));
};

editor.Components.addType('comp-with-js', {
  model: {
    defaults: {
      script,
      // Define default values for your custom properties
      myprop1: 'value1',
      myprop2: '10',
      // Define traits, in order to change your properties
      traits: [
        {
          type: 'select',
          name: 'myprop1',
          changeProp: true,
          options: [
            { value: 'value1', name: 'Value 1' },
            { value: 'value2', name: 'Value 2' },
          ],
        },
        {
          type: 'number',
          name: 'myprop2',
          changeProp: true,
        },
      ],
      // Define which properties to pass (this will also reset your script on their changes)
      'script-props': ['myprop1', 'myprop2'],
      // ...
    },
  },
});
```

Now, if you try to change traits, you'll also see how the script will be triggered with the new updated properties.

## Dependencies

As we mentioned above, scripts are executed independently inside the canvas, without any dependencies, exactly as the final HTML generated by the editor.
If you want to make use of external libraries you have two approaches: component-related and template-related.

### Component related

The component related approach is definitely the best one, as the dependencies will be loaded dynamically and printed in the final HTML only when the component exists in the canvas.
All you have to do is to load your dependencies before executing your initialization script.

```js
const script = function (props) {
  const initLib = function () {
    const el = this;
    const myLibOpts = {
      prop1: props.myprop1,
      prop2: props.myprop2,
    };
    someExtLib(el, myLibOpts);
  };

  if (typeof someExtLib == 'undefined') {
    const script = document.createElement('script');
    script.onload = initLib;
    script.src = 'https://.../somelib.min.js';
    document.body.appendChild(script);
  } else {
    initLib();
  }
};
```

### Template related

A dependency might be used along all your components (eg. JQuery) so instead requiring it inside each script you might want to inject it directly inside the canvas:

```js
const editor = grapesjs.init({
  ...
  canvas: {
    scripts: ['https://.../somelib.min.js'],
    // The same would be for external styles
    styles: ['https://.../ext-style.min.css'],
  }
});
```

Keep in mind that the editor won't render those dependencies in the exported HTML (eg. via `editor.getHtml()`), so it's up to you how to include them in the final page where the final HTML is rendered.

[Traits]: Traits.html

---
title: Component Manager
---

# Component Manager

The Component is a base element of the template. It might be something simple and atomic like an image or a text box, but also complex structures, more probably composed by other components, like sections or pages. The concept of the component was made to allow the developer to bind different behaviors to different elements. For example, opening the Asset Manager on double click of the image is a custom behavior bound to that particular type of element.

::: warning
This guide is referring to GrapesJS v0.15.8 or higher
:::

::: tip
Skip the boilerplate—use a refined component editor out of the box. [Checkout the Grapes Studio SDK!](https://app.grapesjs.com/docs-sdk/configuration/components/overview?utm_source=grapesjs-docs&utm_medium=tip)
:::

[[toc]]

## How Components work?

Let's see in detail how components work by looking at all the steps from adding an HTML string to the editor.

::: tip
All the following snippets can be run directly in console from the [main demo](https://grapesjs.com/demo.html)
:::

This is how we can add new components to the canvas:

```js
// Append components directly to the canvas
editor.addComponents(`<div>
  <img src="https://path/image" />
  <span title="foo">Hello world!!!</span>
</div>`);

// or into some, already defined, component.
// For instance, appending to a selected component would be:
editor.getSelected().append(`<div>...`);

// Actually, editor.addComponents is an alias of...
editor.getWrapper().append(`<div>...`);
```

::: tip
If you need to append a component at a specific position, you can use `at` option. So, to add a component on top of all others (in the same collection) you would use

```js
component.append('<div>...', { at: 0 });
```

or in the middle

```js
const { length } = component.components();
component.append('<div>...', { at: parseInt(length / 2, 10) });
```

:::

### Component Definition

In the first step, the HTML string is parsed and transformed to what is called **Component Definition**, so the result of the input above would be:

```js
{
  tagName: 'div',
  components: [
    {
      type: 'image',
      attributes: { src: 'https://path/image' },
    }, {
      tagName: 'span',
      type: 'text',
      attributes: { title: 'foo' },
      components: [{
        type: 'textnode',
        content: 'Hello world!!!'
      }]
    }
  ]
}
```

The real **Component Definition** would be a little bit bigger so we've reduced the JSON for the sake of simplicity.

You might notice the result is similar to what is generally called a **Virtual DOM**, a lightweight representation of the DOM element. This actually helps the editor to keep track of the state of our elements and make performance-friendly changes/updates.
The meaning of properties like `tagName`, `attributes` and `components` are quite obvious, but what about `type`?! This particular property specifies the **Component Type** of our **Component Definition** (you check the list of default components [below](#built-in-component-types)) and if it's omitted, the default one will be used `type: 'default'`.
At this point, a good question would be, how the editor assigns those types by starting from a simple HTML string? This step is identified as **Component Recognition** and it's explained in detail in the next paragraph.

### Component Recognition and Component Type Stack

As we mentioned before, when you pass an HTML string as a component to the editor, that string is parsed and compiled to the [Component Definition] with a new `type` property. To understand what `type` should be assigned, for each parsed HTML Element, the editor iterates over all the defined components, called **Component Type Stack**, and checks via `isComponent` method (we will see it later) if that component type is appropriate for that element. The Component Type Stack is just a simple array of component types but what matters is the order of those types. Any new added custom **Component Type** (we'll see later how to create them) goes on top of the Component Type Stack and each element returned from the parser iterates the stack from top to bottom (the last element of the stack is the `default` one), the iteration stops once one of the component returns a truthy value from the `isComponent` method.

<img :src="$withBase('/component-type-stack.svg')" class="img-ctr">

::: tip
If you're importing big string chunks of HTML code you might want to improve the performances by skipping the parsing and the component recognition steps by passing directly Component Definition objects or using the JSX syntax.
Read [here](#setup-jsx-syntax) about how to setup JSX syntax parser
:::

### Component instance

Once the **Component Definition** is ready and the type is assigned, the [Component] instance can be created (known also as the **Model**). Let's step back to our previous example with the HTML string, the result of the `append` method is an array of added components.

```js
const component = editor.addComponents(`<div>
  <img src="https://path/image" />
  <span title="foo">Hello world!!!</span>
</div>`)[0];
```

The Component instance contains properties and methods which allows you to obtain its data and change them.
You can read properties with the `get` method, like, for example, the `type`

```js
const componentType = component.get('type'); // eg. 'image'
```

and to update properties you'd use `set`, which might change the way a component behaves in the canvas.

```js
// Make the component not draggable
component.set('draggable', false);
```

You can also use methods like `getAttributes`, `setAttributes`, `components`, etc.

```js
const innerComponents = component.components();
innerComponents.forEach((comp) => console.log(comp.toHTML()));
// Update component content
component.components(`<div>Component 1</div><div>Component 2</div>`);
```

Each component can define its own properties and methods but all of them will always extend, at least, the `default` one (then you will see how to create new custom components and how to extend the already defined) so it's good to check the [Component API] to see all available properties and methods.

The **main purpose of the Component** is to keep track of its data and to return them when necessary. One common thing you might need to ask from the component is to show its current HTML

```js
const componentHTML = component.toHTML();
```

This will return a string containing the HTML of the component and all of its children.
The component implements also `toJSON` methods so you can get its JSON structure in this way

```js
JSON.stringify(component);
```

::: tip
For storing/loading all the components you should rely on the [Storage Manager](/modules/storage.html)
:::

So, the **Component instance** is responsible for the **final data** (eg. HTML, JSON) of your templates. If you need, for example, to update/add some attribute in the HTML you need to update its component (eg. `component.addAttributes({ title: 'Title added' })`), so the Component/Model is your **Source of Truth**.

### Component rendering

Another important part of components is how they are rendered in the **canvas**, this aspect is handled by its **View**. It has nothing to do with the **final HTML data**, you can return a big `<div>...</div>` string as HTML of your component but render it as a simple image in the canvas (think about placeholders for complex/dynamic data).

By default, the view of components is automatically synced with the data of its models (you can't have a View without a Model). If you update the attribute of the component or append a new one as a child, the view will render it in the canvas.

Unfortunately, sometimes, you might need some additional logic to handle better the component result. Think about allowing a user build its `<table>` element, for this specific case you might want to add custom buttons in the canvas, so it'd be easier adding/removing columns/rows. To handle those cases you can rely on the View, where you can add additional DOM component, attach events, etc. All of this will be completely unrelated with the final HTML of the `<table>` (the result the user would expect) as it handled by the Model.
Once the component is rendered you can always access its View and the DOM element.

```js
const component = editor.getSelected();
// Get the View
const view = component.getView();
// Get the DOM element
const el = component.getEl();
```

Generally, the View is something you wouldn't need to change as the default one handles already the sync with the Model but in case you'd need more control over elements (eg. custom UI in canvas) you'll probably need to create a custom component type and extend the default View with your logic. We'll see later how to create custom Component Types.

So far we have seen the core concept behind Components and how they work. The **Model/Component** is the **source of truth** for the final code of templates (eg. the HTML export relies on it) and the **View/ComponentView** is what is used by the editor to **preview our components** to users in the canvas.

<!--
TODO
A more advanced use case of custom components is an implementation of a custom renderer inside of them
-->

## Built-in Component Types

Here below you can see the list of built-in component types, ordered by their position in the **Component Type Stack**

- [`cell`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/dom_components/model/ComponentTableCell.ts) - Component for handle `<td>` and `<th>` elements
- [`row`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/dom_components/model/ComponentTableRow.ts) - Component for handle `<tr>` elements
- [`table`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/dom_components/model/ComponentTable.ts) - Component for handle `<table>` elements
- [`thead`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/dom_components/model/ComponentTableHead.ts) - Component for handle `<thead>` elements
- [`tbody`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/dom_components/model/ComponentTableBody.ts) - Component for handle `<tbody>` elements
- [`tfoot`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/dom_components/model/ComponentTableFoot.ts) - Component for handle `<tfoot>` elements
- [`map`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/dom_components/model/ComponentMap.ts) - Component for handle `<a>` elements
- [`link`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/dom_components/model/ComponentLink.ts) - Component for handle `<a>` elements
- [`label`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/dom_components/model/ComponentLabel.ts) - Component for handle properly `<label>` elements
- [`video`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/dom_components/model/ComponentVideo.ts) - Component for videos
- [`image`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/dom_components/model/ComponentImage.ts) - Component for images
- [`script`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/dom_components/model/ComponentScript.ts) - Component for handle `<script>` elements
- [`svg`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/dom_components/model/ComponentSvg.ts) - Component for handle SVG elements
- [`comment`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/dom_components/model/ComponentComment.ts) - Component for comments (might be useful for email editors)
- [`textnode`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/dom_components/model/ComponentTextNode.ts) - Similar to the textnode in DOM definition, so a text element without a tag element.
- [`text`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/dom_components/model/ComponentText.ts) - A simple text component that can be edited inline
- [`wrapper`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/dom_components/model/ComponentWrapper.ts) - The canvas need to contain a root component, a wrapper, this component was made to identify it
- [`default`](https://github.com/GrapesJS/grapesjs/blob/dev/packages/core/src/dom_components/model/Component.ts) Default base component

## Define Custom Component Type

Now that we know how components work, we can start exploring the process of creating custom **Component Types**.

<u>The first rule of defining new component types is to place the code inside a plugin</u>. This is necessary if you want to load your custom types at the beginning, before any component initialization (eg. a template loaded from DB). The plugin is loaded before component fetch (eg. in case of Storage use) so it's a perfect place to define component types.

```js
const myNewComponentTypes = (editor) => {
  editor.DomComponents.addType(/* API for component type definition */);
};

const editor = grapesjs.init({
  container: '#gjs',
  // ...
  plugins: [myNewComponentTypes],
});
```

Let's say we want to make the editor understand and handle better `<input>` elements. This is how we would start defining our new component type

```js
editor.DomComponents.addType('my-input-type', {
  // Make the editor understand when to bind `my-input-type`
  isComponent: (el) => el.tagName === 'INPUT',

  // Model definition
  model: {
    // Default properties
    defaults: {
      tagName: 'input',
      draggable: 'form, form *', // Can be dropped only inside `form` elements
      droppable: false, // Can't drop other elements inside
      attributes: {
        // Default attributes
        type: 'text',
        name: 'default-name',
        placeholder: 'Insert text here',
      },
      traits: ['name', 'placeholder', { type: 'checkbox', name: 'required' }],
    },
  },
});
```

With this code, the editor will be able to understand simple text `<input>`s, assign default attributes and show some trait for a better attribute handling.

::: tip
To understand better how Traits work you should read its [dedicated page](Traits.html) but we highly suggest to read it after you've finished reading this one
:::

### isComponent

Let's see in detail what we have done so far. The first thing to notice is the `isComponent` function, we have already mentioned its usage in [this](#component-recognition-and-component-type-stack) section and we need it to make the editor understand `<input>` during the component recognition step.
It receives only the `el` argument, which is the parsed HTMLElement node and expects a truthy value in case the element satisfies your logic condition. So, if we add this HTML string as component

```js
// ...after editor initialization
editor.addComponents(`<input name="my-test" title="hello"/>`);
```

The resultant Component Definition will be

```js
{
  type: 'my-input-type',
  attributes: {
    name: 'my-test',
    title: 'hello',
  },
}
```

If you need you can also customize the resultant Component Definition by returning an object as the result:

```js
editor.DomComponents.addType('my-input-type', {
  isComponent: el => {
    if (el.tagName === 'INPUT') {
      // You should explicitly declare the type of your resultant
      // object, otherwise the `default` one will be used
      const result = { type: 'my-input-type' };

      if (/* some other condition */) {
        result.attributes = { title: 'Hi' };
      }

      return result;
    }
  },
  // ...
});
```

::: danger
Keep the `isComponent` function as simple as possible
:::

**Be aware** that this method will probably receive ANY parsed element from your canvas (eg. on load or on add) and not all the nodes have the same interface (eg. properties/methods).
If you do this:

```js
// ...
// Print elements
isComponent: (el) => {
  console.log(el);
  return el.tagName === 'INPUT';
},
  // ...
  editor.addComponents(`<div>
  I'm a text node
  <!-- I'm a comment node -->
  <img alt="Image here"/>
  <input/>
</div>`);
```

You will see printing all the nodes, so doing something like this `el.getAttribute('...')` in your `isComponent` (which will work on the `div` but not on the `text node`), without an appropriate check, will break the code.

It's also important to understand that `isComponent` is executed only if the parsing is required (eg. by adding components as HTML string or initializing the editor with `fromElement`). In case the type is already defined, there is no need for the `isComponent` to be executed.
Let's see some examples:

```js
// isComponent will be executed on some-element
editor.addComponents('<some-element>...</some-element>');

// isComponent WON'T be executed on OBJECTS
// If the object has no `type` key, the `default` one will be used
editor.addComponents({
  type: 'some-component',
});

// isComponent WON'T be executed as we're forcing the type
editor.addComponents('<some-element data-gjs-type="some-component">...');
```

If you define the Component Type without using `isComponent`, the only way for the editor to see that component will be with an explicitly declared type (via an object `{ type: '...' }` or using `data-gjs-type`).

### Model

Now that we got how `isComponent` works we can start to explore the `model` property.
The `model` is probably the one you'll use the most as is what is used for the description of your component and the first thing you can see is its `defaults` key which just stands for _default component properties_ and it reflects the already described [Component Definition]

The model defines also what you will see as the resultant HTML (the export code) and you've probably noticed the use of `tagName` (if not specified the `div` will be used) and `attributes` properties on the model.

One another important property (not used in our input component integration because `<input/>` doesn't need it) might be `components`, which defines default internal components

```js
defaults: {
  tagName: 'div',
  attributes: { title: 'Hello' },
  // Can be a string
  components: `
    <h1>Header test</h1>
    <p>Paragraph test</p>
  `,
  // A component definition
  components: {
    tagName: 'h1',
    components: 'Header test',
  },
  // Array of strings/component definitions
  components: [
    {
      tagName: 'h1',
      components: 'Header test',
    },
    '<p>Paragraph test</p>',
  ],
  // Or a function, which get as an argument the current
  // model and expects as the return one of the possible
  // values described above
  components: model => {
    return `<h1>Header test: ${model.get('type')}</h1>`;
  },
}
```

#### Read and update the model

You can read and update the model properties wherever you have the reference to it. Here some references to the most useful API

```js
// let's use the selected component
const modelComponent = editor.getSelected();

// Get all the model properties
const props = modelComponent.props();

// Get a single property
const tagName = modelComponent.get('tagName');

// Update a single property
modelComponent.set('tagName', '...');

// Update multiple properties
modelComponent.set({
  tagName: '...',
  // ...
});

// Some helpers

// Get all attributes
const attrs = modelComponent.getAttributes();

// Add attributes
modelComponent.addAttributes({ title: 'Test' });

// Replace all attributes
modelComponent.setAttributes({ title: 'Test' });

// Get the collection of all inner components
modelComponent.components().forEach((inner) => console.log(inner.props()));

// Update the inner content with an HTML string/Component Definitions
const addedComponents = modelComponent.components(`<div>...</div>`);

// Find components by query string
modelComponent.find(`.query-string[example=value]`).forEach((inner) => console.log(inner.props()));
```

You'll notice that, on any change, the component in the canvas and its export code are changing accordingly

:::tip
To know all the available methods/properties check the [Component API]
:::

#### Listen to property changes

If you need to accomplish some kind of action on some property change you can set up listeners in the `init` method

```js
editor.DomComponents.addType('my-input-type', {
  // ...
  model: {
    defaults: {
      // ...
      someprop: 'initial value',
    },

    init() {
      this.on('change:someprop', this.handlePropChange);
      // Listen to any attribute change
      this.on('change:attributes', this.handleAttrChange);
      // Listen to title attribute change
      this.on('change:attributes:title', this.handleTitleChange);
    },

    handlePropChange() {
      const { someprop } = this.props();
      console.log('New value of someprop: ', someprop);
    },

    handleAttrChange() {
      console.log('Attributes updated: ', this.getAttributes());
    },

    handleTitleChange() {
      console.log('Attribute title updated: ', this.getAttributes().title);
    },
  },
});
```

You'll find other lifecycle methods, like `init`, [below](#lifecycle-hooks)

Now let's go back to our input component integration and see another useful part for the component customization

### View

Generally, when you create a component in GrapesJS you expect to see in the canvas the preview of what you've defined in the model. Indeed, by default, the editor does the exact thing and updates the element in the canvas when something in the model changes (eg. attributes, tag, etc.) to obtain the classic **WYSIWYG** (What You See Is What You Get) experience. Unfortunately, not always the simplest thing is the right one, by building components for the builder you will notice that sometimes you'll need something more:

- You want to improve the experience of editing of the component.
  A perfect example is the TextComponent, its view is enriched with a built-in RTE (Rich Text Editor) which enables the user to edit the text faster by double-clicking on it.

  So you'll probably feel a need adding actions to react on some DOM events or even custom UI elements (eg. buttons) around the component.

- The DOM representation of the component acts differently from what you'd expect, so you need to change some behavior.
  An example could be a VideoComponent which, for example, is loaded from Youtube via iframe. Once the iframe is loaded, everything inside it is in a different context, the editor is not able to see it, indeed if you point your cursor on the iframe you'll interact with the video and not the editor, so you can't even select your component. To workaround this "issue", in the render, we disabled the pointer interaction with the iframe and wrapped it with another element (without the wrapper the editor would select the parent component). Obviously, all of these changes have nothing to do with the final code, the result will always be a simple iframe

- You need to customize the content or fill it with some data from the server

For all of these cases, you can use the `view` in your Component Type Definition. The `<input>` component is probably not the best use case for this scenario but we'll try to cover most of the cases with an example below

```js
editor.DomComponents.addType('my-input-type', {
  // ...
  model: {
    // ...
  },
  view: {
    // Be default, the tag of the element is the same of the model
    tagName: 'div',

    // Add easily component specific listeners with `events`
    // Being component specific (eg. you can't attach here listeners to window)
    // you don't need to care about removing them when the component is removed,
    // they will be managed automatically by the editor
    events: {
      click: 'clickOnElement',
      // You can also make use of event delegation
      // and listen to events bubbled from some inner element
      'dblclick .inner-el': 'innerElClick',
    },

    innerElClick(ev) {
      ev.stopPropagation();
      // ...

      // If you need you can access the model from any function in the view
      this.model.components('Update inner components');
    },

    // On init you can create listeners, like in the model, or start some other
    // function at the beginning
    init({ model }) {
      // Do something in view on model property change
      this.listenTo(model, 'change:prop', this.handlePropChange);

      // If you attach listeners on outside objects remember to unbind
      // them in `removed` function in order to avoid memory leaks
      this.onDocClick = this.onDocClick.bind(this);
      document.addEventListener('click', this.onDocClick);
    },

    // Callback triggered when the element is removed from the canvas
    removed() {
      document.removeEventListener('click', this.onDocClick);
    },

    // Do something with the content once the element is rendered.
    // The DOM element is passed as `el` in the argument object,
    // but you can access it from any function via `this.el`
    onRender({ el }) {
      const btn = document.createElement('button');
      btn.value = '+';
      // This is just an example, AVOID adding events on inner elements,
      // use `events` for these cases
      btn.addEventListener('click', () => {});
      el.appendChild(btn);
    },

    // Example of async content
    async onRender({ el, model }) {
      const asyncContent = await fetchSomething({
        someDataFromModel: model.get('someData'),
      });
      // Remember, these changes exist only inside the editor canvas
      // None of the DOM change is stored in your template data,
      // if you need to store something, update the model properties
      el.appendChild(asyncContent);
    },
  },
});
```

## Update Component Type

Updating component types is quite easy, let's see how:

```js
const domc = editor.DomComponents;

domc.addType('some-component', {
  // You can update the isComponent logic or leave the one from `some-component`
  // isComponent: (el) => false,

  // Update the model, if you need
  model: {
    // The `defaults` property is handled differently
    // and will be merged with the old `defaults`
    defaults: {
      tagName: '...', // Overrides the old one
      someNewProp: 'Hello', // Add new property
    },
    init() {
      // Ovverride `init` function in `some-component`
    },
  },

  // Update the view, if you need
  view: {},
});
```

### Extend Component Type

Sometimes you would need to create a new type by extending another one. Just use `extend` and `extendView` indicating the component to extend.

```js
comps.addType('my-new-component', {
  isComponent: el => {/* ... */},
  extend: 'other-defined-component',
  model: { ... }, // Will extend the model from 'other-defined-component'
  view: { ... }, // Will extend the view from 'other-defined-component'
});
```

```js
comps.addType('my-new-component', {
  isComponent: el => {/* ... */},
  extend: 'other-defined-component',
  model: { ... }, // Will extend the model from 'other-defined-component'
  extendView: 'other-defined-component-2',
  view: { ... }, // Will extend the view from 'other-defined-component-2'
});
```

### Extend parent functions

When you need to reuse functions, from the parent you're extending, you can avoid writing this:

```js
domc.getType('parent-type').model.prototype.init.apply(this, arguments);
```

by using `extendFn` and `extendFnView` options:

```js
domc.addType('new-type', {
  extend: 'parent-type',
  extendFn: ['init'], // array of model functions to extend from `parent-type`
  model: {
    init() {
      // do something
    },
  },
});
```

The same would be for the view by using `extendFnView`

:::tip
If you need you can also get all the current component types by using `getTypes`

```js
editor.DomComponents.getTypes().forEach((compType) => console.log(compType.id));
```

:::

## Lifecycle Hooks

Each component triggers different lifecycle hooks, which allows you to add custom actions at their specific stages.
We can distinguish 2 different types of hooks: **global** and **local**.
You define **local** hooks when you create/extend a component type (usually via some `model`/`view` method) and the reason is to react to an event of that
particular component type. Instead, the **global** one, will be called indistinctly on any component (you listen to them via `editor.on`) and you can make
use of them for a more generic use case or also listen to them inside other components.

Let's see below the flow of all hooks:

- **Local hook**: `model.init()` method, executed once the model of the component is initialized
- **Global hook**: `component:create` event, called right after `model.init()`. The model is passed as an argument to the callback function.
  Es. `editor.on('component:create', model => console.log('created', model))`
- **Local hook**: `view.init()` method, executed once the view of the component is initialized
- **Local hook**: `view.onRender()` method, executed once the component is rendered on the canvas
- **Global hook**: `component:mount` event, called right after `view.onRender()`. The model is passed as an argument to the callback function.
- **Local hook**: `model.updated()` method, executes when some property of the model is updated.
- **Global hook**: `component:update` event, called after `model.updated()`. The model is passed as an argument to the callback function.
  You can also listen to specific property change via `component:update:{propertyName}`
- **Local hook**: `model.removed()` method, executed when the component is removed.
- **Global hook**: `component:remove` event, called after `model.removed()`. The model is passed as an argument to the callback function.

Below you can find an example usage of all the hooks

```js
editor.DomComponents.addType('test-component', {
  model: {
    defaults: {
      testprop: 1,
    },
    init() {
      console.log('Local hook: model.init');
      this.listenTo(this, 'change:testprop', this.handlePropChange);
      // Here we can listen global hooks with editor.on('...')
    },
    updated(property, value, prevValue) {
      console.log('Local hook: model.updated', 'property', property, 'value', value, 'prevValue', prevValue);
    },
    removed() {
      console.log('Local hook: model.removed');
    },
    handlePropChange() {
      console.log('The value of testprop', this.get('testprop'));
    },
  },
  view: {
    init() {
      console.log('Local hook: view.init');
    },
    onRender() {
      console.log('Local hook: view.onRender');
    },
  },
});

// A block for the custom component
editor.BlockManager.add('test-component', {
  label: 'Test Component',
  content: '<div data-gjs-type="test-component">Test Component</div>',
});

// Global hooks
editor.on(`component:create`, (model) => console.log('Global hook: component:create', model.get('type')));
editor.on(`component:mount`, (model) => console.log('Global hook: component:mount', model.get('type')));
editor.on(`component:update:testprop`, (model) =>
  console.log('Global hook: component:update:testprop', model.get('type')),
);
editor.on(`component:remove`, (model) => console.log('Global hook: component:remove', model.get('type')));
```

## Components & CSS

::: warning
This section is referring to GrapesJS v0.17.27 or higher
:::

If you need to add component-related styles, you can do it via `styles` property.

```js
domc.addType('component-css', {
  model: {
    defaults: {
      attributes: { class: 'cmp-css' },
      components: `
        <span>Component with styles<span>
        <div class="cmp-css-a">Component A</div>
        <div class="cmp-css-b">Component B</div>
      `,
      styles: `
        .cmp-css { color: red }
        .cmp-css-a { color: green }
        .cmp-css-b { color: blue }

        @media (max-width: 992px) {
          .cmp-css{ color: darkred; }
          .cmp-css-a { color: darkgreen }
          .cmp-css-b { color: darkblue }
        }
      `,
    },
  },
});
```

This approach allows the editor to group these styles ([CssRule] instances) and remove them accordingly in case all references of the same component are removed.

::: danger Important caveat
&nbsp;
:::

In the example above we used one custom component and default sub-components. Styles are declared on our custom component only, that means if you remove all `.cmp-css-a` and `.cmp-css-b` instances from the canvas, their CssRules will still be stored in the project (**here we're not talking about the CSS export, which is able to skip not used rules, but instances stored in your project JSON**).

The cleanest approach would be to follow component-oriented styling, where you declare styles only in the scope of the component itself. This is how it would look like with the example above.

```js
domc.addType('cmp-a', {
  model: {
    defaults: {
      attributes: { class: 'cmp-css-a' },
      components: 'Component A',
      styles: `
        .cmp-css-a { color: green }
        @media (max-width: 992px) {
          .cmp-css-a { color: darkgreen }
        }
      `,
    },
  },
});
domc.addType('cmp-b', {
  model: {
    defaults: {
      attributes: { class: 'cmp-css-b' },
      components: 'Component B',
      styles: `
        .cmp-css-b { color: blue }
        @media (max-width: 992px) {
          .cmp-css-b { color: darkblue }
        }
      `,
    },
  },
});
domc.addType('component-css', {
  model: {
    defaults: {
      attributes: { class: 'cmp-css' },
      components: ['<span>Component with styles<span>', { type: 'cmp-a' }, { type: 'cmp-b' }],
      styles: `
        .cmp-css { color: red }
        @media (max-width: 992px) {
          .cmp-css{ color: darkred; }
        }
      `,
    },
  },
});
```

::: tip Component-first styling
By default, when you select a component in the canvas and apply styles on it, changes will be applied on its existent classes. This will result on changing of all the components with those applied classes. If you need the style to be applied only on the specific selected component you have to select componentFirst strategy in this way.

```js
grapesjs.init({
  ...
  selectorManager: {
    componentFirst: true,
  },
})
```

:::

### External CSS

If you need to load external component-specific CSS, you have to rely on the `script` property. For more details please refer to [Components & JS](Components-js.html).

## Components & JS

If you want to know how to create Components with javascript attached (eg. counters, galleries, slideshows, etc.) check the dedicated page
[Components & JS](Components-js.html)

## Tips

### JSX syntax

If you're importing big chunks of HTML string into the editor (eg. defined via Blocks) JSX might be a great compromise between performances and code readibility as it allows you to skip the parsing and the component recognition steps by keeping the HTML syntax.
By default, GrapesJS understands objects generated from React JSX preset, so, if you're working in the React app probably you're already using JSX and you don't need to do anything else, your environment is already configured to parse JSX in javascript files.

So, instead of writing this:

```js
// I'm adding a string, so the parsing and the component recognition steps will be executed
editor.addComponents(`<div>
  <span data-gjs-type="custom-component" data-gjs-prop="someValue" title="foo">
    Hello!
  </span>
</div>`);
```

or this

```js
// I'm passing the Component Definition, so heavy steps will be skipped but the code is less readable
editor.addComponents({
  tagName: 'div',
  components: [
    {...}
  ],
});
```

you can use this format

```js
editor.addComponents(
  <div>
    <custom-component data-gjs-prop="someValue" title="foo">
      Hello!
    </custom-component>
  </div>,
);
```

Another cool feature you will get by using JSX is the ability to pass component types as element tags `<custom-component>` instead of `data-gjs-type="custom-component"`

#### Setup JSX syntax

For those who are not using React you have the following options:

- GrapesJS has an option, `config.domComponents.processor`, thats allows you to easily implement other JSX presets. This scenario is useful if you work with a framework different from React but that uses JSX (eg. Vue). In that case, the result object from JSX pragma function (React uses `React.createElement`) will be different (you can log the JSX to see the result object) and you have to transform that in GrapesJS [Component Definition] object. Below an example of usage

```js
grapesjs.init({
  // ...
  domComponents: {
    processor: (obj) => {
     if (obj.$$typeof) { // eg. this is a React Element
        const compDef = {
         type: obj.type,
         components: obj.props.children,
         ...
        };
        ...
        return compDef;
     }
    }
  }
})
```

- In case you need to support JSX from scratch (you don't use a framework which supports JSX) you have, at first, implement the parser which transforms JSX in your files in something JS-readable.

For Babel users, it's just a matter of adding few plugins: `@babel/plugin-syntax-jsx` and `@babel-plugin-transform-react`. Then update your `.babelrc` file

```json
{
  “plugins”: [
    “@babel/plugin-syntax-jsx”,
    “@babel/plugin-transform-react-jsx”
  ]
}
```

You can also customize the pragma function which executes the transformation `[“@babel/plugin-transform-react-jsx”, { “pragma”: “customCreateEl” }]`, by default `React.createElement` is used (you'll need a React instance available in the file to make it work).

A complete example of this approach can be found [here](https://codesandbox.io/s/x07xf)

[Component Definition]: #component-definition
[Component]: /api/component.html
[CssRule]: /api/css_rule.html
[Component API]: /api/component.html

---
title: Plugins
---

# Plugins

Creating plugins in GrapesJS is pretty straightforward and here you'll get how to achieve it.

::: warning
This guide is referring to GrapesJS v0.21.2 or higher
:::

::: tip
Looking for plugins that are tested, verified, and built to scale? [Browse them all in the Grapes Studio SDK!](https://app.grapesjs.com/docs-sdk/plugins/overview?utm_source=grapesjs-docs&utm_medium=tip)
:::

[[toc]]

## Basic plugin

Plugins are simple functions that are run when the editor is initialized.

```js
function myPlugin(editor) {
  // Use the API: https://grapesjs.com/docs/api/
  editor.Blocks.add('my-first-block', {
    label: 'Simple block',
    content: '<div class="my-block">This is a simple block</div>',
  });
}

const editor = grapesjs.init({
  container: '#gjs',
  plugins: [myPlugin],
});
```

This means plugins can be moved to separate folders to keep thing cleaner or imported from NPM.

```js
import myPlugin from './plugins/myPlugin';
import npmPackage from '@npm/package';

const editor = grapesjs.init({
  container: '#gjs',
  plugins: [myPlugin, npmPackage],
});
```

<!--
## Named plugin

If you're distributing your plugin globally, you may want to make a named plugin. To keep thing cleaner, so you'll probably get a similar structure:

```
/your/path/to/grapesjs.min.js
/your/path/to/grapesjs-plugin.js
```

**Important:** The order that you load files matters. GrapesJS has to be loaded before the plugin. This sets up the `grapejs` global variable.

So, in your `grapesjs-plugin.js` file:

```js
export default grapesjs.plugins.add('my-plugin-name', (editor, options) => {
  /*
  * Here you should rely on GrapesJS APIs, so check 'API Reference' for more info
  * For example, you could do something like this to add some new command:
  *
  * editor.Commands.add(...);
  */
})
```

The name `my-plugin-name` is an ID of your plugin and you'll use it to tell your editor to grab it.

Here is a complete generic example:

```html
<script src="http://code.jquery.com/jquery-2.2.0.min.js"></script>
<link rel="stylesheet" href="path/to/grapes.min.css">
<script src="path/to/grapes.min.js"></script>
<script src="path/to/grapesjs-plugin.js"></script>

<div id="gjs"></div>

<script type="text/javascript">
  var editor = grapesjs.init({
      container : '#gjs',
      plugins: ['my-plugin-name']
  });
</script>
```
-->

## Plugins with options

It's also possible to pass custom parameters to plugins in to make them more flexible.

<!--
```js
  var editor = grapesjs.init({
      container : '#gjs',
      plugins: ['my-plugin-name'],
      pluginsOpts: {
        'my-plugin-name': {
          customField: 'customValue'
        }
      }
  });
```

Inside you plugin you'll get those options via `options` argument

```js
export default grapesjs.plugins.add('my-plugin-name', (editor, options) => {
  console.log(options);
  //{ customField: 'customValue' }
})
```

This also works with plugins that aren't named.

-->

```js
const myPluginWithOptions = (editor, options) => {
  console.log(options);
  // { customField: 'customValue' }
};

const editor = grapesjs.init({
  container: '#gjs',
  plugins: [myPluginWithOptions],
  pluginsOpts: {
    [myPluginWithOptions]: {
      customField: 'customValue',
    },
  },
});
```

<!--
## Named Plugins vs Non-Named Plugins

When you use a named plugin, then that name must be unique across all other plugins.

```js
grapesjs.plugins.add('my-plugin-name', fn);
```

In this example, the plugin name is `my-plugin-name` and can't be used by other plugins. To avoid namespace restrictions use basic plugins that are purely functional.

-->

## Usage with TS

If you're using TypeScript, for a better type safety, we recommend using the `usePlugin` helper.

```ts
import grapesjs, { usePlugin } from 'grapesjs';
import type { Plugin } from 'grapesjs';

interface MyPluginOptions {
  opt1: string;
  opt2?: number;
}

const myPlugin: Plugin<MyPluginOptions> = (editor, options) => {
  // ...
};

grapesjs.init({
  // ...
  plugins: [
    // no need for `pluginsOpts`
    usePlugin(myPlugin, { opt1: 'A', opt2: 1 }),
  ],
});
```

## Boilerplate

For fast plugin development, we highly recommend using [grapesjs-cli](https://github.com/GrapesJS/cli) which helps to avoid the hassle of setting up all the dependencies and configurations for development and building (no need to touch Webpack or Babel configurations). For more information check the repository.

---
title: Style Manager
---

# Style Manager

<p align="center"><img :src="$withBase('/style-manager.jpg')" alt="GrapesJS - Style Manager"/></p>

The Style Manager module is responsible to show and update style properties relative to your [Components]. In this guide, you will see how to setup and take full advantage of the built-in Style Manager UI in GrapesJS.
The default UI is a lightweight component with built-in properties, but as you'll see next in this guide, it's easy to extend with your own elements or even create the Style Manager UI from scratch by using the [Style Manager API].

::: warning
To get a better understanding of the content in this guide, we recommend reading [Components] first
:::
::: warning
This guide is referring to GrapesJS v0.18.1 or higher
:::

::: tip
Looking for a UI that is easy and ready to customize? [Checkout the Grapes Studio SDK!](https://app.grapesjs.com/docs-sdk/configuration/components/overview?utm_source=grapesjs-docs&utm_medium=tip) 
:::

[[toc]]

## Configuration

To change the default configurations you have to pass the `styleManager` property with the main configuration object.

```js
const editor = grapesjs.init({
  ...
  styleManager: {
    sectors: [...],
    ...
  }
});
```

Check the full list of available options here: [Style Manager Config](https://github.com/GrapesJS/grapesjs/blob/master/src/style_manager/config/config.ts)

## Initialization

The Style Manager module is organized in sectors where each sector contains a list of properties to display. The default Style Manager configuration contains already a list of default common property styles and you can use them by simply skipping the `styleManagerConfig.sectors` option.

```js
grapesjs.init({
  ...
  styleManager: {
    // With no defined sectors, the default list will be loaded
    // sectors: [...],
    ...
  },
});
```

::: danger
It makes sense to show the Style Manager UI only when you have at least one component selected, so by default the Style Manager is hidden if there are no selected components.
:::

### Sector definitions

Each sector is identified by its `name` and a list of `properties` to display. You can also specify the `id` in order to access the sector via API (if not indicated it will be generated from the `name`) and the default `open` state.

```js
grapesjs.init({
  // ...
  styleManager: {
    sectors: [
      {
        name: 'First sector',
        properties: [],
        // id: 'first-sector', // Id generated from the name
        // open: true, // The sector is open by default
      },
      {
        open: false, // render it closed by default
        name: 'Second sector',
        properties: [],
      },
    ],
  },
});
```

### Property definitions

Once you have defined your sector you can start adding property definitions inside `properties`. Each property has a common set of options (`label`, `default`, etc.) and others specific by their `type`.

Let's see below a simple definition for controlling the padding style.

```js
sectors: [
  {
    name: 'First sector',
    properties: [
      {
        // Default options
        // id: 'padding', // The id of the property, if missing, will be the same as `property` value
        type: 'number',
        label: 'Padding', // Label for the property
        property: 'padding', // CSS property to change
        default: '0', // Default value to display
        // Additonal `number` options
        units: ['px', '%'], // Units (available only for the 'number' type)
        min: 0, // Min value (available only for the 'number' type)
      },
    ],
  },
];
```

This will render the number input UI and will change the `padding` CSS property on the selected component.

The flexibility of the definition allows you to create easily different UI inputs for any possible CSS property. You're free to decide what will be the best UI for your users. If you take for example a numeric property like `font-size`, you can follow its CSS specification and define it as a `number`

```js
{
  type: 'number',
  label: 'Font size',
  property: 'font-size',
  units: ['px', '%', 'em', 'rem', 'vh', 'vw'],
  min: 0,
}
```

or you can decide to show it as a `select` and make available only a defined set of values (eg. based on your Design System tokens).

```js
{
  type: 'select',
  label: 'Font size',
  property: 'font-size',
  default: '1rem',
  options: [
    { id: '0.7rem', label: 'small' },
    { id: '1rem', label: 'medium' },
    { id: '1.2rem', label: 'large' },
  ]
}
```

Each type defines the specific UI view and a model on how to handle inputs and updates.
Let's see below the list of all available default types with their relative UI and models.

#### Default types

::: tip
Each **Model** describes more in detail available props and their usage.
:::

- `base` - The base type, renders as a simple text input field. **Model**: [Property](/api/property.html)

  <img :src="$withBase('/sm-base-type.jpg')"/>

  ```js
  // Example
  {
    // type: 'base', // Omitting the type in definition will fallback to the 'base'
    property: 'some-css-property',
    label: 'Base type',
    default: 'Default value',
  },
  ```

- `color` - Same props as `base` but the UI is a color picker. **Model**: [Property](/api/property.html)

  <img :src="$withBase('/sm-type-color.jpg')"/>

- `number` - Number input field for numeric values. **Model**: [PropertyNumber](/api/property_number.html)

  <img :src="$withBase('/sm-type-number.jpg')"/>

  ```js
  // Example
  {
    type: 'number',
    property: 'width',
    label: 'Number type',
    default: '0%',
    // Additional props
    units: ['px', '%'],
    min: 0,
    max: 100,
  },
  ```

- `slider` - Same props as `number` but the UI is a slider. **Model**: [PropertyNumber](/api/property_number.html)

  <img :src="$withBase('/sm-type-slider.jpg')"/>

- `select` - Select input with options. **Model**: [PropertySelect](/api/property_select.html)

  <img :src="$withBase('/sm-type-select.jpg')"/>

  ```js
  // Example
  {
    type: 'select',
    property: 'display',
    label: 'Select type',
    default: 'block',
    // Additional props
    options: [
      {id: 'block', label: 'Block'},
      {id: 'inline', label: 'Inline'},
      {id: 'none', label: 'None'},
    ]
  },
  ```

- `radio` - Same props as `select` but the UI is a radio button. **Model**: [PropertySelect](/api/property_select.html)

  <img :src="$withBase('/sm-type-radio.jpg')"/>

- `composite` - This type is great for CSS shorthand properties, where the final value is a composition of multiple sub properties. **Model**: [PropertyComposite](/api/property_composite.html)

  <img :src="$withBase('/sm-type-composite.jpg')"/>

  ```js
  // Example
  {
    type: 'composite',
    property: 'margin',
    label: 'Composite type',
    // Additional props
    properties: [
      { type: 'number', units: ['px'], default: '0', property: 'margin-top' },
      { type: 'number', units: ['px'], default: '0', property: 'margin-right' },
      { type: 'number', units: ['px'], default: '0', property: 'margin-bottom' },
      { type: 'number', units: ['px'], default: '0', property: 'margin-left' },
    ]
  },
  ```

- `stack` - This type is great for CSS multiple properties like `text-shadow`, `box-shadow`, `transform`, etc. **Model**: [PropertyStack](/api/property_stack.html)

  <img :src="$withBase('/sm-type-stack.jpg')"/>

  ```js
  // Example
  {
    type: 'stack',
    property: 'text-shadow',
    label: 'Stack type',
    // Additional props
    properties: [
      { type: 'number', units: ['px'], default: '0', property: 'x' },
      { type: 'number', units: ['px'], default: '0', property: 'y' },
      { type: 'number', units: ['px'], default: '0', property: 'blur' },
      { type: 'color', default: 'black', property: 'color' },
    ]
  },
  ```

#### Built-in properties

In order to speed up the Style Manager configuration, GrapesJS is shipped with a set of already defined common CSS properties which you can reuse and extend.

```js
sectors: [
  {
    name: 'First sector',
    properties: [
      // Pass the built-in CSS property as a string
      'width',
      'min-width',
      // Extend the built-in property with your props
      {
        extend: 'max-width',
        units: ['px', '%'],
      },
      // If the property doesn't exist it will be converted to a base type
      'unknown-property', // -> { type: 'base', property: 'unknown-property' }
    ],
  },
];
```

::: tip
You can check if the property is available by running

```js
editor.StyleManager.getBuiltIn('property-name');
```

or get the list of all available properties with

```js
editor.StyleManager.getBuiltInAll();
```

:::

## I18n

If you're planning to have a multi-language editor you can easily connect sector and property labels to the [I18n] module via their IDs.

```js
grapesjs.init({
  styleManager: {
    sectors: [
      {
        id: 'first-sector-id',
        // You can leave the name as a fallback in case the i18n definition is missing
        name: 'First sector',
        properties: [
          'width',
          {
            id: 'display-prop-id', // By default the id is the same as its property name
            label: 'Display',
            type: 'select',
            property: 'display',
            default: 'block',
            options: [
              { id: 'block', label: 'Block' },
              { id: 'inline', label: 'Inline' },
              { id: 'none', label: 'None' },
            ],
          },
        ],
      },
      // ...
    ],
  },
  i18n: {
    // Use `messagesAdd` in order to extend the default set
    messagesAdd: {
      en: {
        styleManager: {
          sectors: {
            'first-sector-id': 'First sector EN',
          },
          properties: {
            width: 'Width EN',
            'display-prop-id': 'Display EN',
          },
          options: {
            'display-prop-id': {
              block: 'Block EN',
              inline: 'Inline EN',
              none: 'None EN',
            },
          },
        },
      },
    },
  },
});
```

## Component constraints

When you define custom components you can also indicate, via `stylable` and `unstylable` props, which CSS properties should be available for styling. In that case, the Style Manager will only show the available properties. If the sector doesn't contain any available property, it won't be shown.

```js
const customComponents = (editor) => {
  // Component A
  editor.Components.addType('cmp-a', {
    model: {
      defaults: {
        // When this component is selected, the Style Manager will show only the following properties
        stylable: ['width', 'height'],
      },
    },
  });
  // Component B
  editor.Components.addType('cmp-b', {
    model: {
      defaults: {
        // When this component is selected, the Style Manager will hide the following properties
        unstylable: ['color'],
      },
    },
  });
};

grapesjs.init({
  // ...
  plugins: [customComponents],
  components: [
    { type: 'cmp-a', components: 'Component A' },
    { type: 'cmp-b', components: 'Component B' },
  ],
  styleManager: {
    sectors: [
      {
        name: 'First sector',
        properties: ['width', 'min-width', 'height', 'min-height'],
      },
      {
        name: 'Second sector',
        properties: ['color', 'font-size'],
      },
    ],
  },
});
```

## Programmatic usage

For a more advanced usage you can rely on the [Style Manager API] to perform different kind of actions related to the module.

- Managing sectors/properties post-initialization.

  ```js
  // Get the module from the editor instance
  const sm = editor.StyleManager;

  // Add new sector
  const newSector = sm.addSector('sector-id', {
    name: 'New sector',
    open: true,
    properties: ['width'],
  });

  // Add new property to the sector
  sm.addProperty('sector-id', {
    type: 'number',
    property: 'min-width',
  });

  // Remove sector
  sm.removeSector('sector-id');
  ```

- Managing selected targets.

  ```js
  // Select the first button in the current page
  const wrapperCmp = editor.Pages.getSelected().getMainComponent();
  const btnCmp = wrapperCmp.find('button')[0];
  btnCmp && sm.select(btnCmp);

  // Set as a target the CSS selector (the relative CSSRule will be created if doesn't exist yet)
  sm.select('.btn > span');
  // Get the last selected target
  const lastTarget = sm.getLastSelected();
  lastTarget?.toCSS && console.log(lastTarget.toCSS());

  // Update selected targets with a custom style
  sm.addStyleTargets({ color: 'red' });
  ```

- Adding/extending built-in property definitions.

  ```js
  const myPlugin = (editor) => {
    editor.StyleManager.addBuiltIn('new-prop', {
      type: 'number',
      label: 'New prop',
    })
  };

  grapesjs.init({
    // ...
    plugins: [myPlugin],
    styleManager: {
      sectors: [
        {
          name: 'My sector',
          properties: [ 'new-prop', ... ],
        },
      ],
    },
  })
  ```

- [Adding new types](#adding-new-types).

## Customization

The default types should cover most of the common styling properties but in case you need a more advanced control over styling you can define your own types or even create a completely custom Style Manager UI from scratch.

### Adding new types

In order to add a new type you have to use the `styleManager.addType` API call and indicate all the necessary methods to make it work properly with the editor. Here an example of implementation by using the native `range` input.

<demo-viewer value="y1mxv6p5" height="500" darkcode/>

<!-- ```js
const customType = (editor) => {
  editor.StyleManager.addType('my-custom-prop', {
    // Create UI
    create({ props, change }) {
      const el = document.createElement('div');
      el.innerHTML = `<input type="range" class="my-input" min="${props.min}" max="${props.max}"/>`;
      const inputEl = el.querySelector('.my-input');
      inputEl.addEventListener('change', event => change({ event })); // `change` will trigger the emit
      inputEl.addEventListener('input', event => change({ event, partial: true }));
      return el;
    },
    // Propagate UI changes up to the targets
    emit({ props, updateStyle }, { event, partial }) {
      const { value } = event.target;
      updateStyle(`${value}px`, { partial });
    },
    // Update UI (eg. when the target is changed)
    update({ value, el }) {
      el.querySelector('.my-input').value = parseInt(value, 10);
    },
    // Clean the memory from side effects if necessary (eg. global event listeners, etc.)
    destroy() {
    },
  });
};

grapesjs.init({
  // ...
  plugins: [customType],
  styleManager: {
    sectors: [
      {
        name: 'My sector',
        properties: [
          {
            type: 'my-custom-prop',
            property: 'font-size',
            default: '15',
            min: 10,
            max: 70,
          },
        ],
      },
    ],
  },
})
``` -->

### Custom Style Manager

In case you need a completely custom Style Manager UI (eg. you have to use your UI components), you can create it from scratch by relying on the same sector/propriety architecture or even interfacing directly with selected targets.

All you have to do is to indicate the editor your intent to use a custom UI and then subscribe to the `style:custom` event that will let you know when is the right moment to create/update your UI.

```js
const editor = grapesjs.init({
  // ...
  styleManager: {
    custom: true,
    // ...
  },
});

editor.on('style:custom', (props) => {
  // props.container (HTMLElement)
  //    The default element where you can append your
  //    custom UI in order to render it in the default position.
  // Here you would put the logic to render/update your UI by relying on Style Manager API
});
```

Here an example below of a custom Style Manager UI rendered by using Vuetify (Vue Material components) and relying on the default sector/propriety state architecture via API.

<demo-viewer value="46kf7brn" height="500" darkcode/>

From the example above you can notice how we subscribe to `style:custom` and update `this.sectors = sm.getSectors({ visible: true });` on each trigger, and this is enough for the framework to update the rest of the template automatically.

In case you need to get/update the selected style targets directly, you can also rely on these APIs.

```js
// Get the module from the editor instance
const sm = editor.StyleManager;

// Select the first button in the current page
const wrapperCmp = editor.Pages.getSelected().getMainComponent();
const btnCmp = wrapperCmp.find('button')[0];
btnCmp && sm.select(btnCmp);

// You can also select CSS query as a target
sm.select('.btn > span');

// Once the target is selected, you can check its current style object
console.log(sm.getSelected()?.getStyle());

// and update all selected target styles when necessary
sm.addStyleTargets({ color: 'red' });
```

<!--
<style>
  .style-manager {
    font-size: 12px;
  }
  .sm-input-color {
    width: 16px !important;
    height: 15px !important;
    opacity: 0 !important;
  }
  .sm-type-cmp {
    background-color: rgba(255,255,255,.05);
    border: 1px solid rgba(255,255,255,.25);
    border-radius: 3px;
    position: relative;
    min-height: 45px;
  }
  .sm-add-layer {
    position: absolute !important;
    top: -20px;
    right: 12px;
  }
  .sm-layer + .sm-layer {
    border-top: 1px solid rgba(255,255,255,.25);
  }
  .sm-layer-prv,
  .sm-btn-prv {
    width: 16px;
    height: 16px;
    border-radius: 3px;
    display: inline-block;
  }

  .sm-layer-prv--text-shadow::after {
    color: #000;
    content: "T";
    font-weight: 900;
    font-size: 10px;
    text-align: center;
    display: block;
  }

  .gjs-pn-views-container, .gjs-pn-views {
    width: 280px;
  }
  .gjs-pn-options {
    right: 280px;
  }
  .gjs-cv-canvas {
    width: calc(100% - 280px);
  }

  /* Vuetify overrides */
  .v-application {
    background: transparent !important;
  }
  .v-application--wrap {
    min-height: auto;
  }
  .v-input__slot {
    font-size: 12px;
    min-height: 10px !important;
  }
  .v-select__selections {
    flex-wrap: nowrap;
  }
  .v-text-field .v-input__slot {
    padding: 0 10px !important;
  }
  .v-input--selection-controls {
    margin-top: 0;
  }
  .v-text-field__details, .v-messages {
    display: none;
  }
</style>
<div>
  <div class="style-manager">
      <v-app>
        <v-main>
        <v-expansion-panels  accordion multiple>
          <v-expansion-panel v-for="sector in sectors" :key="sector.getId()">
            <v-expansion-panel-header>
              {{ sector.getName() }}
            </v-expansion-panel-header>
            <v-expansion-panel-content>
              <v-row>
              <property-field v-for="prop in sector.getProperties()" :key="prop.getId() + prop.canClear()" :prop="prop"/>
              </v-row>
            </v-expansion-panel-content>
          </v-expansion-panel>
        </v-expansion-panels>
        </v-main>
      </v-app>
  </div>

  <div id="property-field" style="display: none;">
    <v-col :class="['py-0 px-1 mb-1', prop.isFull() && 'mb-3']" :cols="prop.get('full') ? '12' : '6'">
        <v-row :class="labelCls">
          <v-col cols="auto pr-0">{{ prop.getLabel() }}</v-col>
          <v-col cols="auto" v-if="prop.canClear()">
            <v-icon @click="prop.clear()" color="indigo accent-1" small>mdi-close</v-icon>
          </v-col>
        </v-row>
        <div v-if="propType === 'number'">
          <v-text-field :placeholder="defValue" :value="inputValue" @change="handleChange" outlined dense/>
        </div>
        <div v-else-if="propType === 'radio'">
          <v-radio-group :value="prop.getValue()" @change="handleChange" row dense>
            <v-radio v-for="opt in prop.getOptions()" :key="prop.getOptionId(opt)" :label="prop.getOptionLabel(opt)" :value="prop.getOptionId(opt)"/>
          </v-radio-group>
        </div>
        <div v-else-if="propType === 'select'">
          <v-select :items="toOptions" :value="prop.getValue()" @change="handleChange" outlined dense/>
        </div>
        <div v-else-if="propType === 'color'">
          <v-text-field :placeholder="defValue" :value="inputValue" @change="handleChange" outlined dense>
            <template v-slot:append>
              <div :style="{ backgroundColor: prop.hasValue() ? prop.getValue() : defValue }">
                <input class="sm-input-color"
                  type="color"
                  :value="prop.hasValue() ? prop.getValue() : defValue"
                  @change="(ev) => handleChange(ev.target.value)"
                  @input="(ev) => handleInput(ev.target.value)"
                />
              </div>
          </template>
          </v-text-field>
        </div>
        <div v-else-if="propType === 'slider'">
          <v-slider
            track-color="white"
            :value="prop.getValue()"
            :min="prop.getMin()"
            :max="prop.getMax()"
            :step="prop.getStep()"
            @change="handleChange"
            @input="(value) => { console.log('trigger input', value) }"
            @start="(value) => { console.log('trigger start', value) }"
          />
        </div>
        <div v-else-if="propType === 'file'">
          <v-btn @click="openAssets(prop)" block>
            <v-row>
              <v-col v-if="prop.getValue() && prop.getValue() !== defValue" cols="auto">
                <div class="sm-btn-prv" :style="{ backgroundImage: `url(${prop.getValue()})` }"></div>
              </v-col>
              <v-col>Select image</v-col>
            </v-row>
          </v-btn>
        </div>
        <div v-else-if="propType === 'composite'">
          <v-row no-gutters class="sm-type-cmp pa-2">
            <property-field v-for="p in prop.getProperties()" :key="p.getId() + p.canClear()" :prop="p"/>
          </v-row>
        </div>
        <div v-else-if="propType === 'stack'">
          <div class="sm-type-cmp pa-3">
            <v-icon @click="prop.addLayer({}, { at: 0 })" class="sm-add-layer" small>mdi-plus</v-icon>
            <v-row class="sm-layer" v-for="layer in prop.getLayers()" :key="layer.getId()">
              <v-col>
                <v-row>
                  <v-col cols="auto" class="pr-1">
                    <v-icon @click="layer.move(layer.getIndex() - 1)" small>mdi-arrow-up</v-icon>
                  </v-col>
                  <v-col cols="auto" class="pl-1">
                    <v-icon @click="layer.move(layer.getIndex() + 1)" small>mdi-arrow-down</v-icon>
                  </v-col>
                  <v-col @click="layer.select()">{{ layer.getLabel() }}</v-col>
                  <v-col cols="auto">
                      <div :class="['sm-layer-prv white', `sm-layer-prv--${propName}`]" :style="layer.getStylePreview({ number: { min: -3, max: 3 } })"></div>
                  </v-col>
                  <v-col cols="auto">
                    <v-icon @click="layer.remove()" small>mdi-close</v-icon>
                  </v-col>
                </v-row>
                <v-row v-if="layer.isSelected()" no-gutters class="sm-type-cmp pa-2 mt-3">
                  <property-field v-for="p in prop.getProperties()" :key="p.getId()" :prop="p"/>
                </v-row>
              </v-col>
            </v-row>
          </div>
        </div>
        <div v-else>
          <v-text-field :placeholder="defValue" :value="inputValue" @change="handleChange" outlined dense/>
        </div>
    </v-col>
  </div>
</div>
<script>

  const sm = editor.StyleManager;
  const Observer = (new Vue()).$data.__ob__.constructor; // obj.__ob__ = new Observer({});

  Vue.mixin({
    data() {
      return { editor };
    }
  });

  Vue.component('property-field', {
    props: { prop: Object },
    template: '#property-field',
    computed: {
      labelCls() {
        const { prop } = this;
        const parent = prop.getParent();
        const hasParentValue = prop.hasValueParent() && (parent ? parent.isDetached() : true);
        return ['flex-nowrap', prop.canClear() && 'indigo--text text--accent-1', hasParentValue && 'orange--text'];
      },
      inputValue() {
        return this.prop.hasValue() ? this.prop.getValue() : '';
      },
      propName() {
        return this.prop.getName();
      },
      propType() {
        return this.prop.getType();
      },
      defValue() {
        return this.prop.getDefaultValue();
      },
      toOptions() {
        const { prop } = this;
        return prop.getOptions().map(o => ({ value: prop.getOptionId(o), text: prop.getOptionLabel(o) }))
      },
    },
    methods: {
      handleChange(value) {
        this.prop.upValue(value);
      },
      handleInput(value) {
        this.prop.upValue(value, { partial: true });
      },
      openAssets(prop) {
        const { Assets } = this.editor;
        Assets.open({
          select: (asset, complete) => {
            prop.upValue(asset.getSrc(), { partial: !complete });
            complete && Assets.close();
          },
          types: ['image'],
          accept: 'image/*',
        })
      }
    }
  })

  const app = new Vue({
    vuetify: new Vuetify({
      theme: { dark: true },
    }),
    el: '.style-manager',
    data: { sectors: [] },
    mounted() {
      editor.on('style:custom', this.handleCustom);
    },
    destroyed() {
      editor.off('style:custom', this.handleCustom);
    },
    methods: {
      handleCustom(props) {
        if (props.container && !props.container.contains(this.$el)) {
          props.container.appendChild(this.$el);
        }
        this.sectors = sm.getSectors({ visible: true });
      },
    }
  });
</script>
-->

## Events

For a complete list of available events, you can check it [here](/api/style_manager.html#available-events).

[Components]: Components.html
[I18n]: I18n.html
[Style Manager API]: /api/style_manager.html