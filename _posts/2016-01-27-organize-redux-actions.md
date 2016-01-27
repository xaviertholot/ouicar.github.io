---
layout: post
title:  "Organize Redux actions"
date:   2016-01-27 07:05:02 +0100
categories: javascript redux front
author: Guillaume Claret
---
At [OuiCar](http://www.ouicar.fr/), we are migrating to the [React](https://facebook.github.io/react/)/[Redux](http://redux.js.org/) stack to develop our web front-end. While these are great libraries, they do not tell you how to shape your code in the large. We present our way to organize the Redux store and actions.

# Reducers
As with the [`combineReducers()`](http://redux.js.org/docs/api/combineReducers.html) helper, we split our Redux store into a flat list of independent sub-stores. However, our sub-reducers have the following shape:

{% highlight js %}
export default {
  [INIT]: (state = initialState) => state,
  [Search.LOADING_START]: (state, action) => ...,
  [Search.LOADING_SUCCESS]: (state, action) => ...,
  ...
  [OTHER]: (state, action) => ...
};
{% endhighlight %}

A sub-reducer is an object whose keys are action types and those values are plain reducers of shape `(state, action) => state`. This forces us to reason by disjunction over the action type, instead of relying on a more verbose and error-prone `switch` statement. We also introduce a special `OTHER` action type to mimick the `default` case of a `switch`. Here is the code of our "smart" combiner of reducers:

{% highlight js %}
// Use it in place of combineReducers().
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
We attempt to create one type of action per event for each page. An event is for example the result of a user action or a server response. If an action implies the update of many sub-stores, its reducers appear in many sub-stores. Thus, we group our reducers by sub-store rather than by action type.

Because each action is associated to a single event, we can automatically log the trace of all the events occurring in our application by using the [redux-logger](https://github.com/fcomb/redux-logger) middleware.

# Discussion
We do not know yet how to organize the asynchronous actions (apart from using [redux-thunk](https://github.com/gaearon/redux-thunk)), or improve the sharing among reducers for common tasks such as cache handling.
