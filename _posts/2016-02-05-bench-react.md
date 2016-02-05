---
layout: post
title:  "Optimize React rendering"
date:   2016-02-05 07:05:02 +0100
author: Guillaume Claret
---
In order to minimize the number of DOM updates, [React](https://facebook.github.io/react/) uses a [reconciliation algorithm](https://facebook.github.io/react/docs/reconciliation.html). To optimize the rendering even further, we can use [pure components](https://facebook.github.io/react/docs/pure-render-mixin.html) which are re-rendered only if their props change. All of that seems to work well, but in practice we can encounter terrible slowdowns making the interface unusable. This is generally due to components which are needlessly re-rendered.

We will see how to measure the performance of a React application, to understand which components are re-rendered too much, and present some common pitfalls.

# Measuring performance
Facebook proposes the [perf add-on](https://facebook.github.io/react/docs/perf.html) to print a performance evaluation in a given interval of time. Adding the following component in your page:

{% highlight js %}
import React, {Component} from 'react';
import Perf from 'react-addons-perf';

export default class Performance extends Component {
  onStart() {
    Perf.start();
    console.log('Perf started');
  }

  onStop() {
    Perf.stop();
    console.log('Perf stopped');
    const lastMeasurements = Perf.getLastMeasurements();
    Perf.printDOM(lastMeasurements);
    Perf.printInclusive(lastMeasurements);
    Perf.printExclusive(lastMeasurements);
    Perf.printWasted(lastMeasurements);
  }

  render() {
    return (
      <div>
        <p>Performance</p>
        <button onClick={this.onStart}>Start</button>
        <button onClick={this.onStop}>Stop</button>
      </div>
    );
  }
}
{% endhighlight %}

will show you two buttons:

![Performance buttons](../../../assets/2016-02-05-bench-react/performance_buttons.png)

With *Start* we start the recording of performance. With *Stop* we display the results:

![Results](../../../assets/2016-02-05-bench-react/performance_logs.png)

These results are the outputs of the commands `printDOM()`, `printInclusive()`, `printExclusive()` and `printWasted()`. The most important table is the last one: it shows the time wasted by computing a component's DOM which is identical to the previous one. In this example, the text inputs of our forms were unusable due to rendering lags. We presume that the `CarCard` component is the source of the problem.

# Common pitfalls
The most common source of slowdowns is the creation of fresh properties at each re-render, defeating the props comparison of pure components. See this list of [common performance anti-patterns](https://medium.com/@esamatti/react-js-pure-render-performance-anti-pattern-fb88c101332f).

In our example, the mistake is to compute the `car` property of `<CarCard car={...} />` by creating an new object from our Redux store. Every time the store is updated, a new `car` object is created and the pure component `<CarCard car={car} />` is re-rendered. Indeed, to decide if a pure component should update, React compares its new properties according to the `===` equality. By creating new car objects, we create different objects for `===`, even if the values of the fields are the same.

As a solution, we could replace the shallow equality `===` with a deep equality like [`_.isEqual`](https://lodash.com/docs#isEqual) from Lodash. However, this equality is too slow to compute at each rendering, as it is proportional to the size of the properties. A better solution is to somehow memoize the car properties, to prevent the creation of identical car objects. Some libraries such as [Reselect](https://github.com/rackt/reselect) are made exactly for that.

React itself is also a great memoization library, for both the DOM and the props. Moreover, React handles the memoization of arrays of values, thanks to the [`key`](https://facebook.github.io/react/docs/reconciliation.html#keys) attribute. Instead of doing:

{% highlight js %}
class SearchMock extends PureComponent {
  importCar(rawCar) {
    return {
      ...
    };
  }

  render() {
    const {rawCars} = this.props;
    return (
      <div>
        {rawCars.map(rawCar => <CarCard key={rawCar.id} car={this.importCar(rawCar)} />)}
      </div>
    );
  }
}
{% endhighlight %}

we add an intermediate component `CarCardMemo`:

{% highlight js %}
class CarCardMemo extends PureComponent {
  importCar(rawCar) {
    return {
      ...
    };
  }

  render() {
    const {rawCars} = this.props;
    return <CarCard car={this.importCar(rawCar)} />;
  }
}

class SearchMock extends PureComponent {
  render() {
    const {rawCars} = this.props;
    return (
      <div>
        {rawCars.map(rawCar => <CarCardMemo key={rawCar.id} rawCar={rawCar} />)}
      </div>
    );
  }
}
{% endhighlight %}

The `CarCardMemo` component memoizes the result of the `importCar()` function, since it re-renders only when the `rawCar` object changes. Thanks to the diff algorithm of React, even when we insert new elements in the `rawCars` array, older cars are not re-computed.

Finally, we measure a new performance:

![New results](../../../assets/2016-02-05-bench-react/performance_logs2.png)

meaning that we solved our rendering problem.
