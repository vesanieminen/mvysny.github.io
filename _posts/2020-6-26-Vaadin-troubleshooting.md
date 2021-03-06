---
layout: post
title: Vaadin 14 - Troubleshooting
---

Sometimes Vaadin 14 fails mysteriously, which is very unfortunate: it now uses
LOTS of JavaScript code, frameworks and tools. Those things are unfamiliar to server-side
guys, and it's really hard to tell what to do, should one of JavaScript toolchain fail.

This article remedies that, by listing a bunch of troubleshooting ideas and FAQs to
follow and check out. We expect a basic knowledge of Vaadin toolchain here, please
read the [Vaadin: the missing guide](../Vaadin-the-missing-guide/) article
and the [Vaadin Plugin under the hood](../Vaadin-Plugin-under-the-hood/) article first.

## What to verify server-side

Sometimes Vaadin will fail to update `package.yaml` and `package-lock.yaml` with
new versions of npm modules (especially after a Vaadin version update),
or sometimes the `node_modules` becomes corrupted, or `package.yaml` will list
multiple versions of the same thing.

The best thing to do first is to perform the "Vaadin Dance" (akin to [Eclipse Dance](http://underlap.blogspot.com/2011/10/eclipse-dance-steps.html)):
delete all npm-related packages, npm config files and webpack config files and force
Vaadin to start from the clean state. You can achieve this by deleting the following files:

* `node_modules`
* `package.json`
* `package-lock.json` (if using npm)
* `webpack.generated.js`
* `pnpm-lock.yaml` (if using pnpm)
* `pnpmfile.js` (if using pnpm)

(With [Vaadin Gradle plugin](https://github.com/vaadin/vaadin-gradle-plugin) you can simply run the `vaadinClean` task to do this).

Then run the `prepare-frontend` task, to re-create those files. Note that the `node_modules`
is populated later: either by Vaadin Servlet when running in development mode, or
by `build-frontend` when building for production.

### Gradle: use matching version of Vaadin and the Plugin

Use Gradle Plugin 0.6.0 for Vaadin 14.1 and lower; use Gradle Plugin 0.7.0 for Vaadin 14.2 and newer.
Gradle Plugin 0.6.0 doesn't work correctly with Vaadin 14.2.

### Use newest Vaadin

Use the newest Vaadin 14.x - there are also fixes with respect to the JavaScript
toolchain; for example it could be that at some point Vaadin will be able to detect
all breaks and issues and the Vaadin Dance will be unnecessary.

### Production issues: make sure the necessary files are present

Make sure that all of the Vaadin files are present in the WAR/EAR/JAR archive, when
built for production mode via the `build-frontend` Plugin task:

* Most importantly, the `flow-build-info.json` file, located on classpath in the
  `META-INF/VAADIN/config/` package.
  * In a WAR archive it's located in the
  `WEB-INF/classes/META-INF/VAADIN/config/` folder; in a SpringBoot JAR it's located
  in `BOOT-INF/classes/META-INF/VAADIN/config/` folder.
  * Make sure that it sets the following properties as follows:
  `"compatibilityMode": false, "productionMode": true, "enableDevServer": false`
  * See [Vaadin: the missing guide](../Vaadin-the-missing-guide/) for an example of
  how the file should look like.
* The `stats.json` file, located on classpath right next to the `flow-build-info.json`,
  in the same package.
* The `META-INF/VAADIN/build/` folder is populated as follows:

```
├── vaadin-2-18d67c4ccff7e93b081a.cache.js
├── vaadin-2-18d67c4ccff7e93b081a.cache.js.gz
├── vaadin-3-b0147df339bf18eb7618.cache.js
├── vaadin-3-b0147df339bf18eb7618.cache.js.gz
├── vaadin-4-ee1d2e45569f7eca4292.cache.js
├── vaadin-4-ee1d2e45569f7eca4292.cache.js.gz
├── vaadin-5-5e9292474e82143d0a27.cache.js
├── vaadin-5-5e9292474e82143d0a27.cache.js.gz
├── vaadin-bundle-19a00eae62ad7cddd291.cache.js
├── vaadin-bundle-19a00eae62ad7cddd291.cache.js.gz
├── vaadin-bundle.es5-b1c1a3cc054c62ad7949.cache.js
├── vaadin-bundle.es5-b1c1a3cc054c62ad7949.cache.js.gz
└── webcomponentsjs
    ├── bundles
    │   ├── webcomponents-ce.js
    │   ├── webcomponents-ce.js.map
    │   ├── webcomponents-sd-ce.js
    │   ├── webcomponents-sd-ce.js.map
    │   ├── webcomponents-sd-ce-pf.js
    │   ├── webcomponents-sd-ce-pf.js.map
    │   ├── webcomponents-sd.js
    │   └── webcomponents-sd.js.map
    ├── custom-elements-es5-adapter.js
    ├── LICENSE.md
    ├── package.json
    ├── README.md
    ├── src
    │   └── entrypoints
    │       ├── custom-elements-es5-adapter-index.js
    │       ├── webcomponents-bundle-index.js
    │       ├── webcomponents-ce-index.js
    │       ├── webcomponents-sd-ce-index.js
    │       ├── webcomponents-sd-ce-pf-index.js
    │       └── webcomponents-sd-index.js
    ├── webcomponents-bundle.js
    ├── webcomponents-bundle.js.map
    └── webcomponents-loader.js
```

Important: in the development mode, only the `flow-build-info.json` needs to
be present on the classpath. Vaadin Servlet will internally start a child `webpack`
process and will transfer all other files internally from `webpack`.

## What to verify in your browser

You should at some point learn how to use your browser's dev tools:

* [Firefox Dev Tools Guide](https://developer.mozilla.org/en-US/docs/Tools)
* [Chrome Dev Tools Guide](https://developers.google.com/web/tools/chrome-devtools)

The Dev Tools is an IDE integrated into your browser which you use to:

* Debug javascript if something goes wrong
* Debug CSS layouts if something is positioned in an odd way
* Profile javascript when something is slow (e.g. when making a bug report to Vaadin Grid)

If you do not understand something in the text below, please refer to the Dev Tools
guides above.

### Quick tips

Press F12 in your browser to fire up dev tools, then head to the `Network` tab and reload
the page. It should download a bunch of files with the HTTP 200 OK result code, most importantly
the `webcomponents-loader.js`, `vaadin-bundle-*.js` and `client-*.cache.js`:

![dev_tools_network.png]({{ site.baseurl }}/images/2020-6-26/dev_tools_network.png)

If those files fail to download then Vaadin Client side will not initialize at all. Most
common incorrect HTTP codes are:

* 404 NOT FOUND: VaadinServlet is mapped to a different context root, or not mapped at all, or not activated by the servlet container at all.
* 403 FORBIDDEN: Make sure your Spring Security allows those files to pass through.

If everything looks okay in the "Network" tab, go into the "Console" tab and make sure
there are no "red" errors logged in the console, preventing Vaadin from initializing.

Then, type this into the "Console" command prompt:

```javascript
customElements.get("vaadin-button")
```

It should print something like `class xyz { constructor }`: that means that the
`vaadin-button` web component is registered properly. However, if it prints
`undefined` then there's something very wrong going on and Vaadin hasn't been initialized
on the client-side properly.

### My component doesn't work

It's a good idea to confirm that your web component was loaded properly, via the
JavaScript Console:

```javascript
customElements.get("my-component")
```

If this prints `undefined` then:
* Verify that at least Vaadin has loaded properly, by trying `customElements.get("vaadin-button")` as described above.
* If yes, perhaps the webpack bundle was not rebuilt. Try:
   * restart your server
   * perform the Vaadin Dance
   * try to also check that the npm module is present in `package.json`

### Others

Let me know and I'll add more tips.
