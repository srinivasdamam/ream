<p align="center">
<img src="./assets/REAM.png" alt="ream" width="100" />
</p>

<p align="center"><br><a href="https://npmjs.com/package/ream"><img src="https://img.shields.io/npm/v/ream.svg?style=flat" alt="NPM version"></a> <a href="https://npmjs.com/package/ream"><img src="https://img.shields.io/npm/dm/ream.svg?style=flat" alt="NPM downloads"></a> <a href="https://circleci.com/gh/ream/ream"><br/><img src="https://img.shields.io/circleci/project/ream/ream/master.svg?style=flat" alt="Build Status"></a> <a href="https://codecov.io/gh/ream/ream"><img src="https://codecov.io/gh/ream/ream/branch/master/graph/badge.svg" alt="codecov"></a> <a href="https://codeclimate.com/github/ream/ream"><img src="https://codeclimate.com/github/ream/ream/badges/gpa.svg" /></a>
 <a href="https://github.com/egoist/donate"><img src="https://img.shields.io/badge/$-donate-ff69b4.svg?maxAge=2592000&amp;style=flat" alt="donate"></a></p>

<br>

> Build server-rendered apps using Vue.js

## Introduction

Server-side rendered Vue.js app should be made easy, since vue-router is well optimized for SSR, we built ream on the top of it to make you build universal Vue.js app fast with fewer trade-offs, the only requirement is to export router instance in your entry file, which means you have full control of vue-router as well!

You can [try ream with the online playground!](https://glitch.com/~ream)

## Install

```bash
yarn add ream
```

## Usage

Add npm scripts:

```js
{
  "scripts": {
    "build": "ream build",
    "start": "ream start",
    "dev": "ream dev"
  }
}
```

Then populate an `src/index.js` in current working directory and it should export at least `router` instance:

```js
// your vue router instance
import router from './router'

export default { router }
```

Run `npm run dev` to start development server.

To run in production server, run `npm run build && npm start`

### Root component

By default we have a [built-in root component](https://github.com/ream/ream/blob/master/app/App.vue), you can export a custom one as well:

```js
// src/index.js
import App from './components/App.vue'

export default { App }
```

The `App` component will be used in creating Vue instance:

```js
new Vue({
  render: h => h(App)
})
```

### Vuex

You don't have to use Vuex but you can, export Vuex instance `store` in `src/index.js` to enable it:

```js
import store from './store'

export default { store }
```

#### preFetch

Every router-view component can have a `preFetch` property to pre-fetch data to fill Vuex store on the server side.

```js
export default {
  preFetch({ store }) {
    return store.dispatch('asyncFetchData')
  }
}
```

If the action you want to perform in `preFetch` method is async, it should return a Promise.

`preFetch` will be executed after first paint on client-side as well.

Arguments:

- `route`: Current route in vue-router
- `store`: Vuex store instance if any
- `isServer`: Is server-side
- `isClient`: Is client-side

### asyncData

If you want to pre-fetch data but don't want to keep it at a global store like `Vuex`, you can use `asyncData`, it's similar to Next.js's `getInitialProps` but works in the Vue way. You can treat it as an async version of `data` property but it does not have access to component instance:

```vue
<template>
  <div class="page">
    {{ user.username }}
  </div>
</template>

<script>
  export default {
    asyncData({ route }) {
      return axios.get(`https://my-api.com/user/${route.params.username}`).then(res => {
        return {
          user: res.data
        }
      })
    }
  }
</script>
```

Same as `preFetch`, this will be executed after first paint on client-side as well.

Arguments:

- `route`: Current route in vue-router
- `store`: Vuex store instance if any
- `isServer`: Is server-side
- `isClient`: Is client-side

### Progress bar

When you're using `preFetch` or `asyncData` method, you will always need a progress bar to show the loading progress as well.

This might the easiest part, since you can achieve it like a pro by using nprogress in router's hooks.

```js
if (process.env.BROWSER) {
  const nprogress = require('nprogress')
  require('nprogress/nprogress.css')

  router.beforeEach((from, to, next) => {
    nprogress.start()
    next()
  })
  router.afterEach(() => {
    nprogress.done()
  })
}
```

Since you would never want to display progress bar on server-side (and you can't), you need to call it only when `process.env.BROWSER` is `true`.

### Modify `<head>`

`ream` uses [vue-meta](https://github.com/declandewet/vue-meta) under the hood, so you can just set `head` property on Vue component to provide custom head tags:

```js
export default {
  head: {
    title: 'HomePage'
  }
}
```

Check out [vue-meta](https://github.com/declandewet/vue-meta) for details, its usage is the same here except that we're using `head` instead of `metaInfo` as key name.

### Handlers

You can implement your own `preFetch` method by using `handlers`, let's call it `preLoadHandler`:

```js
// src/index.js
function preLoadHandler({ 
  router, 
  store, 
  isServer,
  deliverData
}) {
  isServer && router.beforeEach((to, from, next) => {
    Promise.all(getMatchedComponents(to.matched).map(component => {
      if (component.preLoad) {
        return component.preLoad({ store })
      }
    })).then(() => {
      deliverData({ state: store.state })
      next()
    })
  })

  if (!isServer) {
    store.replaceState(window.__REAM__.data.state)
  }
}

