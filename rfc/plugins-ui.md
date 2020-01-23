
# Plugins UI Design Doc
| | |
|-|-|
| **Status** | **Proposed**, Accepted, Implemented, Obsolete |
| **RFC  Num** | [#74](https://github.com/spinnaker/community/pull/74) |
| **Author(s)** | Brandon Powell (`@bpowell`), Cameron Motevasselani (`@link108`), Clay Mccoy (`@claymccoy`), Chris Thielen (`@christopherthielen`) |
| **SIG / WG** | Platform SIG |
| **Obsoletes** | [Google Doc](https://docs.google.com/document/d/16WmRSziTJsSBZ1kuKUfVleLAYMIxmfvz/edit) |

## Motivation
Spinnaker is used by many companies, each having wildly different needs.  Many enterprise companies have heavily customized Spinnaker improve the experience based on their needs.  Some examples of extensions seen in the real world are: 

1. Custom stages
2. Modifying header and footer
3. Custom search filters
4. Custom routes
5. Custom details panels
6. Adding labels or other information to sections in the infrastructure tab
 
However, the process of customization necessitates a custom build of the spinnaker services and UI.  Adding a build step greatly increases the operational costs and complexity of running spinnaker.  Smaller companies with fewer resources likely lack the engineering and ops bandwidth to do this.

This proposal explores a world where the the UI (Deck) is extensible _without a build step_.  
  
This design doc references the [Plugins RFC](./plugins.md).

This doc focuses on:
* [Loading Plugin code into Deck](#Loading-Plugin-code-into-Deck)
* [Extension points](#Extension-Points)
* [Plugin development](#Plugin-development)
* [Bootstrapping Deck](#bootstrapping-deck)

## Loading Plugin code into Deck

### Gathering plugin metadata
In order for Deck to know which plugins it can load, it must have access to some configuration information, including the name of the plugins that are enabled and where to download the plugin resources (JavaScript bundles, CSS, images, etc).
This Plugin Configuration will be fetched from Front50 via Gate.
For local development, plugin configuration will be loaded from `settings.js` and merged with the configuration from Front50.

Please see the current [Plugins RFC](./plugins.md) for details around Front50 as the source of truth for plugin metadata.
Until the Front50 work is completed, `settings.js` will be used.

### Loading plugin resources
The JavaScript Plugin resources will be fetched and loaded by Deck using native Javascript `import`.
Plugin resources must be served from a server that is configured to be [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) aware.

Some potential servers may be:
- An artifact store such as Artifactory
- Deck's standard Docker image.  Plugin resources would have to be located on-instance [1]
- A proxy service that serves Deck resources as well as resources from a storage service such as S3 (this is the case at Netflix)
- Front50 itself, via Gate.  This seems like a promising option.

[1] Halyard could configure an init container which would download plugin resources into the container.  The resources would be served by Apache in the same way that the standard resources are served.  Note: if such an init container were created, it could potentially serve the same purpose to front-load plugin assets for backend services.

## Extension Points

Deck should provide a set of well defined extension points.
A number of extension points exist today.
Examples include:

- Stage Registry
- Cloud provider Registry
- Data Source Registry
- Application tabs/routes
- Component Overrides

Today, these extension points are imperative, e.g., `Registry.pipeline.registerStage(stage)`.
Deck will define a Plugin interface, `IDeckPlugin`, which will have declarative fields for each extension point.
Some extension points (such as application tabs) may not support a purely declarative model.
For such extension points, we may define initialization callback signatures that allow the plugin to initialize those Extensions.

## Plugin Development

Plugin developers should have a development experience that closely resembles development of Deck itself.
Spinnaker should provide a plugin quick-start, such as a github repository that developers can clone.
This quick start should provide:

- A standard build system (Typescript, Rollup, etc)
- Templates for creating specific extension points, i.e., an example stage
- A kork-like `@spinnaker/plugins` package (name TBD) which includes:
  - The interfaces that the plugin should conform to
  - A curated set of transitive dependencies (3rd party libraries) shared with core Deck
  - A set of reusable UI components (that spinnaker itself is built upon)
  - A set of linter rules which match the code style in core Deck.

The build system will build a plugin as one or more ES6 module(s).
Deck will import these ES6 modules using native browser `import()`.
The plugin should provide an entry point for these module(s) at `/index.js`.
This file should export the plugin object using a named export of `plugin`.
The plugin object must conform to the plugin interface specified by `@spinnaker/plugins`.
This interface will have properties for each extension point which the plugin can populate.  
For example:

```js
import { fancyStage } from './fancyStage';
export const plugin = {
  stages: [fancyStage],
}
```

The build system should ideally support code splitting to enable lazy loading of portions of plugin code.
The build system should not include the transitive dependencies that are already loaded by core Deck.
Instead, it should share those curated dependencies with the plugin at runtime from the existing code loaded into core deck.

### Imperative Initialization
To make front end plugins more similar to way in which back end plugins work, there needs to be way for plugin creators to be able to do things that there are currently no extensions for. This will allow plugin developers to add extra functionality that is currently not a well defined extension point. Over time as more plugins are created, extension points will be added. To do this the `plugin` that each plugin has to export can have a method attached to it called `undefinedExtensions` that will be executed by Deck for any code that is defined in there to run.

```
import { fancyStage } from './fancyStage';
import { myUndefinedExtension } from './myUndefinedExtension';
export const plugin = {
  stages: [fancyStage],
  myUndefinedExtension: myUndefinedExtension,
}
```

Then were we call to register the stages, the `undefinedExtensions` method will be called if it is defined:
```
plugin.undefinedExtensions && plugin.undefinedExtensions();
```

Doing this solves the chicken and egg problem that we currently have. Only Stages are exposed for plugin developers. If a plugin developer wanted to do anything else as a plugin in Deck, there is no way to do it.

## Bootstrapping Deck

Plugins will be able to use shared library code which is provided by Deck itself.
These shared libraries should be loaded before the plugins themselves.
Because of this, plugins should be loaded after Deck has exposed the shared libraries.
The order of operations likely should be:

- Load Deck assets
- Deck Bootstrap
  - Load plugin manifest
  - Load each plugin's entrypoint as a module
  - Register extensions from the plugin(s)
  - Start Deck (initialize the router)

During Deck's bootstrap it should inspect the `plugin` object of each plugin module.
It should iterate each extension found and register it with Deck.
For example, Deck will inspect the plugin object and look for the `stages` property and register each as a Deck stage.

```js
import(pluginUrl).then(pluginModule => {
  const { plugin } = pluginModule;
  plugin.stages?.forEach(stage => Registry.pipeline.registerStage(stage);
}).catch(loadError => {
  // Error handling behavior TBD
});
```

## Known Unknowns
* Where are resources downloaded from? How do we ensure end users can download plugin resources?
* Failure handling.  
  * What should deck do if it fails to reach Front50 or to load plugins?
    * Spinnaker alert/banner at top of page should inform the user that plugins were not able to be loaded
      * Only show this warning if plugins are enabled
      * Give guidance on how to resolve error or look up more info
  * What should deck do if it fails to download a plugin resource? 
* Plugin manifests will contain locations where assets are, but another place may serve them (eg secure artifact store), can halyard help with this?

## Alternatives to Loading

### Build step
We may be able to modify Webpack to download plugins and compile Deck at deploy time. This eliminates the need for users to reach out to an artifact store and leads to a smoother UX. It would increase deploy times by a lot though, so that is something to consider. Going this path would also mean that Deck would have to be redeployed any time a new plugin has been added.

### RequireJS
RequireJS could be used to load plugins. When doing initial prototyping with RequireJS, it was complaining at compile time about not being able to find plugins because they are supposed to be added at runtime, not compile time. 
