---
layout: post
title:  "Organize Redux actions"
date:   2016-01-27 07:05:02 +0100
categories: javascript redux front
author: Guillaume Claret
---
At [OuiCar](http://www.ouicar.fr/), we are migrating to the [React](https://facebook.github.io/react/)/[Redux](http://redux.js.org/) stack to manage our web front-end. While these are great libraries to write maintainable and reusable interfaces, they do not tell you how to organize your code in the large. We present our way to organize the Redux store and actions.

# Reducers
As with `combineReducers`, we split our Redux store into a flat list of independent sub-stores. However, our sub-reducers have the following shape:

{% highlight js %}
export default {
  [INIT]: (state = initialState) => state,
  [Search.LOADING_START]: (state, action) => ...,
  [Search.LOADING_SUCCESS]: (state, action) => ...,
  ...
  [OTHER]: (state, action) => ...
};
{% endhighlight %}

A sub-reducer is an object whose keys are action types and those values are plain reducers of shape `(state, action) => state`. This forces us to reason by disjunction over the action type, instead of relying on a more verbose and error-prone `switch` statement. We also introduce a special `OTHER` action type to mimick the `default` case of a `switch`. Here is the code of our "smart" reducer:

{% highlight js %}
export function smartCombineReducers(stores) {
  // Cache reverse hashmap action => reducers.
  const actionReducers = Object.keys(stores).reduce((result, storeKey) => {
    const actions = stores[storeKey];

    Object.keys(actions).forEach((actionKey) => {
      result[actionKey] = _assign({}, result[actionKey], {
        [storeKey]: actions[actionKey]});
    });

    return result;
  }, {});

  return (state = {}, action) => {
    // Do not mutate the state object.
    const newState = _assign({}, state);
    const reducers = actionReducers[action.type] || {};
    Object.keys(reducers).forEach((key) => {
      newState[key] = reducers[key](state[key], action);
    });

    // Handling of the special action `OTHER`.
    if (ACTION_OTHER in actionReducers) {
      const reducersOnOther = actionReducers[ACTION_OTHER];
      Object.keys(reducersOnOther).forEach((key) => {
        if (!reducers[key]) {
          newState[key] = reducersOnOther[key](newState[key], action);
        }
      });
    }

    return newState;
  };
}
{% endhighlight %}

# Actions

# Discussion
...
