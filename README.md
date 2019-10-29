# vuex-store-builder

[![npm version](https://badge.fury.io/js/vuex-store-builder.svg)](https://badge.fury.io/js/vuex-store-builder) [![Build Status](https://travis-ci.org/Path-AI/vuex-store-builder.svg?branch=master)](https://travis-ci.org/Path-AI/vuex-store-builder) [![codecov](https://codecov.io/gh/Path-AI/vuex-store-builder/branch/master/graph/badge.svg)](https://codecov.io/gh/Path-AI/vuex-store-builder)

Customizable utility functions to generate vuex stores that facilitate network requests.

### About

`vuexStoreBuilder` is a convenience function that generates a Vuex store configuration containing an action that makes a network call. This allows developers to offload weighty vuex configuration syntax onto this utility.

This is mostly intended to minimize the amount of manually written code necessary to interface with a RESTful API.

This library was developed on top of a RESTful backend, using the [Axios](https://github.com/axios/axios) library to make requests.

### Usage

#### Installation

Install with `npm`:

```bash
npm install vuex-store-builder --save
```

#### Types

Currently, this library is provided in Javascript.
However, it will be provided in Typescript in the near future.

In the meantime, the types for the default exported function would be something like as follows:

```typescript
function vuexStoreBuilder(
  slug: String,
  call: Function(params?: Object): Promise<Object | Array>,
  overrides?: Object, // Please see the Overrides section of the readme
): {
  namespaced: Boolean,
  state: Object,
  actions: Object<String, Function>,
  mutations: Object<String, Function>,
  getters: Object<String, Function>,
} {
  // Please see source code for further implementation details
}
```

## Interacting with stores generated by this utility

This library uses string builders to provide a semantic, readable, and normalized approach to mutation naming within a namespaced store module.

Committing or dispatching inline string literals is discouraged, as maintaining such strings is more difficult than using the strings derived from functions.

#### Mutations

Two groups of mutation string builders are exported from `./mutations`:

Specifically related to mutations called by actions, facilitating network calls

- requested
- received
- errored
- failed

More general mutations

- loaded
- update
- increment
- clear

## Interacting with stores provided by vuexStoreBuilder

At minimum, `vuexStoreBuilder` requires two arguments, and consumes an optional third.

```javascript
import vuexStoreBuilder from "vuex-store-builder";
import getListOfCats from "api";

const cats = vuexStoreBuilder(
  "get", // 🐈
  getListOfCats
);

export default cats;
```

With no customization, `vuexStoreBuilder` will create a store configuration that will hold data records in an Object, using the `id` attribute of each record as its key.

There will be a getter that will return an array representation of this data.

Three mutations will be generated as well. These mutations are

- `requested(slug)`
- `received(slug)`
- `failed(slug)`

`requested(slug)` handles individual records as well as lists of records. In the case of receiving a single `record` from the server, the function will insert `record` into `state.slug` with `record.id` as its key.
If the server returns a list of records, the function will iterate through said list, inserting each record into `state.slug` using each records' id as the key.

Please see this example for more specifics about how the generated action and mutations will interact with state.

## Example

```javascript
import vuexStoreBuilder from "vuex-store-builder";
import { getListOfCats } from "api/cats"; // 🐈

export const cats = vuexStoreBuilder("get", getListOfCats);

export default cats;
```

The object exported by the above example will equivalent to the following store configuration:

```javascript
import { getListOfCats } from "api/cats";

export const cats = {
  namespaced: true,
  state: {
    byId: {},
    loadedGet: false,
    erroredGet: undefined
  },
  getters: {
    list: state => Object.values(state.byId),
    errors: state => state.erroredGet.errors
  },
  mutations: {
    requestedGet(state, params) {
      state.loadedGet = false;
      state.erroredGet = {};
    },
    receivedGet(state, { response, params }) {
      response.forEach(datum => Vue.set(state.byId, datum.id, datum));
      state.loadedGet = true;
    },
    failedGet(state, errors) {
      state.erroredGet = errors;
    }
  },
  actions: {
    async get({ commit }, params) {
      let response;
      commit("requestedGet", params);
      try {
        response = await getListOfCats(params);
      } catch (error) {
        commit("failedGet", error);
        return error;
      }
      commit("receivedGet", { response, params });
      return response;
    }
  }
};

export default cats;
```

In order to set a debugger or breakpoint inside of the action or mutation functions, that debugger or breakpoint would have to be set inside of the utility functions.

## Custom behavior

Most of the default configuration can be overridden through the optional options argument.

```javascript
import vuexStoreBuilder, { strings } from "vuex-store-builder";
import { getListOfDogs } from "api/dogs";

export const getDogId = ({ dogId }) => dogId;
export const getOwnerId = ({ owner: { ownerId } }) => ownerId;

export default vuexStoreBuilder("get", getListOfDogs, {
  getKey: getDogId,
  state: {
    ownerIdToDogIds: {}
  },
  mutations: {
    clearDogs(state) {
      state.byId = {};
      state.ownerIdToDogIds = {};
    }
  },
  receive(state, { response, params }) {
    response.forEach(dog => {
      const dogId = getDogId(dog);
      state.byId[dogId] = dog;

      const ownerId = getOwnerId(dog);
      if (!state.ownerIdToDogIds) {
        state.ownerIdToDogIds[ownerId] = [];
      }
      state.owneridToDogIds[ownerId].push(dogId);
    });
    state[strings.loaded("get")] = true;
  },
  actions: {
    clearAndGetDogs({ dispatch, commit }) {
      commit("clearDogs");
      return dispatch("get");
    }
  }
});
```

### List of provided overrides

Overrides are provided through the third argument of the `vuexStoreBuilder` function.

| Attribute             | Type     | Behavior                                                                                                                                                                                                                           |
| --------------------- | -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| getKey                | Function | This function extracts an identifier from each record. This identifier will be used to store records in state.                                                                                                                     |
| requestedMutationName | String   | This string is what will be used to define and commit the requested mutation.                                                                                                                                                      |
| receivedMutationName  | String   | This string is what will be used to define and commit the received mutation.                                                                                                                                                       |
| failedMutationName    | String   | This string is what will be used to define and commit the failed mutation.                                                                                                                                                         |
| actionBuilder         | function | A function that takes `slug` and returns an action that takes `params`.                                                                                                                                                            |
| action                | Function | If the default generated action is not sufficient, it can be overridden by providing the desired function here.                                                                                                                    |
| request               | Function | This function replaces the requested mutation.                                                                                                                                                                                     |
| receive               | Function | This function replaces the received mutation.                                                                                                                                                                                      |
| fail                  | Function | This function replaces the failed mutation                                                                                                                                                                                         |
| error                 | Function | This is a function that is a state factory for the default error state.                                                                                                                                                            |
| state                 | Object   | This object will be spread into state following the default state. To override a default state attribute, simply include a key-value pair in this object where the key is what you'd like to override.                             |
| mutations             | Object   | This object will be spread into the mutations object following the default mutations. Overriding default behavior can be done in the same manner as overriding default state. Entities in this object should be unbound Functions. |
| actions               | Object   | This object will be spread into the collection of actions following the default action. Again, entities in this collection should be unbound Functions.                                                                            |
| getters               | Object   | This object will be spread into the getters following the default getters. These functions can be bound or unbound. If you desire access to `this`, the function should be unbound.                                                |
| modules               | Object   | This object will be passed into the generated store config to define submodules.                                                                                                                                                   |
