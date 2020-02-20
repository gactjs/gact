# The Reactive Framework Accounting Problem

- [Introduction](#introduction)
- [Prior Art](#prior-art)
  - [React](#prior-art-react)
  - [Svelte](#prior-art-svelte)
- [We Should Account](#we-should-account)
  - [Optimizes Time-to-interactive](#optimizes-time-to-interactive)
  - [Optimizes Memory Resources](#optimizes-memory-resources)
  - [Need Not Affect Data Usage](#need-not-affect-data-usage)
  - [Makes Rendering A O(small view size) Problem](#makes-rendering-a-small-view-size-problem)
  - [Reduces Deep Tree Rendering](#reduces-rendering-a-deep-tree)
- [Granular Accounting](#granular-accounting)
- [Conclusion](#conclusion)

<a name="introduction"></a>

## Introduction

A reactive framework needs to synchronize view with state. The key to efficiently maintaining the view-state correspondence is to only update view that depends on updated state. This insight gives rise to the reactive framework accounting problem: what state was updated and what view depends on the updated state?

<a name="prior-art"></a>

## Prior Work

<a name="prior-art-react"></a>

### React

React is [intentionally](https://overreacted.io/react-as-a-ui-runtime/) apathetic towards accounting. React does not try to figure out which state elements in particular have changed. React also does not try to figure out which parts of the view depend on which state elements.

React's answer to the accounting problem is predicated on component boundaries. The view of the instance is dependent on the instance's state. If an instance's state changes in any way, then React fully reconciles the instance's tree.

Let's look at an [example](https://codesandbox.io/s/serverless-field-5uudh):

```javascript
function App() {
  let [a, setA] = useState(0);
  let [b, setB] = useState(0);
  return (
    <div>
      <p onClick={() => setA(a + 1)}>{a}</p>
      <p onClick={() => setB(b + 1)}>{b}</p>
    </div>
  );
}
```

Regardless of which `p` we click, React will call `App` and create a virtual node completely describing `App`'s updated view.

<a name="prior-art-svelte"></a>

### Svelte

Svelte tracks changes by tracking assignments. Unfortunately, assignment tracking is an insufficient mechanism for granular accounting. Svelte knows that a value changed, but it does not know which part of the value changed.

The following example demonstrates the inefficiency that arises with loose accounting:

```html
<script>
let a = Symbol("a");
let b = Symbol("b");
let accounts = new Map([[a, 0], [b, 0]])

function deposit(account, amount) {
  let newBalance = accounts.get(account) + amount;
  accounts.set(account, newBalance);
  accounts = accounts;
}
</script>

<div on:click={() => { deposit(a, 10)}}>{accounts.get(b)}!</div>
```

On each click of our `div`, we deposit `10` into `a`'s account. We only display the balance of `b`. The state update has no impact on view, and no additional work is needed. However, Svelte does not recognize the null intersection.

```javascript
function deposit(account, amount) {
  let newBalance = accounts.get(account) + amount;
  accounts.set(account, newBalance);
  $$invalidate("accounts", accounts);
}
```

On each deposit, Svelte invalidates the entire `accounts` `Map`.

Then it tries to figure out if the updated `accounts` necessitates DOM updates:

```javascript
p(changed, ctx) {
    if ((changed.accounts) && t1_value !== (t1_value = ctx.accounts.get(ctx.b) + "")) {
        set_data(t1, t1_value);
    }
}
```

Another issue with using only assignment tracking is it forces the developer to superfluously assign values. In the example above:

```javascript
accounts.set(account, newBalance);
accounts = accounts;
```

The more [idiomatic](https://svelte.dev/tutorial/updating-arrays-and-objects) answer to triggering assignment is more problematic. In this Twitter [thread](https://twitter.com/MateuszOkon/status/1162679000331956224), Svelte's creator defended the suggestion as promoting immutability and a matter of taste.

The problem is [immutability's benefits](https://www.youtube.com/watch?v=I7IdS-PbEgI) are irrelevant to Svelte's suggestion:

- Svelte is encouraging naive copying of values as opposed to a sophisticated strategy like structural sharing. Naive copying has a significant performance penalty.
- The immutability does not aid change tracking. In fact, it obstructs granular change tracking.
- No guarantees. Data is still mutable.
- We're dealing with a single-threaded environment.

<a name="we-should-account"></a>

## We Should Account

Sophisticated accounting is the key to optimal performance.

The React team [argues](https://overreacted.io/react-as-a-ui-runtime/) that fine-grained listeners are undesirable because they:

- increase time-to-interactive
- waste memory resources
- make data usage more complex and subtle
- make rendering a O(model size) rather than O(view size) problem
- do not help with rendering a deep tree

In the following sections, I will counter each point.

<a name="optimizes-time-to-interactive"></a>

### Optimizes Time-to-interactive

Fine-grained tracking does require overhead that impacts time-to-interactive. But the employment of fine-grained tracking enables maximally efficient reconciliation. Consequently, Gact can employ a much simpler and smaller runtime. A smaller runtime decreases time-to-interactive because less code is sent to and paresd by the browser.

A smaller runtime also improves another key metric: time-to-interactive variance. The network is intrinsically unpredictable. By decreasing bundle size we're able to make time-to-interactive more consistent.

<a name="optimizes-memory-resources"></a>

### Optimizes Memory Resources

Fine-grained tracking optimizes memory resources by enabling minimal reconciliation trees and direct handling of [value updates](https://github.com/gactjs/gact/blob/master/docs/long-live-the-virtual-dom.md).

Let's revisit our [example](https://codesandbox.io/s/serverless-field-5uudh):

```javascript
function App() {
  let [a, setA] = useState(0);
  let [b, setB] = useState(0);
  return (
    <div>
      <p onClick={() => setA(a + 1)}>{a}</p>
      <p onClick={() => setB(b + 1)}>{b}</p>
    </div>
  );
}
```

On each click, React has to create and recycle a complete virtual node. In contrast, Gact would only create the `App` virtual node once.

It is true that rerendering a large part of your view will invalidate subscriptions. However, such updates are [rare](https://overreacted.io/react-as-a-ui-runtime/): "the host tree is relatively stable and most updates don’t radically change its overall structure." Hence, on average fine-grained subscriptions will more than pay for their cost before expiration.

<a name="need-not-affect-data-usage"></a>

### Need Not Affect Data Usage

Today's attempts at fine-grained tracking have made data usage more complex (e.g Svelte's assignment based reactivity). However, Gact enables fine-graned tracking without complicating data usage.

<a name="makes-rendering-a-small-view-size-problem"></a>

### Makes Rendering A O(small view size) Problem

Fine-grained tracking does not force you to do anything with your model on render. Rendering remains O(view size) except that view size becomes much smaller.

In the [example](https://codesandbox.io/s/serverless-field-5uudh), rerendering is O(view size) for React, but free for Gact because we never need to rerender.

<a name="reduces-rendering-a-deep-tree"></a>

### Reduces Deep Tree Rendering

Granular accounting will not help if you really need to render a deep tree:

```javascript
if (path === "gangster") {
  return <Gact />;
} else {
  return <React />;
}
```

As stated, however, fine-grained tracking enables minimal reconciliation trees and direct handling of [value updates](https://github.com/gactjs/gact/blob/master/docs/long-live-the-virtual-dom.md). Hence, granular accounting reduces the depth and number of deep trees you need to render.

<a name="granular-accounting"></a>

## Granular Accounting

State is a tree of containers (e.g Array) and base elements (e.g "Gact"). Granular accounting is the idea of tracking exactly which parts of the tree are used. None of the existing reactivity systems are granular. Without granular accounting we are doomed to have imprecise answers to the accounting problem (e.g this instance uses some part of this array and some part of this array changed).

Gact employs a granular accounting system to perfectly solve the accounting problem (e.g this instance reads the third element of this array and the third element of this array changed).

Gact track reads and writes to figure out which parts of the interface need to be updated. We track usage by the path (e.g `["login", "username"]`) of the values . We know a part of our interface needs to be updated if there is a non-null intersection between the state it reads and the state written by an update.

<a name="conclusion"></a>

## Conclusion

The reactive framework accounting problem is to compute the intersection of updated state and view dependent on that state.
To build a maximally performant reactive framework, we need to perfectly solve the accounting problem. Today's frameworks have failed to optimally solve the accounting problem either because of disinterest (React) or insufficient tracking (Svelte). Gact introduces granular accounting to completely solve the reactive framework accounting problem.