function getMatchedComponents(route) {
  let res = []
  for (const record of route) {
    res.push(record.component)
    if (record.children) {
      res = [...res, ...getMatchedComponents(record.children)]
    }
  }
  return res
}

export default { router, store, handlers: [preLoadHandler] }
```

Arguments of `handler` function:

- router: vue-router instance
- store: vuex instance (only exists if you exported it in entry file)
- isServer: 
- isClient
- deliverData: pass data down from server to client, eg: `deliverData({foo: 123})` then it will be available as `window.__REAM__.data.foo`

### webpack

#### Code split

You can use `import()` or `require.ensure()` to split modules for lazy-loading.

#### JS

JS is transpiled by Babel using [babel-preset-vue-app](https://github.com/egoist/babel-preset-vue-app), which means you can use all latest ECMAScript features and stage-2 features.

We automatically load Babel config by default.

#### CSS

Support all CSS preprocessors, you can install its loader to use them, for example to use `scss`

```js
yarn add sass-loader node-sass --dev
```

We automatically load PostCSS config by default.

#### Public folder

`./dist` folder is served as static files, and files inside `./static` will be copied to `./dist` folder as well.

`./public` folder is also served as static files.

#### Development

Hot Reloading enabled

#### Production

3rd-party libraries are automatically extracted into a single `vendor` chunk.

All output files are minifies and optimized.

### Generate static website

When you visit a web page, the only context we need it a `URL` at some point, so you can just provide an array of routes and we will generated corresponding static pages for you:

```js
// ream.config.js
module.exports = {
  generate: {
    routes: ['/', '/user/egoist/', '/user/rem']
  }
}
```

Check out more usage in its [API](/api#app-generateoptions).

## Production deployment

To deploy, you need to build before running production server:

```bash
ream build
ream start
```

For example, to deploy with [now](https://zeit.co/now) a package.json like follows is recommended:

```json
{
  "name": "my-app",
  "dependencies": {
    "ream": "latest"
  },
  "scripts": {
    "dev": "ream dev",
    "build": "ream build",
    "start": "ream start"
  }
}
```

Then run `now` and enjoy! `now` will automically run `npm run build` before `npm start`.

Note: we recommend putting `.ream` folder in your `.gitignore` file, since the generated files are populated in `.ream/dist` folder.

## FAQ

### Here's a missing feature!

**"Can you update webpack config *this way* so I can use that feature?"** If you have the same question, before we actually think this feature is necessary and add it, you can [extend webpack config](#extendwebpack) yourself to implement it. With [webpack-chain](https://github.com/mozilla-rpweb/webpack-chain) you have full control of our webpack config, check out the default [config instance](https://github.com/ream/ream/blob/master/lib/create-config.js).

### How big is it?

The runtime bundle (Vue + vue-router) is around 30KB gzipped.

You can replace `127.0.0.1` with the hostname you're actually running at, by default the server we're running in `ream dev` and `ream start` command runs at `0.0.0.0` which means all IPv4 addresses on the local machine.

## Contributing

1. Fork it!
2. Create your feature branch: `git checkout -b my-new-feature`
3. Commit your changes: `git commit -am 'Add some feature'`
4. Push to the branch: `git push origin my-new-feature`
5. Submit a pull request :D


## Author

**ream** © [egoist](https://github.com/egoist), Released under the [MIT](https://github.com/ream/ream/blob/master/LICENSE) License.<br>
Authored and maintained by egoist with help from contributors ([list](https://github.com/ream/ream/contributors)).

> [egoistian.com](https://egoistian.com) · GitHub [@egoist](https://github.com/egoist) · Twitter [@rem_rin_rin](https://twitter.com/rem_rin_rin)
