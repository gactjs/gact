# Long Live the Virtual DOM

- [Introduction](#introduction)
- [Value vs Structural Updates](#value-vs-structural-updates)
- [How to Handle Value Updates](#how-to-handle-value-updates)
- [How to Handle Structural Updates](#how-to-handle-structural-updates)
- [A Compiler Cannot Do Better](#a-compiler-cannot-do-better)
- [Conclusion](#conclusion)

<a name="#introduction"></a>

## Introduction

[Svelte](https://svelte.dev/blog/virtual-dom-is-pure-overhead) claims that the "Virtual DOM is pure overhead". This claim has received widespread acceptance. Even the React team focuses their defense of the Virtual DOM on the possibilities it enables (e.g. Concurrent React). Nevertheless, the claim by Svelte is unequivocally false. The key to this realization is a distinction between the two classes of updates that need to be performed by a reactive framework: value and structural. In the case of value updates, the Virtual DOM is indeed the wrong tool. In the case of structural updates, the Virtual DOM is indispensable.

<a name="#value-vs-structural-updates"></a>

## Value vs Structural Updates

There are two classes of updates that need to be performed by a reactive framework:

- **value**: updates that change the value of some part of the DOM (e.g the value of an attribute, the value of a text node)
- **structural**: updates that change the structure of the DOM (e.g change the number of nodes, the types of nodes)

<a name="#how-to-handle-value-updates"></a>

## How to Handle Value Updates

Virtual DOM reconciliation is inapt for value updates. Virtual DOM reconciliation consists of creating a virtual representation of the desired tree and comparing it to the current tree. Since the trees are structurally identical this is almost completely wasted effort. The way to handle value updates is to keep track of the parts of the DOM that depend on a value, and update those parts when the value changes. This is possible because the tree structure remains unchanged.

Let's compare how React and Svelte handle a form with a single field:

```javascript
class SimpleForm extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      value: ""
    };

    this.handleChange = this.handleChange.bind(this);
  }

  handleInputChange(event) {
    this.setState({
      value: event.target.value
    });
  }

  render() {
    return <input value={this.state.value} onChange={this.handleChange} />;
  }
}
```

`onChange` React will create a virtual node representing the new view for `SimpleFrom`. It will look like this:

```javascript
let vnode = {
  nodeName: "input",
  attributes: {
    value: "...",
    onChange: onChangeHandler
  },
  children: []
};
```

React will then compare the vnode with the current tree, and realize that it only needs to update the `value` attribute.

Let's contrast this with an equivalent example in Svelte:

```
<script>
let value = ""

function handleChange(event) {
    value = event.target.value;
 }
 </script>

<input on:change={handleChange} bind:value />
```

`onChange` Svelte will directly update the `value` attribute.

<a name="#how-to-handle-structural-updates"></a>

## How to Handle Structural Updates

Virtual DOM reconciliation is apt for structural updates. We can input virtual representations of the current and desired trees to an edit distance algorithm to compute an optimal edit script (most efficient set of operations needed to transform one tree into another). Without these virtual representations, we have no basis to form a reconciliation strategy. Thus, our only option is to follow the pessimal edit script: completely destroy the current tree and completely build the desired tree. The more similar the trees, the more wasted effort. If cost(compute optimal edit script) + cost(optimal edit script) < cost(pessimal edit script), then the Virtual DOM is an optimization.

In the example below, we alternate between wrapping and not wrapping an identical paragraph with a div every second:

```
<script>

import { onMount } from 'svelte';

let container = false;

function toggleContainer() {
    container = !container
}

onMount(() => {
    let interval = setInterval(toggleContainer, 1000);

    return () => clearInterval(interval);
});
</script>

{#if container}
	<div><p>surgical</p></div>
{:else}
	<p>surgical</p>
{/if}
```

We only ever need to create the paragraph containing the word "surgical" once since it never leaves the screen. However, this is not what Svelte does. Each second Svelte will completely destroy the tree from the previous second, and completely rebuild the tree for this second. Not exactly surgical (at least, not the work of any surgeon I'd want operating on me).

<a name="#a-compiler-cannot-do-better"></a>

## A Compiler Cannot Do Better

Your first reaction to Svelte's failure to handle conditionals efficiently may be that the compiler needs to be more clever. But how?

You need to compute optimal edit scripts between views. The work required to transform one branch of the conditional to another is different for each pair of branches. You also have to compute the way to transition from no view to each branch. In total, you have to compute `(n * (n-1)) + n` different transitions where `n` is the number of branches.

For a 3 branch conditional (i.e if / else if / else) that's 9 transitions:

- 0 -> 1
- 0 -> 2
- 0 -> 3
- 1 -> 2
- 1 -> 3
- 2 -> 3
- 2 -> 1
- 3 -> 1
- 3 -> 2

For a 4 branch conditional (i.e if / else if / else if / else) that's 16 transitions:

- 0 -> 1
- 0 -> 2
- 0 -> 3
- 0 -> 4
- 1 -> 2
- 1 -> 3
- 1 -> 4
- 2 -> 1
- 2 -> 3
- 2 -> 4
- 3 -> 1
- 3 -> 2
- 3 -> 4
- 4 -> 1
- 4 -> 2
- 4 -> 3

How will we determine the minimal set of changes needed to move from one branch to another? That's right, the Virtual DOM and a tree edit distance algorithm. The problem with all this? The compiler will slow down dramatically and your bundles will explode.

<a name="#conclusion"></a>

## Conclusion

The distinction between value and structural updates is the foundation of the best reconciliation strategy. The best way to handle value updates is to keep track of the parts of the DOM that depend on a value, and update those parts when the value changes. The best way to handle structural updates is to use the Virtual DOM and an edit distance algorithm to compute the optimal edit script. The Virtual DOM is fundamental to the optimization of structural updates. A compiler too would need to use a Virtual DOM for optimal structural updates. However, because the number of transitions scales quadratically with the number of branches in a conditional, optimal structural updates are intractable for a compiler. In conclusion, the best reconciliation strategy will use the Virtual DOM and compute edit scripts at runtime.
