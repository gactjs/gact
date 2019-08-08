# View Tree Duality

- [Introduction](#introduction)
- [Intrinsic Tree and Instance Tree](#intrinsic-tree-and-instance-tree)
  - [Intrinsic Tree](#intrinsic-tree)
  - [Instance Tree](#instance-tree)
- [React's Reconciliation](#reacts-reconciliation)
  - [Performance](#reacts-reconciliation-performance)
  - [Instance Lifecycle](#reacts-reconciliation-and-instance-lifecycle)
  - [Example](#reacts-reconciliation-example)
- [Intrinsic Tree and Reconciliation](#intrinsic-tree-and-reconciliation)
  - [Example](#intrinsic-tree-and-reconciliation-example)
- [Instance Tree and Instance Lifecycle](#instance-tree-and-instance-lifecycle)
  - [Example](#instance-tree-and-instance-lifecycle-example)
- [Conclusion](#conclusion)

<a name="#introduction"></a>

## Introduction

In a component-based framework, the view is a tree of intrinsic elements (e.g `<div/>`) and instances (e.g. `<House/>`). The user-defined tree can be viewed as two different trees: **intrinsic tree** and **instance tree**. The intrinsic tree illuminates effective reconciliation strategies. The instance tree enables accurately computing instance lifecycle.

<a name="intrinsic-tree-and-instance-tree"></a>

## Intrinsic Tree and Instance Tree

In a component-based framework, the view is a tree of intrinsic elements (e.g `<div/>`) and instances (e.g. `<House/>`):

```javascript
function Bedroom() {
  return <div>Bedroom</div>;
}

function Bathroom() {
  return <div>Bathroom</div>;
}

function House() {
  return (
    <div>
      <Bedroom />
      <Bathroom />
    </div>
  );
}

function Car() {
  return <div>Car</div>;
}

function Village() {
  return (
    <div>
      <House />
      <Car />
      <Car />
    </div>
  );
}
```

The user-defined tree can be viewed as two different trees: **intrinsic tree** and **instance tree**.

<a name="intrinsic-tree"></a>

### Intrinsic Tree

The **intrinsic tree** is the user-defined tree described only in terms of intrinsic elements.

Intrinsic elements are the base cases of our view. The view for an instance is a tree of intrinsic elements and potentially other instances. Some of our instances are defined only in terms of intrinsic elements. If we unravel the user-defined tree, we get the intrinsic tree.

The intrinsic tree for `<Village/>` would be:

```javascript
<div>
  <div>
    <div>Bedroom</div>
    <div>Bathroom</div>
  </div>
  <div>Car</div>
  <div>Car</div>
</div>
```

<a name="instance-tree"></a>

### Instance Tree

The **instance tree** describes the instance composition of the user-defined tree.
The instance tree for `<Village/>` would be:

```javascript
<House>
    <Bedroom/>
    <Bathroom/>
</House>
<Car/>
<Car/>
```

The computation of the instance tree filters out intrinsic elements. For example, the following three user-defined trees have the same instance tree:

```javascript
<Village />
```

```javascript
<div>
  <Village />
</div>
```

```javascript
<div>
  <p>
    <Village />
  </p>
</div>
```

A noteworthy property of the instance tree is that each instance only directly shapes a single level:

```javascript
function CrazyVillage() {
  return (
    <div>
      <House />
      <Car>
        <House />
      </Car>
    </div>
  );
}
```

`<Village/>` and `<CrazyVillage/>` each add a single `<House/>` to the **instance tree**. It's up to `<Car/>` to do something with the nested `<House/>`.

<a name="reacts-reconciliation"></a>

## React's Reconciliation

<a name="reacts-reconciliation-performance"></a>

### Performance

React only considers the user-defined tree. They use the term element to refer to both an intrinsic element (e.g `<div/>`) and an instance (e.g `<Counter/>`). The core assumption of React's [reconciliation](https://reactjs.org/docs/reconciliation.html) algorithm is: "two elements of different types will produce different trees." The intuition behind this assumption is that instances of the same component will produce similar trees across all invocations. Concretely, a `<Counter/>` is unlikely to produce a radically different view from one invocation to the next. The virtue of this heuristic is you get substantial information for a single comparison. In general, the trees produced by two invocations of the same component will be considerably more similar than two arbitrary trees.

The core insight, two invocations of the same component will generally be similar, is valid. The other half of the assumption concerning intrinsic elements is problematic. The intrinsic type of a tree's root is a dubious indicator of tree similarity. By definition `<div>`s and `<span>`s will contain arbitrary trees. In the case of the other intrinsic elements, type is not a reliable signal of similarity either. For example, we really have no idea how the trees of two `<ul>`s will compare. The deficiency of this heuristic causes React to unnecessarily throw away large DOM trees. In many cases, React does not get a chance to make use of the legitimate half of their assumption. If two instances of the same component have any divergence in ancestral element types, they won't be compared.

The React assumption is myopic in another important way: two instances of different components could have similar views. For example, `<LoginForm/>` and `<SignupForm/>`. If someone moves from the login page to the signup page, you could save a lot of work with a more sophisticated strategy. The React team [suggests](https://reactjs.org/docs/reconciliation.html) the following: "if you see yourself alternating between two component types with very similar output, you may want to make it the same type." In some cases, that would be exceedingly awkward because you will merge logically distinct components. In all cases, it would increase complexity. You would need to add props and conditionals to manage all divergence (i.e view, state management, lifecycle methods) between the merged components. Further, we should avoid leaking implementation details. If React decides to change it's reconciliation algorithm, then we have a bunch of user code that needs to be updated. Finally, we don't simply want to handle cases with _very_ similar output, but all cases with useful similarity.

The shortcomings of React's heuristic reconciliation algorithm are not a secret. React justifies its use by [claiming](https://reactjs.org/docs/reconciliation.html) that the world class edit distance algorithms are too slow: "displaying 1000 elements would require in the order of one billion comparisons". However, you almost never have to compute the edit distance between trees that large. The reason is that most state updates only influence small parts of the view. If you can architect a framework so that the preponderance of reconciliation involves small subtrees, then you can use a world class edit distance algorithm.

<a name="reacts-reconciliation-and-instance-lifecycle"></a>

### Instance Lifecycle

Reconciliation strategy and instance lifecycle are fundamentally separate concerns. The algorithm used to determine how to move from the current tree to the desired tree should not influence which instances mount or unmount. The React team [acknowledges](https://reactjs.org/docs/reconciliation.html) this: "...it doesn’t mean React will unmount and remount them." Nevertheless, React currently computes instance lifecycle according to its reconciliation algorithm. If an instance experiences any divergence in ancestral element types, then React will unmount and remount an instance that never left.

<a name="reacts-reconciliation-example"></a>

### Example

```javascript
class Counter extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      count: 0
    };

    this.increment = this.increment.bind(this);
  }

  componentDidMount() {
    console.log("I love life");
  }

  componentWillUnmount() {
    console.log("I can't do this anymore");
  }

  increment() {
    this.setState({
      count: this.state.count + 1
    });
  }

  render() {
    return <span onClick={this.increment}>{this.state.count}</span>;
  }
}

class App extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      container: true
    };

    this.interval = null;
    this.toggleContainer = this.toggleContainer.bind(this);
  }

  componentDidMount() {
    this.interval = setInterval(this.toggleContainer, 1000);
  }

  componentWillUnmount() {
    clearInterval(this.interval);
  }

  toggleContainer() {
    this.setState({
      container: !this.state.container
    });
  }

  render() {
    if (this.state.container) {
      return (
        <div>
          <Counter />
        </div>
      );
    }

    return <Counter />;
  }
}
```

In this [example](https://codesandbox.io/embed/cranky-grothendieck-xtqnn), we alternate between wrapping and not wrapping a `<Counter/>` with a div every second. This provides us with the most basic example of ancestral element type divergence. Our `<App/>` continuously renders a `<Counter/>`. Despite this each second we will destroy and rebuild an identical `<Counter/>` view, reset `<Counter/>`'s state, and run `Counter`'s `componentDidMount` and `componentWillUnmount` handlers.

<a name="intrinsic-tree-and-reconciliation"></a>

## Intrinsic Tree and Reconciliation

The essential error of the React approach is it predicates reconciliation strategy on something other than the relevant intrinsic trees. The browser only cares about intrinsic elements. By comparing the current intrinsic tree with the desired intrinsic tree we can identify the DOM operations we need to legitimately perform.

<a name="intrinsic-tree-and-reconciliation-example"></a>

### Example

```javascript
function Odd(props) {
  if (props.integer % 2) {
    return props.integer;
  }

  return null;
}

function Even(props) {
  if (props.integer % 2) {
    return null;
  } else {
    return props.integer;
  }
}

class Integers extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      integer: 0
    };
    this.interval = null;
    this.next = this.next.bind(this);
  }

  componentDidMount() {
    this.interval = setInterval(this.next, 1000);
  }

  componentWillUnmount() {
    clearInterval(this.interval);
  }

  next() {
    this.setState({
      integer: this.state.integer + 1
    });
  }

  render() {
    return (
      <>
        <Odd integer={this.state.integer} />
        <Even integer={this.state.integer} />
      </>
    );
  }
}
```

In this [example](https://codesandbox.io/s/angry-saha-cfmcp),`<Integers/>` continuously renders an integer. By using intrinsic trees as the basis of our reconciliation, we can realize to update the value of the existing text node on each re-render. In contrast, the React approach destroys the text node created by `<Odd/>` and builds a new one for `<Even/>` and vice versa every second.

<a name="instance-tree-and-instance-lifecycle"></a>

## Instance Tree and Instance Lifecycle

The only thing that should determine instance lifecycle is the instance tree. By comparing the instance composition of the current tree and desired tree we can accurately identify which instances are: mounting, updating, or umounting.

<a name="instance-tree-and-instance-lifecycle-example"></a>

### Example

Let's take another look at our [counter](https://codesandbox.io/embed/cranky-grothendieck-xtqnn) example. The instance tree for `<App/>` is always:

```javascript
<Counter />
```

This reflects the fact that `<Counter/>` is rendered continuously. If we used the instance tree to compute instance lifecycle, the `<Counter/>` would correctly update each second instead of unmount and mount.

<a name="conclusion"></a>

## Conclusion

Any reconciliation strategy that ignores the intrinsic tree misses many opportunities to reuse DOM. Any computation of instance lifecycle that ignores the instance tree is inaccurate. In conclusion, the user-defined tree must be viewed as both the intrinsic tree and instance tree.
