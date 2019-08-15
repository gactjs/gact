# The Component State Chimera

- [Introduction](#introduction)
- [What is Component State?](#what-is-component-state)
- [Component State is a Chimera](#component-state-is-a-chimera)
  - [State's Broad Implications](#states-broad-implications)
  - [State and Instance Lifecycle Mismatch](#state-instance-lifecycle-mismatch)
- [State Hoisting](#state-hoisting)
  - [Extending State Access](#extending-state-access)
  - [Extending State Lifecycle](#extending-state-lifecycle)
- [State Hoisting Consequences](#state-hoisting-consequences)
  - [Subverts the Component Model](#subverts-the-component-model)
  - [Obscures State](#obscures-state)
  - [Transition Rule Disintegration](#transition-rule-disintegration)
  - [Inefficient Reconciliation](#inefficient-reconciliation)
  - [Memory Overhead](#memory-overhead)
  - [It Only Gets Worse](#it-only-gets-worse)
- [Conclusion](#conclusion)

<a name="introduction"></a>

## Introduction

All the current component-based frameworks have a notion of component state. A careful examination of the nature of component state exposes component state as a chimera. This truth has been concealed by the employment of state hoisting. Superficially, state hoisting seems like a resolution to component state's limitations. In reality, state hoisting is the source of much of the frustration with today's frameworks.

<a name="what-is-component-state"></a>

## What is Component State?

Component state is state declared by a component and owned by each instance of this component.

Component state is tied to an instance:

- The state lifecycle mirrors the instance lifecycle: it's created when the instance is mounted and destroyed when the instance is unmounted.
- Access to the state is limited to the instance:
  - only the instance itself has direct access
  - indirect access can be provided to descendants

Let's look at an example in React:

```javascript
// pulled from https://reactjs.org/docs/forms.html
class Reservation extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      isGoing: true,
      numberOfGuests: 2
    };

    this.handleInputChange = this.handleInputChange.bind(this);
  }

  handleInputChange(event) {
    const target = event.target;
    const value = target.type === "checkbox" ? target.checked : target.value;
    const name = target.name;

    this.setState({
      [name]: value
    });
  }

  render() {
    return (
      <form>
        <label>
          Is going:
          <input
            name="isGoing"
            type="checkbox"
            checked={this.state.isGoing}
            onChange={this.handleInputChange}
          />
        </label>
        <br />
        <label>
          Number of guests:
          <input
            name="numberOfGuests"
            type="number"
            value={this.state.numberOfGuests}
            onChange={this.handleInputChange}
          />
        </label>
      </form>
    );
  }
}
```

The `Reservation` component declares a form to make a reservation. This form declares the following component state:

```javascript
this.state = {
  isGoing: true,
  numberOfGuests: 2
};
```

<a name="component-state-is-a-chimera"></a>

## Component State is a Chimera

The nature of component state is ill-fitted to most state.

<a name="states-broad-implications"></a>

### State's Broad Implications

State often has broad implications. Let's again consider our `Reservation` form. Let's say that above the form we want to display the expected cost for attending the event with the specified number of guests. The problem? The `CostEstimator` component needs access to the specified number of guests, but that state is imprisoned within the `Reservation` component.

<a name="state-instance-lifecycle-mismatch"></a>

### State and Instance Lifecycle Mismatch

In practice, the state and instance lifecycles commonly diverge. Let's consider our reservation application. The user is carefully filling out the form. As they're filling out the form, they decide to convince a buddy to join them. The user navigates to a different page with more details of the event to make a more compelling case to the aforementioned acquaintance. The user convinces their buddy, and then goes back to the reservation form. The problem? The form's reset to its initial state, and the user has to fill it out again. When the user navigated to the events details page, the component was unmounted, and its state was killed.

<a name="state-hoisting"></a>

## State Hoisting

The workaround employed whenever we discover state has broader implications or state and instance lifecycles diverge is **state hoisting**. State hoisting is the movement of state to an ancestor.

<a name="extending-state-access"></a>

### Extending State Access

We want both the `CostEstimator` and `Reservation` components to have access to `numberOfGuests`. As mentioned above, only the instance itself has direct access to component state, but indirect access can be provided to descendants. If we move `numberOfGuests` to a common ancestor, then both the `CostEstimator` and `Reservation` components can be given access to `numberOfGuests`.

Let's take a look at the code for this:

```javascript
class ReservationManagement extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      numberOfGuests: 2
    };

    this.streamNumberOfGuests = this.streamNumberOfGuests.bind(this);
  }

  streamNumberOfGuests(event) {
    this.setState({
      numberOfGuests: event.target.value
    });
  }

  render() {
    return (
      <div>
        <CostEstimator numberOfGuests={this.state.numberOfGuest} />
        <Reservation
          numberOfGuests={this.state.numberOfGuest}
          streamNumberOfGuests={this.streamNumberOfGuests}
        />
      </div>
    );
  }
}

class Reservation extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      isGoing: true
    };

    this.streamIsGoing = this.streamIsGoing.bind(this);
  }

  streamIsGoing(event) {
    this.setState({
      isGoing: event.target.checked
    });
  }

  render() {
    return (
      <form>
        <label>
          Is going:
          <input
            name="isGoing"
            type="checkbox"
            checked={this.state.isGoing}
            onChange={this.streamIsGoing}
          />
        </label>
        <br />
        <label>
          Number of guests:
          <input
            name="numberOfGuests"
            type="number"
            value={this.props.numberOfGuests}
            onChange={this.props.streamNumberOfGuests}
          />
        </label>
      </form>
    );
  }
}
```

<a name="extending-state-lifecycle"></a>

### Extending State Lifecycle

The state lifecycle mirrors the instance lifecycle. If you want to extend the state lifecycle you have to move the state to an ancestor with a longer lifecycle. The limit of state hoisting is a global store tied to the root of your application. Since the application is always mounted, the state lives as you move throughout the app.

<a name="state-hoisting-consequences"></a>

## State Hoisting Consequences

At first glance, state hoisting may seem like a fine way to deal with the limitations of component state. At a closer inspection, however, we see that state hoisting has a bevy of harmful ramifications.

<a name="subverts-the-component-model"></a>

### Subverts the Component Model

A component should encapsulate a piece of an interface, and compose well with other such pieces.

Hoisting breaks encapsulation by leaking state management details. After hoisting `numberOfGuest`, we cannot understand the `Reservation` component in isolation. We must always consider it alongside the `ReservationManagement` component.

Hoisting hinders composition by encoding assumptions about component usage. After hoisting `numberOfGuest`, we can only use `Reservation` in a context where an ancestor manages `numberOfGuest`.

<a name="obscures-state"></a>

### Obscures State

State hoisting obscures the state that impacts a given instance. We have to consider the state owned by the instance plus the state owned by all of its ancestors. Further, we have to consider how ancestors' state is transformed as it travels from ancestors to the instance in question!

<a name="transistion-rule-disintegration"></a>

### Transition Rule Disintegration

React has popularized thinking about your user interface as a function between state and view: state => view. Each state corresponds to a view. This is an accurate way to think about declarative user interfaces. However, it is an incomplete model.

The correct mathematical model of a declarative user interface is a **state machine**. A state machine is completely described by a set of states and the rules to transition between states. And yes, each state corresponds to a view. A view encodes transition rules. A **transition rule** consists of an event (e.g a user clicks a button) and the corresponding state update (e.g we increase count by 1).

We have already seen how state hoisting obscures the value of state by creating many cascading bubbles of state. Additionally, state hoisting complicates reasoning about transitions by fostering **transition rule disintegration**: distance between the two elements of a transition rule (i.e update and event).

Let's consider the transition rule for `numberOfGuests`. The event that triggers an update of `numberOfGuests` is a change to the corresponding input located in the `Reservation` component. The update triggered on that event is described in `ReservationManagement`. This disintegration makes the transition more difficult to understand, we again have to reason about `ReservationManagement` and `Reservation` simultaneously.

<a name="inefficient-reconciliation"></a>

### Inefficient Reconciliation

State hoisting promotes inefficient reconciliation.

The more we hoist state:

- the bigger the subtrees we have to reconcile
- the smaller the ratio: view that needs to be updated / view that was diffed

In summary, a minor update with minor impact forces reconciliation of large subtrees.

<a name="memory-overhead"></a>

### Memory Overhead

The shackles of component state force state and state updates to travel along paths. These paths incur memory overhead. This is simple to see, we hold the state and props of every instance in memory. The `ReservationManagement` component stores `numberOfGuests` even though it never makes any real use of its value.

<a name="it-only-gets-worse"></a>

### It Only Gets Worse

All the negative consequences of hoisting state get more dramatic the more we hoist:

- Further subversion of the component model
- Further obscured state
- Further transition rule disintegration
- Even more inefficient reconciliation
- Greater memory overhead

The `Reservation` example is really just a toy example. But consider the principles discussed above for a large form with several sections as is typical in an enterprise application.

<a name="conclusion"></a>

## Conclusion

Component state is an illusion. We pay a very steep price to maintain this illusion.
