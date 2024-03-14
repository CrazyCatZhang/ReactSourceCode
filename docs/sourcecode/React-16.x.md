## React 16.x

### JSX converted to ReactElement

JSX is compiled by Babel into calls to the React.createElement method. The createElement method, upon being called, returns a ReactElement, which is essentially the Virtual DOM.

`location: v16.13.1/react/src/ReactElement.js`

###### createElement

```javascript
/**
 * Create and return a new ReactElement of the given type.
 * See https://reactjs.org/docs/react-api.html#createelement
 *
 * 1. Separate props attributes and special attributes
 * 2. Mount child elements into props.children
 * 3. Resolve default props
 * 4. Created and return ReactElement
 */
export function createElement(type, config, children) {
  let propName;

  // Reserved names are extracted
  const props = {};

  // Special attributes are extracted
  let key = null;
  let ref = null;
  let self = null;
  let source = null;

  if (config != null) {
    if (hasValidRef(config)) {
      ref = config.ref;

      if (__DEV__) {
        warnIfStringRefCannotBeAutoConverted(config);
      }
    }
    if (hasValidKey(config)) {
      key = "" + config.key;
    }

    self = config.__self === undefined ? null : config.__self;
    source = config.__source === undefined ? null : config.__source;

    // Remaining properties are added to a new props object
    for (propName in config) {
      if (
        hasOwnProperty.call(config, propName) &&
        !RESERVED_PROPS.hasOwnProperty(propName)
      ) {
        props[propName] = config[propName];
      }
    }
  }

  // Children can be more than one argument, and those are transferred onto
  // the newly allocated props object.
  const childrenLength = arguments.length - 2;
  if (childrenLength === 1) {
    props.children = children;
  } else if (childrenLength > 1) {
    const childArray = Array(childrenLength);
    for (let i = 0; i < childrenLength; i++) {
      childArray[i] = arguments[i + 2];
    }
    if (__DEV__) {
      if (Object.freeze) {
        Object.freeze(childArray);
      }
    }
    props.children = childArray;
  }

  // Resolve default props
  if (type && type.defaultProps) {
    const defaultProps = type.defaultProps;
    for (propName in defaultProps) {
      if (props[propName] === undefined) {
        props[propName] = defaultProps[propName];
      }
    }
  }

  // A warning will appear when using props to obtain key or ref in the development environment.
  if (__DEV__) {
    if (key || ref) {
      const displayName =
        typeof type === "function"
          ? type.displayName || type.name || "Unknown"
          : type;
      if (key) {
        defineKeyPropWarningGetter(props, displayName);
      }
      if (ref) {
        defineRefPropWarningGetter(props, displayName);
      }
    }
  }
  return ReactElement(
    type,
    key,
    ref,
    self,
    source,
    ReactCurrentOwner.current,
    props
  );
}
```

###### ReactElement

```javascript
/**
 * Factory method to create a new React element. This no longer adheres to
 * the class pattern, so do not use new to call it. Also, instanceof check
 * will not work. Instead test $$typeof field against Symbol.for('react.element') to check
 * if something is a React Element.
 *
 * @param {*} type
 * @param {*} props
 * @param {*} key
 * @param {string|object} ref
 * @param {*} owner
 * @param {*} self A *temporary* helper to detect places where `this` is
 * different from the `owner` when React.createElement is called, so that we
 * can warn. We want to get rid of owner and replace string `ref`s with arrow
 * functions, and as long as `this` and owner are the same, there will be no
 * change in behavior.
 * @param {*} source An annotation object (added by a transpiler or otherwise)
 * indicating filename, line number, and/or other information.
 * @internal
 */
const ReactElement = function (type, key, ref, self, source, owner, props) {
  const element = {
    // This tag allows us to uniquely identify this as a React Element
    $$typeof: REACT_ELEMENT_TYPE,

    // Built-in properties that belong on the element
    type: type,
    key: key,
    ref: ref,
    props: props,

    // Record the component responsible for creating this element.
    _owner: owner,
  };

  if (__DEV__) {
    // The validation flag is currently mutative. We put it on
    // an external backing store so that we can freeze the whole object.
    // This can be replaced with a WeakMap once they are implemented in
    // commonly used development environments.
    element._store = {};

    // To make comparing ReactElements easier for testing purposes, we make
    // the validation flag non-enumerable (where possible, which should
    // include every environment we run tests in), so the test framework
    // ignores it.
    Object.defineProperty(element._store, "validated", {
      configurable: false,
      enumerable: false,
      writable: true,
      value: false,
    });
    // self and source are DEV only properties.
    Object.defineProperty(element, "_self", {
      configurable: false,
      enumerable: false,
      writable: false,
      value: self,
    });
    // Two elements created in two different places should be considered
    // equal for testing purposes and therefore we hide it from enumeration.
    Object.defineProperty(element, "_source", {
      configurable: false,
      enumerable: false,
      writable: false,
      value: source,
    });
    if (Object.freeze) {
      Object.freeze(element.props);
      Object.freeze(element);
    }
  }

  return element;
};
```

### Internal implementation of `isValidElement`

```javascript
/**
 * Verifies the object is a ReactElement.
 * See https://reactjs.org/docs/react-api.html#isvalidelement
 * @param {?object} object
 * @return {boolean} True if `object` is a ReactElement.
 * @final
 */
export function isValidElement(object) {
  return (
    typeof object === "object" &&
    object !== null &&
    object.$$typeof === REACT_ELEMENT_TYPE
  );
}
```

### Architecture

The architecture of the React 16.x can be divided into three layers: scheduling layer, coordination layer, and rendering layer.

- Scheduler (scheduling layer): the priority of scheduling tasks, high-quality tasks enter the coordinator first
- Reconciler (coordination layer): build Fiber data structure, compare Fiber objects to find differences, record the DOM operations to be performed on Fiber objects
- Renderer (Rendering layer): Responsible for rendering the changed parts to the page

#### Scheduler

In the React 15.x, a loop plus recursion method is used to compare the virtualDOM. Since recursion uses JavaScript’s own execution stack, it cannot be stopped once started until the task execution is completed. If the level of the VirtualDOM tree is relatively deep, the comparison of VirtualDOM will occupy the JavaScript main thread for a long time. Since JavaScript is single-threaded and cannot perform other tasks at the same time, it cannot respond to user operations during the comparison process and cannot perform element animations instantly. , causing the page to freeze. In the React 16 version, the Javascript recursive method is abandoned for virtualDOM comparison, and loops are used to simulate recursion. Moreover, the comparison process is completed using the browser’s idle time and will not occupy the main thread for a long time. This solves the problem of page lag caused by virtualDOM comparison.

In the React 16.x,the Javascript recursive method is abandoned for virtualDOM comparison, and loop simulation recursion is used instead. Moreover, the comparison process is completed using the browser’s idle time and will not occupy the main thread for a long time. This solves the problem of page lag caused by virtualDOM comparison.

The `requestldleCallback` API is provided in the window object, which can use the browser’s idle time to perform tasks, but it also has some problems of its own. For example, not all browsers support it, and its trigger frequency is not very stable. So React finally gave up the use of `requestldleCallback`.

In React, the official implemented its own task scheduling library, which is called Scheduler. It can also execute tasks when the browser is idle, and can also set the priority of the task. High-priority tasks are executed first, and low-priority tasks are executed later.

BTW, Scheduler is stored in scheduler folder.

#### Reconciler

In the React 15.x, the coordinator and renderer execute alternately, that is, the differences are updated directly when differences are found. In the React 16 version, this situation has changed, and the coordinator and renderer no longer execute alternately. The coordinator is responsible for finding the differences. After all the differences are found, they are handed over to the renderer to update the DOM. In other words, the main task of the coordinator is to find the differences and mark the differences.

#### Renderer

The renderer synchronously performs corresponding DOM operations based on the marks marked by the coordinator for the Fiber node.

Now that the comparison process has changed from recursive to an interruptible loop, how does React solve the problem of incomplete DOM rendering when interrupting updates?

In fact, this problem does not exist at all, because during the entire process, the work of the scheduler and coordinator is completed in memory and can be interrupted, and the work of the renderer is set not to be interrupted, so it is not There is an issue with incomplete DOM rendering.

### Data Structure

#### Fiber

`location: v16.3.1/react-reconciler/ReactInternalTypes.js`

```javascript
// A Fiber is work on a Component that needs to be done or was done. There can
// be more than one per component.
export type Fiber = {|
  // These first fields are conceptually members of an Instance. This used to
  // be split into a separate type and intersected with the other Fiber fields,
  // but until Flow fixes its intersection bugs, we've merged them into a
  // single type.

  // An Instance is shared between all versions of a component. We can easily
  // break this out into a separate object to avoid copying so much to the
  // alternate versions of the tree. We put this on a single object for now to
  // minimize the number of objects created during the initial render.

  // Tag identifying the type of fiber.
  tag: WorkTag,

  // Unique identifier of this child.
  key: null | string,

  // The value of element.type which is used to preserve the identity during
  // reconciliation of this child.
  elementType: any,

  // The resolved function/class/ associated with this fiber.
  type: any,

  // The local state associated with this fiber.
  stateNode: any,

  // Conceptual aliases
  // parent : Instance -> return The parent happens to be the same as the
  // return fiber since we've merged the fiber and instance.

  // Remaining fields belong to Fiber

  // The Fiber to return to after finishing processing this one.
  // This is effectively the parent, but there can be multiple parents (two)
  // so this is only the parent of the thing we're currently processing.
  // It is conceptually the same as the return address of a stack frame.
  return: Fiber | null,

  // Singly Linked List Tree Structure.
  child: Fiber | null,
  sibling: Fiber | null,
  index: number,

  // The ref last used to attach this node.
  // I'll avoid adding an owner field for prod and model that as functions.
  ref:
    | null
    | (((handle: mixed) => void) & { _stringRef: ?string, ... })
    | RefObject,

  // Input is the data coming into process this fiber. Arguments. Props.
  pendingProps: any, // This type will be more specific once we overload the tag.
  memoizedProps: any, // The props used to create the output.

  // A queue of state updates and callbacks.
  updateQueue: mixed,

  // The state used to create the output
  memoizedState: any,

  // Dependencies (contexts, events) for this fiber, if it has any
  dependencies: Dependencies | null,

  // Bitfield that describes properties about the fiber and its subtree. E.g.
  // the ConcurrentMode flag indicates whether the subtree should be async-by-
  // default. When a fiber is created, it inherits the mode of its
  // parent. Additional flags can be set at creation time, but after that the
  // value should remain unchanged throughout the fiber's lifetime, particularly
  // before its child fibers are created.
  mode: TypeOfMode,

  // Effect
  effectTag: SideEffectTag,

  // Singly linked list fast path to the next fiber with side-effects.
  nextEffect: Fiber | null,

  // The first and last fiber with side-effect within this subtree. This allows
  // us to reuse a slice of the linked list when we reuse the work done within
  // this fiber.
  firstEffect: Fiber | null,
  lastEffect: Fiber | null,

  lanes: Lanes,
  childLanes: Lanes,

  // This is a pooled version of a Fiber. Every fiber that gets updated will
  // eventually have a pair. There are cases when we can clean up pairs to save
  // memory if we need to.
  alternate: Fiber | null,

  // Time spent rendering this Fiber and its descendants for the current update.
  // This tells us how well the tree makes use of sCU for memoization.
  // It is reset to 0 each time we render and only updated when we don't bailout.
  // This field is only set when the enableProfilerTimer flag is enabled.
  actualDuration?: number,

  // If the Fiber is currently active in the "render" phase,
  // This marks the time at which the work began.
  // This field is only set when the enableProfilerTimer flag is enabled.
  actualStartTime?: number,

  // Duration of the most recent render time for this Fiber.
  // This value is not updated when we bailout for memoization purposes.
  // This field is only set when the enableProfilerTimer flag is enabled.
  selfBaseDuration?: number,

  // Sum of base times for all descendants of this Fiber.
  // This value bubbles up during the "complete" phase.
  // This field is only set when the enableProfilerTimer flag is enabled.
  treeBaseDuration?: number,

  // Conceptual aliases
  // workInProgress : Fiber ->  alternate The alternate used for reuse happens
  // to be the same as work in progress.
  // __DEV__ only
  _debugID?: number,
  _debugSource?: Source | null,
  _debugOwner?: Fiber | null,
  _debugIsCurrentlyTiming?: boolean,
  _debugNeedsRemount?: boolean,

  // Used to verify that the order of hooks does not change between renders.
  _debugHookTypes?: Array<HookType> | null,
|};
```

#### WorkTags

WorkTags used in the Fiber in the tag attributes

`location: v16.3.1/react-reconciler/ReactWorkTags.js`

```javascript
export type WorkTag =
  | 0
  | 1
  | 2
  | 3
  | 4
  | 5
  | 6
  | 7
  | 8
  | 9
  | 10
  | 11
  | 12
  | 13
  | 14
  | 15
  | 16
  | 17
  | 18
  | 19
  | 20
  | 21
  | 22
  | 23
  | 24;

export const FunctionComponent = 0;
export const ClassComponent = 1;
export const IndeterminateComponent = 2; // Before we know whether it is function or class
export const HostRoot = 3; // Root of a host tree. Could be nested inside another node.
export const HostPortal = 4; // A subtree. Could be an entry point to a different renderer.
export const HostComponent = 5;
export const HostText = 6;
export const Fragment = 7;
export const Mode = 8;
export const ContextConsumer = 9;
export const ContextProvider = 10;
export const ForwardRef = 11;
export const Profiler = 12;
export const SuspenseComponent = 13;
export const MemoComponent = 14;
export const SimpleMemoComponent = 15;
export const LazyComponent = 16;
export const IncompleteClassComponent = 17;
export const DehydratedFragment = 18;
export const SuspenseListComponent = 19;
export const FundamentalComponent = 20;
export const ScopeComponent = 21;
export const Block = 22;
export const OffscreenComponent = 23;
export const LegacyHiddenComponent = 24;
```

#### TypeOfMode

TypeOfMode is used for the mode attribute in Fiber

`location: v16.3.1/react-reconciler/ReactTypeOfMode.js`

```javascript
export type TypeOfMode = number;

// 0 Synchronous rendering mode
export const NoMode = 0b00000;
// 1 Strict Mode
export const StrictMode = 0b00001;
//10 Asynchronous rendering transition mode
export const BlockingMode = 0b00010;
//100 Asynchronous rendering mode
export const ConcurrentMode = 0b00100;
//1000 Performance test mode
export const ProfileMode = 0b01000;
export const DebugTracingMode = 0b10000;
```

#### SideEffectTags

SideEffectTags is used for the effectTag attribute in Fiber

`location: v16.3.1/react-reconciler/ReactSideEffectTags.js`

```javascript
export type SideEffectTag = number;

// Don't change these two values. They're used by React Dev Tools.
export const NoEffect = /*                     */ 0b000000000000000;
export const PerformedWork = /*                */ 0b000000000000001;

// You can change the rest (and add more).
export const Placement = /*                    */ 0b000000000000010;
export const Update = /*                       */ 0b000000000000100;
export const PlacementAndUpdate = /*           */ 0b000000000000110;
export const Deletion = /*                     */ 0b000000000001000;
export const ContentReset = /*                 */ 0b000000000010000;
export const Callback = /*                     */ 0b000000000100000;
export const DidCapture = /*                   */ 0b000000001000000;
export const Ref = /*                          */ 0b000000010000000;
export const Snapshot = /*                     */ 0b000000100000000;
export const Passive = /*                      */ 0b000001000000000;
export const PassiveUnmountPendingDev = /*     */ 0b010000000000000;
export const Hydrating = /*                    */ 0b000010000000000;
export const HydratingAndUpdate = /*           */ 0b000010000000100;

// Passive & Update & Callback & Ref & Snapshot
export const LifecycleEffectMask = /*          */ 0b000001110100100;

// Union of all host effects
export const HostEffectMask = /*               */ 0b000011111111111;

// These are not really side effects, but we still reuse this field.
export const Incomplete = /*                   */ 0b000100000000000;
export const ShouldCapture = /*                */ 0b001000000000000;
export const ForceUpdateForLegacySuspense = /* */ 0b100000000000000;
```

### Double Buffer Technology

In React, DOM updates use double buffer technology, which is dedicated to faster DOM updates.

When we draw animations with canvas, we call ctx.clearRect before drawing each frame to clear the previous frame's image. If the current frame's computation is relatively large, leading to a significant gap between clearing the last frame and drawing the current one, a white screen may appear.

To solve this problem, we can draw the current frame's animation in memory. After the drawing is complete, we directly replace the last frame with the current one. Since this approach eliminates the computation time between two frames' replacement, it avoids the flicker from a white screen to the appearance of the image.

This technique of building in memory and directly replacing is called double buffering.

React uses double buffer technology to complete the construction and replacement of Fiber trees to achieve rapid updates of DOM objects.

In React, there will be at most two Fiber trees at the same time. The Fiber tree corresponding to the content currently displayed on the screen is called the `current` Fiber tree. When an update occurs, React will rebuild a new Fiber tree in the memory. This Fiber tree is currently The constructed Fiber tree is called the `workInProgress` Fiber tree. In double buffer technology, the `workInProgress` Fiber tree is the Fiber tree that will be displayed on the page. When the Fiber tree is built, React will use it to directly replace the `current` Fiber tree to quickly update the DOM, because the `workInProgress` Fiber tree is It’s built in memory so building it is very fast.

Once the `workInProgress` Fiber tree is rendered on the screen, it becomes the `current` Fiber tree.

There is an alternate property in the `current` Fiber node object that points to the corresponding `workInProgress` Fiber node object, and there is an alternate property in the `workInProgress` Fiber node that also points to the corresponding `current` Fiber node object.

### fiberRoot & rootFiber

fiberRoot represents the Fiber data structure object, which is the outermost object in the Fiber data structure.

rootFiber represents the Fiber object corresponding to the component mounting point. For example, the default component mounting point in a React application is the div with the id of root.

fiberRoot contains rootFiber. In fiberRoot There is a current attribute in the object, which stores rootFiber.

rootFiber points to fiberRoot. There is a stateNode attribute in the rootFiber object, which points to fiberRoot.

In a React application, there is only one FiberRoot, but there can be multiple rootFibers because the render method can be called multiple times.

fiberRoot will record the update information of the application. For example, after the coordinator completes the work, the work results will be stored in fiberRoot.

### Initial Rendering

To render React elements into the page, it is divided into two phases, the render phase and the commit phase.

The render phase is responsible for creating the Fiber data structure and marking the Fiber nodes, marking the DOM operations to be performed on the current Fiber node.

The commit phase is responsible for performing corresponding DOM operations based on the Fiber node tag (effectTag).

#### Render

##### Rendering Entry

`location: v16.13.1/react-dom/src/client/ReactDOMLegacy.js`

```javascript
/**
 * Rendering Entry
 *
 * @param {*} element The ReactElement to be rendered, the return value of the createElement method
 * @param {*} container Rendered container
 * @param {*} callback Rendered callback
 *
 */
export function render(
  element: React$Element<any>,
  container: Container,
  callback: ?Function
) {
  // Check if the container is a valid DOM element
  invariant(
    isValidContainer(container),
    "Target container is not a DOM element."
  );
  if (__DEV__) {
    const isModernRoot =
      isContainerMarkedAsRoot(container) &&
      container._reactRootContainer === undefined;
    if (isModernRoot) {
      console.error(
        "You are calling ReactDOM.render() on a container that was previously " +
          "passed to ReactDOM.createRoot(). This is not supported. " +
          "Did you mean to call root.render(element)?"
      );
    }
  }

  return legacyRenderSubtreeIntoContainer(
    null,
    element,
    container,
    false,
    callback
  );
}
```

##### isValidContainer

`location: v16.3.1/react-dom/src/client/ReactDOMRoot.js`

```javascript
/**
 *
 * Determine whether node is a DOM node that meets the requirements
 * 1. Element node
 * 2. Document node
 * 3. Document fragment node
 * 4. Annotation node but the annotation content must be react-mount-point-unstable. React will internally find the parent of the annotation node and insert the element in front of the annotation node by calling the insertBefore method of the parent element.
 *
 */
export function isValidContainer(node: mixed): boolean {
  return !!(
    node &&
    (node.nodeType === ELEMENT_NODE ||
      node.nodeType === DOCUMENT_NODE ||
      node.nodeType === DOCUMENT_FRAGMENT_NODE ||
      (node.nodeType === COMMENT_NODE &&
        (node: any).nodeValue === " react-mount-point-unstable "))
  );
}
```

##### Initialize FiberRoot

`location: v16.3.1/react-dom/src/client/ReactDOMLegacy.js`

###### legacyRenderSubtreeIntoContainer

```javascript
/**
 * Initialize Fiber data structure: create fiberRoot and rootFiber
 *
 * @param {*} parentComponent Parent component, initial rendering passed in null
 * @param {*} children The first parameter in the render method, the ReactElement to be rendered
 * @param {*} container Rendering container
 * @param {*} forceHydrate true for server-side rendering, false for client-side rendering
 * @param {*} callback The callback function that needs to be executed after the component rendering is completed
 *
 */
function legacyRenderSubtreeIntoContainer(
  parentComponent: ?React$Component<any, any>,
  children: ReactNodeList,
  container: Container,
  forceHydrate: boolean,
  callback: ?Function
) {
  if (__DEV__) {
    topLevelUpdateWarnings(container);
    warnOnInvalidCallback(callback === undefined ? null : callback, "render");
  }

  /**
   *
   * Check whether container is already an initialized rendering container
   * React will add the _reactRootContainer attribute to the outermost container during initial rendering.
   * React will perform different rendering methods based on this attribute
   * The absence of root indicates initial rendering, and the existence of root indicates update.
   *
   */
  let root: RootType = (container._reactRootContainer: any);
  let fiberRoot;
  if (!root) {
    // Initial mount
    // Initial fiberRoot & rootFiber
    root = container._reactRootContainer = legacyCreateRootFromDOMContainer(
      container,
      forceHydrate
    );

    // root._internalRoot ---> fiberRoot
    fiberRoot = root._internalRoot;

    /**
     * Change this pointer in the callback function
     * Make it point to the instance of the first parameter of the render method
     */
    if (typeof callback === "function") {
      const originalCallback = callback;
      callback = function () {
        const instance = getPublicRootInstance(fiberRoot);
        originalCallback.call(instance);
      };
    }

    // Initial mount should not be batched.
    // Because the batch update is asynchronous and can be interrupted, the initial rendering should be completed as soon as possible and cannot be interrupted.
    unbatchedUpdates(() => {
      updateContainer(children, fiberRoot, parentComponent, callback);
    });
  } else {
    fiberRoot = root._internalRoot;
    if (typeof callback === "function") {
      const originalCallback = callback;
      callback = function () {
        const instance = getPublicRootInstance(fiberRoot);
        originalCallback.call(instance);
      };
    }
    // Update
    updateContainer(children, fiberRoot, parentComponent, callback);
  }
  return getPublicRootInstance(fiberRoot);
}
```

###### legacyCreateRootFromDOMContainer

```javascript
/**
 * Determine whether it is server-side rendering.
 * If it is not server-side rendering, clear the nodes in the container.
 */
function legacyCreateRootFromDOMContainer(
  container: Container,
  forceHydrate: boolean
): RootType {
  const shouldHydrate =
    forceHydrate || shouldHydrateDueToLegacyHeuristic(container);
  // First clear any existing content.
  if (!shouldHydrate) {
    let warned = false;
    let rootSibling;
    while ((rootSibling = container.lastChild)) {
      if (__DEV__) {
        if (
          !warned &&
          rootSibling.nodeType === ELEMENT_NODE &&
          (rootSibling: any).hasAttribute(ROOT_ATTRIBUTE_NAME)
        ) {
          warned = true;
          console.error(
            "render(): Target node has markup rendered by React, but there " +
              "are unrelated nodes as well. This is most commonly caused by " +
              "white-space inserted around server-rendered markup."
          );
        }
      }
      container.removeChild(rootSibling);
    }
  }
  if (__DEV__) {
    if (shouldHydrate && !forceHydrate && !warnedAboutHydrateAPI) {
      warnedAboutHydrateAPI = true;
      console.warn(
        "render(): Calling ReactDOM.render() to hydrate server-rendered markup " +
          "will stop working in React v18. Replace the ReactDOM.render() call " +
          "with ReactDOM.hydrate() if you want React to attach to the server HTML."
      );
    }
  }

  return createLegacyRoot(
    container,
    shouldHydrate
      ? {
          hydrate: true,
        }
      : undefined
  );
}
```

`location: v16.3.1/react-dom/src/client/ReactDOMRoot.js`

###### createLegacyRoot

```javascript
/**
 * Create LegacyRoot by instantiating the ReactDOMBlockingRoot class
 * The I container created through the render method is LegacyRoot
 */
export function createLegacyRoot(
  container: Container,
  options?: RootOptions
): RootType {
  return new ReactDOMBlockingRoot(container, LegacyRoot, options);
}
```

###### ReactDOMBlockingRoot

```javascript
/**
 * Create LegacyRoot Fiber data structure
 */
function ReactDOMBlockingRoot(
  container: Container,
  tag: RootTag,
  options: void | RootOptions
) {
  // tag => 0 => legacyRoot
  // container => <div id="root"></div>
  // container._reactRootContainer = {_internalRoot: {}}
  this._internalRoot = createRootImpl(container, tag, options);
}
```

`location: v16.13.1/react-reconciler/src/ReactFiberReconciler.js`

###### createContainer

```javascript
export function createContainer(
  containerInfo: Container,
  tag: RootTag,
  hydrate: boolean,
  hydrationCallbacks: null | SuspenseHydrationCallbacks
): OpaqueRoot {
  return createFiberRoot(containerInfo, tag, hydrate, hydrationCallbacks);
}
```

`location: v16.13.1/react-reconciler/src/ReactFiberRoot.js`

###### createFiberRoot

```javascript
export function createFiberRoot(
  containerInfo: any,
  tag: RootTag,
  hydrate: boolean,
  hydrationCallbacks: null | SuspenseHydrationCallbacks
): FiberRoot {
  // Create fiberRoot
  const root: FiberRoot = (new FiberRootNode(containerInfo, tag, hydrate): any);
  if (enableSuspenseCallback) {
    root.hydrationCallbacks = hydrationCallbacks;
  }

  // Cyclic construction. This cheats the type system right now because
  // stateNode is any.

  // Create rootFiber
  const uninitializedFiber = createHostRootFiber(tag);

  // Connection fiberRoot and rootFiber
  root.current = uninitializedFiber;
  uninitializedFiber.stateNode = root;

  // Initialize update queue
  // Used to record changes in component state
  initializeUpdateQueue(uninitializedFiber);

  return root;
}
```

##### Create & Schedule Update Tasks

###### updateContainer

`location: v16.13.1/react-reconciler/src/ReactFiberReconciler.js`

```javascript
/**
 * Create an update task and put it in the update queue
 * Schedule the task and wait for the browser to execute it during idle time
 * @param {*} element ReactElement to render
 * @param {*} container fiberRoot
 * @param {*} parentComponent Parent Component, Initial render is null
 * @param {*} callback Callback executed after rendering is complete
 * @returns
 */
export function updateContainer(
  element: ReactNodeList,
  container: OpaqueRoot,
  parentComponent: ?React$Component<any, any>,
  callback: ?Function
): Lane {
  if (__DEV__) {
    onScheduleRoot(container, element);
  }
  const current = container.current;
  const eventTime = requestEventTime();
  if (__DEV__) {
    // $FlowExpectedError - jest isn't a global, and isn't recognized outside of tests
    if ("undefined" !== typeof jest) {
      warnIfUnmockedScheduler(current);
      warnIfNotScopedWithMatchingAct(current);
    }
  }
  const suspenseConfig = requestCurrentSuspenseConfig();
  const lane = requestUpdateLane(current, suspenseConfig);

  // fiberRoot.context = {}
  const context = getContextForSubtree(parentComponent);
  if (container.context === null) {
    container.context = context;
  } else {
    container.pendingContext = context;
  }

  if (__DEV__) {
    if (
      ReactCurrentFiberIsRendering &&
      ReactCurrentFiberCurrent !== null &&
      !didWarnAboutNestedUpdates
    ) {
      didWarnAboutNestedUpdates = true;
      console.error(
        "Render methods should be a pure function of props and state; " +
          "triggering nested component updates from render is not allowed. " +
          "If necessary, trigger nested updates in componentDidUpdate.\n\n" +
          "Check the render method of %s.",
        getComponentName(ReactCurrentFiberCurrent.type) || "Unknown"
      );
    }
  }

  // Create a to-be-executed task
  const update = createUpdate(eventTime, lane, suspenseConfig);
  // Caution: React DevTools currently depends on this property
  // being called "element".
  // 第一次渲染时，element作为update的payload
  update.payload = { element };
  callback = callback === undefined ? null : callback;
  if (callback !== null) {
    if (__DEV__) {
      if (typeof callback !== "function") {
        console.error(
          "render(...): Expected the last optional `callback` argument to be a " +
            "function. Instead received: %s.",
          callback
        );
      }
    }
    update.callback = callback;
  }

  // Put update into fiberRoot's update queue
  enqueueUpdate(current, update);

  // Schedule the current obj
  scheduleUpdateOnFiber(current, lane, eventTime);

  return lane;
}
```

###### enqueueUpdate

`location: v16.13.1/react-reconciler/src/ReactUpdateQueue.js`

```javascript
/**
 * Store the task (Update) in the task queue (updateQueue)
 * Create a one-way linked list structure to store update, next is used to concatenate update
 */
export function enqueueUpdate<State>(fiber: Fiber, update: Update<State>) {
  // Get updateQueue in rootFiber
  const updateQueue = fiber.updateQueue;
  if (updateQueue === null) {
    // Only occurs if the fiber has been unmounted.
    return;
  }

  // Get the Update task to be executed
  // Initial rendering has no pending tasks
  const sharedQueue: SharedQueue<State> = (updateQueue: any).shared;
  const pending = sharedQueue.pending;
  if (pending === null) {
    // This is the first update. Create a circular list.
    // 创建单向环状链表
    update.next = update;
  } else {
    update.next = pending.next;
    pending.next = update;
  }
  // 更新队列的链表总是指向最后一个update
  sharedQueue.pending = update;

  if (__DEV__) {
    if (
      currentlyProcessingQueue === sharedQueue &&
      !didWarnUpdateInsideUpdate
    ) {
      console.error(
        "An update (setState, replaceState, or forceUpdate) was scheduled " +
          "from inside an update function. Update functions should be pure, " +
          "with zero side-effects. Consider using componentDidUpdate or a " +
          "callback."
      );
      didWarnUpdateInsideUpdate = true;
    }
  }
}
```

###### scheduleUpdateOnFiber

`location: v16.13.1/react-reconciler/src/ReactFiberWorkLoop.js`

```javascript
export function scheduleUpdateOnFiber(
  fiber: Fiber,
  lane: Lane,
  eventTime: number
) {
  console.log("本轮渲染的渲染优先级：", lane);
  // 判断是否是无限循环的update
  checkForNestedUpdates();
  warnAboutRenderPhaseUpdatesInDEV(fiber);
  // 找到rootFiber并遍历更新子节点的优先级
  const root = markUpdateLaneFromFiberToRoot(fiber, lane);
  console.log(root);
  console.log("root.pendingLanes：", root.pendingLanes);
  if (root === null) {
    warnAboutUpdateOnUnmountedFiberInDEV(fiber);
    return null;
  }

  // TODO: requestUpdateLanePriority also reads the priority. Pass the
  // priority as an argument to that function and this one.
  const priorityLevel = getCurrentPriorityLevel();
  // 获取到当前的优先级
  if (lane === SyncLane) {
    if (
      // Check if we're inside unbatchedUpdates
      (executionContext & LegacyUnbatchedContext) !== NoContext &&
      // Check if we're not already rendering
      (executionContext & (RenderContext | CommitContext)) === NoContext
    ) {
      // Register pending interactions on the root to avoid losing traced interaction data.
      // 没处在渲染过程中，没有进行批量更新，在root
      // 上注册那些待处理的交互，来避免丢失已跟踪的
      // 交互数据
      schedulePendingInteractions(root, lane);
      // This is a legacy edge case. The initial mount of a ReactDOM.render-ed
      // root inside of batchedUpdates should be synchronous, but layout updates
      // should be deferred until the end of the batch.
      // 初始挂载，ReactDOM渲染的root节点的批量更新
      // 应该是同步的，但布局更新应该延迟到批处理结束。
      performSyncWorkOnRoot(root);
    } else {
      // 更新组件时，需要确保root组件已经被调度。
      // 如被已经被调度了，则检查它的优先级有没有变化
      // 有变化，就取消上一个调度，重新安排一个调度任务
      // 没变化则直接退出
      ensureRootIsScheduled(root, eventTime);
      schedulePendingInteractions(root, lane);
      if (executionContext === NoContext) {
        // Flush the synchronous work now, unless we're already working or inside
        // a batch. This is intentionally inside scheduleUpdateOnFiber instead of
        // scheduleCallbackForFiber to preserve the ability to schedule a callback
        // without immediately flushing it. We only do this for user-initiated
        // updates, to preserve historical behavior of legacy mode.
        /**
         * 只有在执行上下文为空的时候，才去刷新同步回调队列，这样可以保证刚才调度的回调不会立即执行
         * */
        // 刷新同步回调队列
        flushSyncCallbackQueue();
      }
    }
  } else {
    // Schedule a discrete update but only if it's not Sync.
    // 调度一个离散的更新，只是因为它不是同步的
    if (
      (executionContext & DiscreteEventContext) !== NoContext &&
      // Only updates at user-blocking priority or greater are considered
      // discrete, even inside a discrete event.
      // 只有用户阻塞优先级或更高的更新被认为是离散的，即使在离散事件中也是如此。
      (priorityLevel === UserBlockingSchedulerPriority ||
        priorityLevel === ImmediateSchedulerPriority)
    ) {
      // This is the result of a discrete event. Track the lowest priority
      // discrete update per root so we can flush them early, if needed.
      if (rootsWithPendingDiscreteUpdates === null) {
        rootsWithPendingDiscreteUpdates = new Set([root]);
      } else {
        rootsWithPendingDiscreteUpdates.add(root);
      }
    }
    // Schedule other updates after in case the callback is sync.
    ensureRootIsScheduled(root, eventTime);
    schedulePendingInteractions(root, lane);
  }

  // We use this when assigning a lane for a transition inside
  // `requestUpdateLane`. We assume it's the same as the root being updated,
  // since in the common case of a single root app it probably is. If it's not
  // the same root, then it's not a huge deal, we just might batch more stuff
  // together more than necessary.
  mostRecentlyUpdatedRoot = root;
}
```

##### Build workInProgress Fiber tree(rootFiber)

###### performSyncWorkOnRoot

`location: v16.13.1/react-reconciler/src/ReactFiberWorkLoop.js`

```javascript
// This is the entry point for synchronous tasks that don't go through Scheduler
// 这是不经过Scheduler的同步任务的入口点
function performSyncWorkOnRoot(root) {
  invariant(
    (executionContext & (RenderContext | CommitContext)) === NoContext,
    "Should not already be working."
  );
  // 刷新被动的副作用
  flushPassiveEffects();
  let lanes;
  let exitStatus; // 声明一个状态，来保存render阶段完成时的状态，对某些错误做特定处理
  if (
    root === workInProgressRoot &&
    includesSomeLane(root.expiredLanes, workInProgressRootRenderLanes)
  ) {
    // There's a partial tree, and at least one of its lanes has expired. Finish
    // rendering it before rendering the rest of the expired work.
    // 这是一个不完整的tree，至少有一个lanes已经过期了，在渲染剩余的过期任务之前完成渲染
    lanes = workInProgressRootRenderLanes;
    exitStatus = renderRootSync(root, lanes);
    if (
      includesSomeLane(
        workInProgressRootIncludedLanes,
        workInProgressRootUpdatedLanes
      )
    ) {
      // The render included lanes that were updated during the render phase.
      // For example, when unhiding a hidden tree, we include all the lanes
      // that were previously skipped when the tree was hidden. That set of
      // lanes is a superset of the lanes we started rendering with.
      // 本次渲染包含渲染阶段被更新的lanes，例如，当显示一个隐藏的树的时候，将会包含
      // 这棵树当时被隐藏时跳过的所有lanes。这组lanes是开始渲染时的lanes的超集。
      //
      // Note that this only happens when part of the tree is rendered
      // concurrently. If the whole tree is rendered synchronously, then there
      // are no interleaved events.
      // 只有当树的一部分是并发渲染的时候才会发生这种情况，如果整棵树是同步渲染的，
      // 将不会发生交叉事件
      lanes = getNextLanes(root, lanes);
      exitStatus = renderRootSync(root, lanes);
    }
  } else {
    lanes = getNextLanes(root, NoLanes);
    exitStatus = renderRootSync(root, lanes);
  }
  // 有错误的情况
  if (root.tag !== LegacyRoot && exitStatus === RootErrored) {
    executionContext |= RetryAfterError;

    // If an error occurred during hydration,
    // discard server response and fall back to client side render.
    // 一旦hydrate过程中出现了错误，放弃服务端返回的内容并启用客户端渲染。
    // 这是服务端渲染相关的逻辑，ReactDOM.hydrate会复用服务端发来的HTML结构。
    if (root.hydrate) {
      root.hydrate = false;
      clearContainer(root.containerInfo);
    }

    // If something threw an error, try rendering one more time. We'll render
    // synchronously to block concurrent data mutations, and we'll includes
    // all pending updates are included. If it still fails after the second
    // attempt, we'll give up and commit the resulting tree.
    // 如果有错误，再重新渲染一次，使用同步渲染来阻止并发的数据变化。这将包括所有
    // 等待处理的更新，如果在第二次尝试后仍然失败，那么直接放弃，然后提交当前已
    // 经渲染的tree。
    lanes = getLanesToRetrySynchronouslyOnError(root);
    if (lanes !== NoLanes) {
      exitStatus = renderRootSync(root, lanes);
    }
  }

  if (exitStatus === RootFatalErrored) {
    const fatalError = workInProgressRootFatalError;
    prepareFreshStack(root, NoLanes);
    markRootSuspended(root, lanes);
    ensureRootIsScheduled(root, now());
    throw fatalError;
  }

  // We now have a consistent tree. Because this is a sync render, we
  // will commit it even if something suspended.
  const finishedWork: Fiber = (root.current.alternate: any);
  root.finishedWork = finishedWork;
  root.finishedLanes = lanes;
  // 进入commit阶段
  commitRoot(root);

  // Before exiting, make sure there's a callback scheduled for the next
  // pending level.
  // 再次对fiberRoot进行调度(退出之前保证fiberRoot没有需要调度的任务)
  ensureRootIsScheduled(root, now());
  return null;
}
```

###### renderRootSync

`location: v16.13.1/react-reconciler/src/ReactFiberWorkLoop.js`

```javascript
function renderRootSync(root: FiberRoot, lanes: Lanes) {
  const prevExecutionContext = executionContext;
  executionContext |= RenderContext;
  const prevDispatcher = pushDispatcher(root);

  // If the root or lanes have changed, throw out the existing stack
  // and prepare a fresh one. Otherwise we'll continue where we left off.
  if (workInProgressRoot !== root || workInProgressRootRenderLanes !== lanes) {
    // Build the workInProgress Fiber tree and its rootFiber object
    prepareFreshStack(root, lanes);
    startWorkOnPendingInteractions(root, lanes);
  }

  const prevInteractions = pushInteractions(root);

  do {
    try {
      workLoopSync();
      break;
    } catch (thrownValue) {
      // 渲染报错
      handleError(root, thrownValue);
    }
  } while (true);
  resetContextDependencies();
  if (enableSchedulerTracing) {
    popInteractions(((prevInteractions: any): Set<Interaction>));
  }

  executionContext = prevExecutionContext;
  popDispatcher(prevDispatcher);

  if (workInProgress !== null) {
    // This is a sync render, so we should have finished the whole tree.
    invariant(
      false,
      "Cannot commit an incomplete root. This error is likely caused by a " +
        "bug in React. Please file an issue."
    );
  }

  // Set this to null to indicate there's no in-progress render.
  workInProgressRoot = null;
  workInProgressRootRenderLanes = NoLanes;

  return workInProgressRootExitStatus;
}
```

###### prepareFreshStack

`location: v16.13.1/react-reconciler/src/ReactFiberWorkLoop.js`

```javascript
function prepareFreshStack(root: FiberRoot, lanes: Lanes) {
  // finishedWork represents the Fiber object to be submitted that is built after the render phase is completed.
  root.finishedWork = null;
  root.finishedLanes = NoLanes;

  const timeoutHandle = root.timeoutHandle;
  if (timeoutHandle !== noTimeout) {
    // The root previous suspended and scheduled a timeout to commit a fallback
    // state. Now that we have additional work, cancel the timeout.
    root.timeoutHandle = noTimeout;
    // $FlowFixMe Complains noTimeout is not a TimeoutID, despite the check above
    cancelTimeout(timeoutHandle);
  }

  if (workInProgress !== null) {
    let interruptedWork = workInProgress.return;
    while (interruptedWork !== null) {
      unwindInterruptedWork(interruptedWork);
      interruptedWork = interruptedWork.return;
    }
  }

  // Construct the fiberRoot object of the workInProgress Fiber tree
  workInProgressRoot = root;
  // Build the rootFiber in the workInProgress Fiber tree
  workInProgress = createWorkInProgress(root.current, null);
  workInProgressRootRenderLanes =
    subtreeRenderLanes =
    workInProgressRootIncludedLanes =
      lanes;
  workInProgressRootExitStatus = RootIncomplete;
  workInProgressRootFatalError = null;
  workInProgressRootLatestProcessedEventTime = NoTimestamp;
  workInProgressRootLatestSuspenseTimeout = NoTimestamp;
  workInProgressRootCanSuspendUsingConfig = null;
  workInProgressRootSkippedLanes = NoLanes;
  workInProgressRootUpdatedLanes = NoLanes;
  workInProgressRootPingedLanes = NoLanes;

  if (enableSchedulerTracing) {
    spawnedWorkDuringRender = null;
  }

  if (__DEV__) {
    ReactStrictModeWarnings.discardPendingWarnings();
  }
}
```

###### createWorkInProgress

`location: v16.13.1/react-reconciler/src/ReactFiber.js`

```javascript
// This is used to create an alternate fiber to do work on.
export function createWorkInProgress(current: Fiber, pendingProps: any): Fiber {
  let workInProgress = current.alternate;
  if (workInProgress === null) {
    // We use a double buffering pooling technique because we know that we'll
    // only ever need at most two versions of a tree. We pool the "other" unused
    // node that we're free to reuse. This is lazily created to avoid allocating
    // extra objects for things that are never updated. It also allow us to
    // reclaim the extra memory if needed.
    workInProgress = createFiber(
      current.tag,
      pendingProps,
      current.key,
      current.mode
    );
    workInProgress.elementType = current.elementType;
    workInProgress.type = current.type;
    workInProgress.stateNode = current.stateNode;

    if (__DEV__) {
      // DEV-only fields
      workInProgress._debugID = current._debugID;
      workInProgress._debugSource = current._debugSource;
      workInProgress._debugOwner = current._debugOwner;
      workInProgress._debugHookTypes = current._debugHookTypes;
    }

    workInProgress.alternate = current;
    current.alternate = workInProgress;
  } else {
    workInProgress.pendingProps = pendingProps;
    // Needed because Blocks store data on type.
    workInProgress.type = current.type;

    // We already have an alternate.
    // Reset the effect tag.
    workInProgress.effectTag = NoEffect;

    // The effect list is no longer valid.
    workInProgress.nextEffect = null;
    workInProgress.firstEffect = null;
    workInProgress.lastEffect = null;

    if (enableProfilerTimer) {
      // We intentionally reset, rather than copy, actualDuration & actualStartTime.
      // This prevents time from endlessly accumulating in new commits.
      // This has the downside of resetting values for different priority renders,
      // But works for yielding (the common case) and should support resuming.
      workInProgress.actualDuration = 0;
      workInProgress.actualStartTime = -1;
    }
  }

  workInProgress.childLanes = current.childLanes;
  workInProgress.lanes = current.lanes;

  workInProgress.child = current.child;
  workInProgress.memoizedProps = current.memoizedProps;
  workInProgress.memoizedState = current.memoizedState;
  workInProgress.updateQueue = current.updateQueue;

  // Clone the dependencies object. This is mutated during the render phase, so
  // it cannot be shared with the current fiber.
  const currentDependencies = current.dependencies;
  workInProgress.dependencies =
    currentDependencies === null
      ? null
      : {
          lanes: currentDependencies.lanes,
          firstContext: currentDependencies.firstContext,
          responders: currentDependencies.responders,
        };

  // These will be overridden during the parent's reconciliation
  workInProgress.sibling = current.sibling;
  workInProgress.index = current.index;
  workInProgress.ref = current.ref;

  if (enableProfilerTimer) {
    workInProgress.selfBaseDuration = current.selfBaseDuration;
    workInProgress.treeBaseDuration = current.treeBaseDuration;
  }

  if (__DEV__) {
    workInProgress._debugNeedsRemount = current._debugNeedsRemount;
    switch (workInProgress.tag) {
      case IndeterminateComponent:
      case FunctionComponent:
      case SimpleMemoComponent:
        workInProgress.type = resolveFunctionForHotReloading(current.type);
        break;
      case ClassComponent:
        workInProgress.type = resolveClassForHotReloading(current.type);
        break;
      case ForwardRef:
        workInProgress.type = resolveForwardRefForHotReloading(current.type);
        break;
      default:
        break;
    }
  }

  return workInProgress;
}
```

##### Construct child Fiber objects

###### workLoopSync

`location: v16.13.1/react-reconciler/src/ReactFiberWorkLoop.js`

```javascript
// The work loop is an extremely hot path. Tell Closure not to inline it.
/** @noinline */
function workLoopSync() {
  // Already timed out, so perform work without checking if we need to yield.
  // 已经超时了，所以不用检查是否需要让出执行权
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}
```

###### performUnitOfWork

`location: v16.13.1/react-reconciler/src/ReactFiberWorkLoop.js`

```javascript
function performUnitOfWork(unitOfWork: Fiber): void {
  // The current, flushed, state of this fiber is the alternate. Ideally
  // nothing should rely on this, but relying on it here means that we don't
  // need an additional field on the work in progress.
  const current = unitOfWork.alternate;
  setCurrentDebugFiberInDEV(unitOfWork);

  let next;
  // beginWork阶段 从父到子 构建Fiber对象
  if (enableProfilerTimer && (unitOfWork.mode & ProfileMode) !== NoMode) {
    startProfilerTimer(unitOfWork);
    next = beginWork(current, unitOfWork, subtreeRenderLanes);
    stopProfilerTimerIfRunningAndRecordDelta(unitOfWork, true);
  } else {
    next = beginWork(current, unitOfWork, subtreeRenderLanes);
  }

  resetCurrentDebugFiberInDEV();
  unitOfWork.memoizedProps = unitOfWork.pendingProps;

  // If the child node does not exist, it means that the current node has traversed down the child node to the end.
  // Continue to return upward, encounter sibling nodes, and construct child Fiber objects of sibling nodes until returning to the root Fiber object.
  if (next === null) {
    // If this doesn't spawn new work, complete the current work.
    // complete阶段 从子到父 构建其余的子级Fiber对象
    completeUnitOfWork(unitOfWork);
  } else {
    workInProgress = next;
  }

  ReactCurrentOwner.current = null;
}
```

###### beginWork

`location: v16.13.1/react-reconciler/src/ReactFiberBeginWork.js`

```javascript
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes
): Fiber | null {
  /*
   * current：当前处理的Fiber节点的上一次更新时的Fiber，就是workInProgress.alternate。
   * workInProgress： 当前处理的Fiber节点
   *
   * 基于双缓冲的概念，current是当前展示出来给用户看到的对应的结构Fiber，
   * workInProgress是正在后台构建的Fiber。由于本次更新完毕之后，workInProgress会
   * 成为current，所以在下次更新时，current就是新构建的workInProgress上次更新时对
   * 应的Fiber
   *
   * */
  const updateLanes = workInProgress.lanes;

  if (__DEV__) {
    if (workInProgress._debugNeedsRemount && current !== null) {
      // This will restart the begin phase with a new fiber.
      return remountFiber(
        current,
        workInProgress,
        createFiberFromTypeAndProps(
          workInProgress.type,
          workInProgress.key,
          workInProgress.pendingProps,
          workInProgress._debugOwner || null,
          workInProgress.mode,
          workInProgress.lanes
        )
      );
    }
  }
  /*
   * 在第一次渲染时，root是第一次更新，所以它没有上一次更新的Fiber节点，但更新时就不一样了。
   * 表现在workInProgress树的构建过程中就是，第一次渲染时，除了rootFiber，其余的子节点要根据
   * fiber.tag创建出来，更新时，如果fiber上无需更新，那么可以直接复用已有节点。
   * */
  if (current !== null) {
    const oldProps = current.memoizedProps;
    const newProps = workInProgress.pendingProps;

    if (
      oldProps !== newProps ||
      hasLegacyContextChanged() ||
      // Force a re-render if the implementation changed due to hot reload:
      (__DEV__ ? workInProgress.type !== current.type : false)
    ) {
      // 如果props或者context发生了变化，标记一下说明需要更新
      // If props or context changed, mark the fiber as having performed work.
      // This may be unset if the props are determined to be equal later (memo).
      didReceiveUpdate = true;
    } else if (!includesSomeLane(renderLanes, updateLanes)) {
      didReceiveUpdate = false;
      // This fiber does not have any pending work. Bailout without entering
      // the begin phase. There's still some bookkeeping we that needs to be done
      // in this optimized path, mostly pushing stuff onto the stack.
      switch (workInProgress.tag) {
        case HostRoot:
          pushHostRootContext(workInProgress);
          resetHydrationState();
          break;
        case HostComponent:
          pushHostContext(workInProgress);
          break;
        case ClassComponent: {
          const Component = workInProgress.type;
          if (isLegacyContextProvider(Component)) {
            pushLegacyContextProvider(workInProgress);
          }
          break;
        }
        case HostPortal:
          pushHostContainer(
            workInProgress,
            workInProgress.stateNode.containerInfo
          );
          break;
        case ContextProvider: {
          const newValue = workInProgress.memoizedProps.value;
          pushProvider(workInProgress, newValue);
          break;
        }
        case Profiler:
          if (enableProfilerTimer) {
            // Profiler should only call onRender when one of its descendants actually rendered.
            const hasChildWork = includesSomeLane(
              renderLanes,
              workInProgress.childLanes
            );
            if (hasChildWork) {
              workInProgress.effectTag |= Update;
            }

            // Reset effect durations for the next eventual effect phase.
            // These are reset during render to allow the DevTools commit hook a chance to read them,
            const stateNode = workInProgress.stateNode;
            stateNode.effectDuration = 0;
            stateNode.passiveEffectDuration = 0;
          }
          break;
        case SuspenseComponent: {
          const state: SuspenseState | null = workInProgress.memoizedState;
          if (state !== null) {
            if (enableSuspenseServerRenderer) {
              if (state.dehydrated !== null) {
                pushSuspenseContext(
                  workInProgress,
                  setDefaultShallowSuspenseContext(suspenseStackCursor.current)
                );
                // We know that this component will suspend again because if it has
                // been unsuspended it has committed as a resolved Suspense component.
                // If it needs to be retried, it should have work scheduled on it.
                workInProgress.effectTag |= DidCapture;
                // We should never render the children of a dehydrated boundary until we
                // upgrade it. We return null instead of bailoutOnAlreadyFinishedWork.
                return null;
              }
            }

            // If this boundary is currently timed out, we need to decide
            // whether to retry the primary children, or to skip over it and
            // go straight to the fallback. Check the priority of the primary
            // child fragment.
            const primaryChildFragment: Fiber = (workInProgress.child: any);
            const primaryChildLanes = primaryChildFragment.childLanes;
            if (includesSomeLane(renderLanes, primaryChildLanes)) {
              // The primary children have pending work. Use the normal path
              // to attempt to render the primary children again.
              return updateSuspenseComponent(
                current,
                workInProgress,
                renderLanes
              );
            } else {
              // The primary child fragment does not have pending work marked
              // on it
              pushSuspenseContext(
                workInProgress,
                setDefaultShallowSuspenseContext(suspenseStackCursor.current)
              );
              // The primary children do not have pending work with sufficient
              // priority. Bailout.
              const child = bailoutOnAlreadyFinishedWork(
                current,
                workInProgress,
                renderLanes
              );
              if (child !== null) {
                // The fallback children have pending work. Skip over the
                // primary children and work on the fallback.
                return child.sibling;
              } else {
                return null;
              }
            }
          } else {
            pushSuspenseContext(
              workInProgress,
              setDefaultShallowSuspenseContext(suspenseStackCursor.current)
            );
          }
          break;
        }
        case SuspenseListComponent: {
          const didSuspendBefore =
            (current.effectTag & DidCapture) !== NoEffect;

          const hasChildWork = includesSomeLane(
            renderLanes,
            workInProgress.childLanes
          );

          if (didSuspendBefore) {
            if (hasChildWork) {
              // If something was in fallback state last time, and we have all the
              // same children then we're still in progressive loading state.
              // Something might get unblocked by state updates or retries in the
              // tree which will affect the tail. So we need to use the normal
              // path to compute the correct tail.
              /*
               * 如果SuspenseList中的某个Suspense加载完成，这意味着需要更新SuspenseList。
               * 主要是依据revealOrder和tail决定子Suspense的显示效果。
               * */
              return updateSuspenseListComponent(
                current,
                workInProgress,
                renderLanes
              );
            }
            // If none of the children had any work, that means that none of
            // them got retried so they'll still be blocked in the same way
            // as before. We can fast bail out.
            workInProgress.effectTag |= DidCapture;
          }

          // If nothing suspended before and we're rendering the same children,
          // then the tail doesn't matter. Anything new that suspends will work
          // in the "together" mode, so we can continue from the state we had.
          const renderState = workInProgress.memoizedState;
          if (renderState !== null) {
            // Reset to the "together" mode in case we've started a different
            // update in the past but didn't complete it.
            renderState.rendering = null;
            renderState.tail = null;
            renderState.lastEffect = null;
          }
          pushSuspenseContext(workInProgress, suspenseStackCursor.current);

          if (hasChildWork) {
            break;
          } else {
            // If none of the children had any work, that means that none of
            // them got retried so they'll still be blocked in the same way
            // as before. We can fast bail out.
            return null;
          }
        }
        case OffscreenComponent:
        case LegacyHiddenComponent: {
          // Need to check if the tree still needs to be deferred. This is
          // almost identical to the logic used in the normal update path,
          // so we'll just enter that. The only difference is we'll bail out
          // at the next level instead of this one, because the child props
          // have not changed. Which is fine.
          // TODO: Probably should refactor `beginWork` to split the bailout
          // path from the normal path. I'm tempted to do a labeled break here
          // but I won't :)
          workInProgress.lanes = NoLanes;
          return updateOffscreenComponent(current, workInProgress, renderLanes);
        }
      }
      // bailoutOnAlreadyFinishedWork 跳过已完成的fiber任务
      return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
    } else {
      if ((current.effectTag & ForceUpdateForLegacySuspense) !== NoEffect) {
        // This is a special case that only exists for legacy mode.
        // See https://github.com/facebook/react/pull/19216.
        didReceiveUpdate = true;
      } else {
        // An update was scheduled on this fiber, but there are no new props
        // nor legacy context. Set this to false. If an update queue or context
        // consumer produces a changed value, it will set this to true. Otherwise,
        // the component will assume the children have not changed and bail out.
        /*
         * 当前这个fiber计划进行更新，但是没有新的props或者context，所以didReceiveUpdate设置为false。
         * 如果更新队列或者context 的consumer产生了一个改变的值，它将把它设置为true。否则，组件将假定
         * 子组件没有更改并退出。
         * */
        didReceiveUpdate = false;
      }
    }
  } else {
    didReceiveUpdate = false;
  }

  // Before entering the begin phase, clear pending update priority.
  // TODO: This assumes that we're about to evaluate the component and process
  // the update queue. However, there's an exception: SimpleMemoComponent
  // sometimes bails out later in the begin phase. This indicates that we should
  // move this assignment out of the common path and into each branch.
  workInProgress.lanes = NoLanes;

  switch (workInProgress.tag) {
    case IndeterminateComponent: {
      return mountIndeterminateComponent(
        current,
        workInProgress,
        workInProgress.type,
        renderLanes
      );
    }
    case LazyComponent: {
      const elementType = workInProgress.elementType;
      return mountLazyComponent(
        current,
        workInProgress,
        elementType,
        updateLanes,
        renderLanes
      );
    }
    case FunctionComponent: {
      const Component = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      const resolvedProps =
        workInProgress.elementType === Component
          ? unresolvedProps
          : resolveDefaultProps(Component, unresolvedProps);
      return updateFunctionComponent(
        current,
        workInProgress,
        Component,
        resolvedProps,
        renderLanes
      );
    }
    case ClassComponent: {
      const Component = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      const resolvedProps =
        workInProgress.elementType === Component
          ? unresolvedProps
          : resolveDefaultProps(Component, unresolvedProps);
      return updateClassComponent(
        current,
        workInProgress,
        Component,
        resolvedProps,
        renderLanes
      );
    }
    case HostRoot:
      return updateHostRoot(current, workInProgress, renderLanes);
    case HostComponent:
      return updateHostComponent(current, workInProgress, renderLanes);
    case HostText:
      return updateHostText(current, workInProgress);
    case SuspenseComponent:
      return updateSuspenseComponent(current, workInProgress, renderLanes);
    case HostPortal:
      return updatePortalComponent(current, workInProgress, renderLanes);
    case ForwardRef: {
      const type = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      const resolvedProps =
        workInProgress.elementType === type
          ? unresolvedProps
          : resolveDefaultProps(type, unresolvedProps);
      return updateForwardRef(
        current,
        workInProgress,
        type,
        resolvedProps,
        renderLanes
      );
    }
    case Fragment:
      return updateFragment(current, workInProgress, renderLanes);
    case Mode:
      return updateMode(current, workInProgress, renderLanes);
    case Profiler:
      return updateProfiler(current, workInProgress, renderLanes);
    case ContextProvider:
      return updateContextProvider(current, workInProgress, renderLanes);
    case ContextConsumer:
      return updateContextConsumer(current, workInProgress, renderLanes);
    case MemoComponent: {
      const type = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      // Resolve outer props first, then resolve inner props.
      let resolvedProps = resolveDefaultProps(type, unresolvedProps);
      if (__DEV__) {
        if (workInProgress.type !== workInProgress.elementType) {
          const outerPropTypes = type.propTypes;
          if (outerPropTypes) {
            checkPropTypes(
              outerPropTypes,
              resolvedProps, // Resolved for outer only
              "prop",
              getComponentName(type)
            );
          }
        }
      }
      resolvedProps = resolveDefaultProps(type.type, resolvedProps);
      return updateMemoComponent(
        current,
        workInProgress,
        type,
        resolvedProps,
        updateLanes,
        renderLanes
      );
    }
    case SimpleMemoComponent: {
      return updateSimpleMemoComponent(
        current,
        workInProgress,
        workInProgress.type,
        workInProgress.pendingProps,
        updateLanes,
        renderLanes
      );
    }
    case IncompleteClassComponent: {
      const Component = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      const resolvedProps =
        workInProgress.elementType === Component
          ? unresolvedProps
          : resolveDefaultProps(Component, unresolvedProps);
      return mountIncompleteClassComponent(
        current,
        workInProgress,
        Component,
        resolvedProps,
        renderLanes
      );
    }
    case SuspenseListComponent: {
      return updateSuspenseListComponent(current, workInProgress, renderLanes);
    }
    case FundamentalComponent: {
      if (enableFundamentalAPI) {
        return updateFundamentalComponent(current, workInProgress, renderLanes);
      }
      break;
    }
    case ScopeComponent: {
      if (enableScopeAPI) {
        return updateScopeComponent(current, workInProgress, renderLanes);
      }
      break;
    }
    case Block: {
      if (enableBlocksAPI) {
        const block = workInProgress.type;
        const props = workInProgress.pendingProps;
        return updateBlock(current, workInProgress, block, props, renderLanes);
      }
      break;
    }
    case OffscreenComponent: {
      return updateOffscreenComponent(current, workInProgress, renderLanes);
    }
    case LegacyHiddenComponent: {
      return updateLegacyHiddenComponent(current, workInProgress, renderLanes);
    }
  }
  invariant(
    false,
    "Unknown unit of work tag (%s). This error is likely caused by a bug in " +
      "React. Please file an issue.",
    workInProgress.tag
  );
}
```

###### updateHostRoot

`location: v16.13.1/react-reconciler/src/ReactFiberBeginWork.js`

```javascript
function updateHostRoot(current, workInProgress, renderLanes) {
  pushHostRootContext(workInProgress);
  const updateQueue = workInProgress.updateQueue;
  invariant(
    current !== null && updateQueue !== null,
    "If the root does not have an updateQueue, we should have already " +
      "bailed out. This error is likely caused by a bug in React. Please " +
      "file an issue."
  );
  const nextProps = workInProgress.pendingProps;
  const prevState = workInProgress.memoizedState;
  const prevChildren = prevState !== null ? prevState.element : null;

  // Avoid confusion with reference properties
  cloneUpdateQueue(current, workInProgress);

  // workInProgress.memoizedState = updateQueue.payload
  processUpdateQueue(workInProgress, nextProps, null, renderLanes);
  const nextState = workInProgress.memoizedState;
  // Caution: React DevTools currently depends on this property
  // being called "element".
  const nextChildren = nextState.element;
  if (nextChildren === prevChildren) {
    resetHydrationState();
    return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
  }
  const root: FiberRoot = workInProgress.stateNode;
  if (root.hydrate && enterHydrationState(workInProgress)) {
    // If we don't have any current children this might be the first pass.
    // We always try to hydrate. If this isn't a hydration pass there won't
    // be any children to hydrate which is effectively the same thing as
    // not hydrating.

    if (supportsHydration) {
      const mutableSourceEagerHydrationData =
        root.mutableSourceEagerHydrationData;
      if (mutableSourceEagerHydrationData != null) {
        for (let i = 0; i < mutableSourceEagerHydrationData.length; i += 2) {
          const mutableSource = ((mutableSourceEagerHydrationData[
            i
          ]: any): MutableSource<any>);
          const version = mutableSourceEagerHydrationData[i + 1];
          setWorkInProgressVersion(mutableSource, version);
        }
      }
    }

    const child = mountChildFibers(
      workInProgress,
      null,
      nextChildren,
      renderLanes
    );
    workInProgress.child = child;

    let node = child;
    while (node) {
      // Mark each child as hydrating. This is a fast path to know whether this
      // tree is part of a hydrating tree. This is used to determine if a child
      // node has fully mounted yet, and for scheduling event replaying.
      // Conceptually this is similar to Placement in that a new subtree is
      // inserted into the React tree here. It just happens to not need DOM
      // mutations because it already exists.
      node.effectTag = (node.effectTag & ~Placement) | Hydrating;
      node = node.sibling;
    }
  } else {
    // Otherwise reset hydration state in case we aborted and resumed another
    // root.
    reconcileChildren(current, workInProgress, nextChildren, renderLanes);
    resetHydrationState();
  }
  return workInProgress.child;
}
```

###### reconcileChildren

`location: v16.13.1/react-reconciler/src/ReactFiberBeginWork.js`

```javascript
export function reconcileChildren(
  current: Fiber | null,
  workInProgress: Fiber,
  nextChildren: any,
  renderLanes: Lanes
) {
  if (current === null) {
    // If this is a fresh new component that hasn't been rendered yet, we
    // won't update its child set by applying minimal side-effects. Instead,
    // we will add them all to the child before it gets rendered. That means
    // we can optimize this reconciliation pass by not tracking side-effects.
    workInProgress.child = mountChildFibers(
      workInProgress,
      null,
      nextChildren,
      renderLanes
    );
  } else {
    // If the current child is the same as the work in progress, it means that
    // we haven't yet started any work on these children. Therefore, we use
    // the clone algorithm to create a copy of all the current children.

    // If we had any progressed work already, that is invalid at this point so
    // let's throw it out.
    workInProgress.child = reconcileChildFibers(
      workInProgress,
      current.child,
      nextChildren,
      renderLanes
    );
  }
}
```

###### mountChildFibers

`location: v16.13.1/react-reconciler/src/ReactChildFiber.js`

```javascript
/**
 * shouldTrackSideEffects flag, whether to add effectTag to Fiber object
 * true to add, false to not add
 * For initial rendering, only the root component needs to be added, and other elements do not need to be added to prevent excessive DOM operations.
 */
// Used to update
export const reconcileChildFibers = ChildReconciler(true);
// Used to initialize rendering
export const mountChildFibers = ChildReconciler(false);
```

###### ChildReconciler

`location: v16.13.1/react-reconciler/src/ReactChildFiber.js`

```javascript
// This wrapper function exists because I expect to clone the code in each path
// to be able to optimize each path individually by branching early. This needs
// a compiler or we can do it manually. Helpers that don't need this branching
// live outside of this function.
function ChildReconciler(shouldTrackSideEffects) {
  function reconcileSingleElement(
    returnFiber: Fiber,
    currentFirstChild: Fiber | null,
    element: ReactElement,
    lanes: Lanes
  ): Fiber {
    const key = element.key;
    let child = currentFirstChild;
    while (child !== null) {
      // TODO: If key === null and child.key === null, then this only applies to
      // the first item in the list.
      if (child.key === key) {
        switch (child.tag) {
          case Fragment: {
            if (element.type === REACT_FRAGMENT_TYPE) {
              deleteRemainingChildren(returnFiber, child.sibling);
              const existing = useFiber(child, element.props.children);
              existing.return = returnFiber;
              if (__DEV__) {
                existing._debugSource = element._source;
                existing._debugOwner = element._owner;
              }
              return existing;
            }
            break;
          }
          case Block:
            if (enableBlocksAPI) {
              let type = element.type;
              if (type.$$typeof === REACT_LAZY_TYPE) {
                type = resolveLazyType(type);
              }
              if (type.$$typeof === REACT_BLOCK_TYPE) {
                // The new Block might not be initialized yet. We need to initialize
                // it in case initializing it turns out it would match.
                if (
                  ((type: any): BlockComponent<any, any>)._render ===
                  (child.type: BlockComponent<any, any>)._render
                ) {
                  deleteRemainingChildren(returnFiber, child.sibling);
                  const existing = useFiber(child, element.props);
                  existing.type = type;
                  existing.return = returnFiber;
                  if (__DEV__) {
                    existing._debugSource = element._source;
                    existing._debugOwner = element._owner;
                  }
                  return existing;
                }
              }
            }
          // We intentionally fallthrough here if enableBlocksAPI is not on.
          // eslint-disable-next-lined no-fallthrough
          default: {
            if (
              child.elementType === element.type ||
              // Keep this check inline so it only runs on the false path:
              (__DEV__
                ? isCompatibleFamilyForHotReloading(child, element)
                : false)
            ) {
              deleteRemainingChildren(returnFiber, child.sibling);
              const existing = useFiber(child, element.props);
              existing.ref = coerceRef(returnFiber, child, element);
              existing.return = returnFiber;
              if (__DEV__) {
                existing._debugSource = element._source;
                existing._debugOwner = element._owner;
              }
              return existing;
            }
            break;
          }
        }
        // Didn't match.
        deleteRemainingChildren(returnFiber, child);
        break;
      } else {
        deleteChild(returnFiber, child);
      }
      child = child.sibling;
    }

    if (element.type === REACT_FRAGMENT_TYPE) {
      const created = createFiberFromFragment(
        element.props.children,
        returnFiber.mode,
        lanes,
        element.key
      );
      created.return = returnFiber;
      return created;
    } else {
      const created = createFiberFromElement(element, returnFiber.mode, lanes);
      created.ref = coerceRef(returnFiber, currentFirstChild, element);
      created.return = returnFiber;
      return created;
    }
  }

  function reconcileChildrenArray(
    returnFiber: Fiber,
    currentFirstChild: Fiber | null,
    newChildren: Array<*>,
    lanes: Lanes
  ): Fiber | null {
    // console.log('newChildrenIterable', newChildren);
    // This algorithm can't optimize by searching from both ends since we
    // don't have backpointers on fibers. I'm trying to see how far we can get
    // with that model. If it ends up not being worth the tradeoffs, we can
    // add it later.
    /*
     * 这个算法不能通过从两端搜索来优化，因为Fiber上没有反向指针。我想看看这个模型能走多远。
     * 如果它最终不值得权衡，我们可以稍后添加它。
     * */
    // Even with a two ended optimization, we'd want to optimize for the case
    // where there are few changes and brute force the comparison instead of
    // going for the Map. It'd like to explore hitting that path first in
    // forward-only mode and only go for the Map once we notice that we need
    // lots of look ahead. This doesn't handle reversal as well as two ended
    // search but that's unusual. Besides, for the two ended optimization to
    // work on Iterables, we'd need to copy the whole set.
    /*
     * 即使有两个端点的优化，我们也希望优化很少变化的情况，并使用蛮力进行比较，而不是使用映
     * 射。它会先在前进模式中探索路径，然后在我们注意到需要向前看时才去找地图。这不能处理反
     * 转以及两个结束的搜索，但这是不寻常的。此外，为了使这两个结束的优化工作在迭代上，我们
     * 需要复制整个集合。
     * */
    // In this first iteration, we'll just live with hitting the bad case
    // (adding everything to a Map) in for every insert/move.
    /*
     * 在第一次迭代中，我们将在每次插入/移动中执行糟糕的情况(将所有内容添加到映射中)。
     * */
    // If you change this code, also update reconcileChildrenIterator() which
    // uses the same algorithm.

    if (__DEV__) {
      // First, validate keys.
      let knownKeys = null;
      for (let i = 0; i < newChildren.length; i++) {
        const child = newChildren[i];
        knownKeys = warnOnInvalidKey(child, knownKeys, returnFiber);
      }
    }
    /*
     * returnFiber：currentFirstChild的父级fiber节点
     * currentFirstChild：当前执行更新任务的WIP（fiber）节点
     * newChildren：组件的render方法渲染出的新的ReactElement节点
     * lanes：优先级相关
     * */

    // resultingFirstChild是diff之后的新fiber链表的第一个fiber。
    let resultingFirstChild: Fiber | null = null;
    // resultingFirstChild是新链表的第一个fiber。
    // previousNewFiber用来将后续的新fiber接到第一个fiber之后
    let previousNewFiber: Fiber | null = null;

    // 旧fiber节点，新的child节点会和它进行比较
    let oldFiber = currentFirstChild;
    // 存储固定节点的位置
    let lastPlacedIndex = 0;
    // 存储遍历的新节点它的索引
    let newIdx = 0;
    // 记录目前遍历到的旧fiber的下一个节点
    let nextOldFiber = null;

    // 该轮遍历来处理节点更新，依据节点是否可复用来决定是否中断遍历
    for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {
      // 新节点遍历完了，旧节点没有遍历完，此时需要中断遍历
      if (oldFiber.index > newIdx) {
        nextOldFiber = oldFiber;
        oldFiber = null;
      } else {
        // 用nextOldFiber存储当前遍历到的旧节点的下一个节点
        nextOldFiber = oldFiber.sibling;
      }
      // 生成新的节点，判断key与tag是否相同就在updateSlot中
      // 对DOM类型的元素来说，key 和 tag都相同才会复用旧节点
      // 并返回出去，否则返回null
      const newFiber = updateSlot(
        returnFiber,
        oldFiber,
        newChildren[newIdx],
        lanes
      );

      // newFiber为 null说明 key 或 tag 不同，节点不
      // 可复用，中断遍历
      if (newFiber === null) {
        if (oldFiber === null) {
          // oldFiber 为null说明旧fiber此时也遍历完了
          // 是以下场景，D为新增节点
          // 旧 A - B - C
          // 新 A - B - C - D
          oldFiber = nextOldFiber;
        }
        break;
      }
      if (shouldTrackSideEffects) {
        // shouldTrackSideEffects 为true表示是更新过程
        if (oldFiber && newFiber.alternate === null) {
          // newFiber.alternate 等同于 oldFiber.alternate
          // oldFiber为WIP节点，它的alternate 就是 current节点

          // 有旧节点，并且经过更新后的新节点它还没有current节点,
          // 说明更新后展现在屏幕上不会有current节点，而更新后WIP
          // 节点会称为current节点，所以需要删除已有的WIP节点
          deleteChild(returnFiber, oldFiber);
        }
      }
      // 记录固定节点的位置
      lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
      // 将新fiber连接成以sibling为指针的单向链表
      if (previousNewFiber === null) {
        resultingFirstChild = newFiber;
      } else {
        previousNewFiber.sibling = newFiber;
      }
      previousNewFiber = newFiber;

      // 将旧节点指向下一个，与新节点同步移动
      oldFiber = nextOldFiber;
    }

    // 新子节点遍历完，说明剩下的旧fiber都是没用的了，可以删除.
    if (newIdx === newChildren.length) {
      // 新节点遍历结束，删除掉旧节点中的剩下的节点
      deleteRemainingChildren(returnFiber, oldFiber);
      return resultingFirstChild;
    }
    if (oldFiber === null) {
      // 旧的遍历完了，能复用的都复用了，所以意味着新的都是新插入的了
      // 初始化渲染也是走的这
      for (; newIdx < newChildren.length; newIdx++) {
        // 基于新生成的ReactElement创建新的Fiber节点
        const newFiber = createChild(returnFiber, newChildren[newIdx], lanes);
        if (newFiber === null) {
          continue;
        }
        // 记录固定节点的位置lastPlacedIndex
        lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
        // 将新生成的fiber节点连接成以sibling为指针的单向链表
        if (previousNewFiber === null) {
          resultingFirstChild = newFiber;
        } else {
          previousNewFiber.sibling = newFiber;
        }
        previousNewFiber = newFiber;
      }
      return resultingFirstChild;
    }
    // 现在这种情况是都没遍历完的情况，把剩余的旧子节点放入一个以key为键,值为旧fiber节点的map中
    // 这样在基于旧fiber节点新建新的fiber节点时，可以通过key快速地找出旧fiber
    const existingChildren = mapRemainingChildren(returnFiber, oldFiber);

    // 继续遍历新子节点并且从map中找到被遍历到的当前节点在旧的fiber节点中的index，和lastPlacedIndex去比
    // 大于的话说明不用动，小于的话说明需要右移
    for (; newIdx < newChildren.length; newIdx++) {
      // 基于map中的旧fiber节点来创建新fiber
      const newFiber = updateFromMap(
        existingChildren,
        returnFiber,
        newIdx,
        newChildren[newIdx],
        lanes
      );
      if (newFiber !== null) {
        if (shouldTrackSideEffects) {
          if (newFiber.alternate !== null) {
            // 因为剩余的新节点有可能和旧fiber节点一样,只是位置换了,有可能是是新增的.

            // 如果newFiber的alternate不为空,则说明newFiber是和旧节点一样,而不是新增的.
            // 也就说明着它是基于map中的旧fiber节点新建的,意味着旧fiber已经被使用了,所以需
            // 要从map中删去旧fiber
            existingChildren.delete(
              newFiber.key === null ? newIdx : newFiber.key
            );
          }
        }
        // 调整位置，多节点diff的核心，这里真正会实现节点的移动
        lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
        // 将新fiber连接成以sibling为指针的单向链表
        if (previousNewFiber === null) {
          resultingFirstChild = newFiber;
        } else {
          previousNewFiber.sibling = newFiber;
        }
        previousNewFiber = newFiber;
      }
    }

    if (shouldTrackSideEffects) {
      // 此时新节点遍历完了，该移动的都移动了，那么删除剩下的旧节点
      existingChildren.forEach((child) => deleteChild(returnFiber, child));
    }
    return resultingFirstChild;
  }

  // This API will tag the children with the side-effect of the reconciliation
  // itself. They will be added to the side-effect list as we pass through the
  // children and the parent.
  // 该函数会在子节点上打上effect标签，然后在向上遍历的时候收集到side-effect列表上
  /*
   * newChild是组件render方法产生的ReactElement节点
   * currentFirstChild是当前的workInProgress节点
   * returnFiber是workInProgress节点的父节点
   *
   * React diff算法做的事情就是将新产生的节点,和已有的节点比对.
   * */
  function reconcileChildFibers(
    returnFiber: Fiber,
    currentFirstChild: Fiber | null,
    newChild: any,
    lanes: Lanes
  ): Fiber | null {
    // This function is not recursive.
    // If the top level item is an array, we treat it as a set of children,
    // not as a fragment. Nested arrays on the other hand will be treated as
    // fragment nodes. Recursion happens at the normal flow.

    // Handle top level unkeyed fragments as if they were arrays.
    // This leads to an ambiguity between <>{[...]}</> and <>...</>.
    // We treat the ambiguous cases above the same.
    const isUnkeyedTopLevelFragment =
      typeof newChild === "object" &&
      newChild !== null &&
      newChild.type === REACT_FRAGMENT_TYPE &&
      newChild.key === null;
    if (isUnkeyedTopLevelFragment) {
      newChild = newChild.props.children;
    }

    // Handle object types
    const isObject = typeof newChild === "object" && newChild !== null;

    if (isObject) {
      switch (newChild.$$typeof) {
        case REACT_ELEMENT_TYPE:
          return placeSingleChild(
            reconcileSingleElement(
              returnFiber,
              currentFirstChild,
              newChild,
              lanes
            )
          );
        case REACT_PORTAL_TYPE:
          return placeSingleChild(
            reconcileSinglePortal(
              returnFiber,
              currentFirstChild,
              newChild,
              lanes
            )
          );
        case REACT_LAZY_TYPE:
          if (enableLazyElements) {
            const payload = newChild._payload;
            const init = newChild._init;
            // TODO: This function is supposed to be non-recursive.
            return reconcileChildFibers(
              returnFiber,
              currentFirstChild,
              init(payload),
              lanes
            );
          }
      }
    }

    if (typeof newChild === "string" || typeof newChild === "number") {
      return placeSingleChild(
        reconcileSingleTextNode(
          returnFiber,
          currentFirstChild,
          "" + newChild,
          lanes
        )
      );
    }

    if (isArray(newChild)) {
      return reconcileChildrenArray(
        returnFiber,
        currentFirstChild,
        newChild,
        lanes
      );
    }

    if (getIteratorFn(newChild)) {
      return reconcileChildrenIterator(
        returnFiber,
        currentFirstChild,
        newChild,
        lanes
      );
    }

    if (isObject) {
      throwOnInvalidObjectType(returnFiber, newChild);
    }

    if (__DEV__) {
      if (typeof newChild === "function") {
        warnOnFunctionType(returnFiber);
      }
    }
    if (typeof newChild === "undefined" && !isUnkeyedTopLevelFragment) {
      // If the new child is undefined, and the return fiber is a composite
      // component, throw an error. If Fiber return types are disabled,
      // we already threw above.
      switch (returnFiber.tag) {
        case ClassComponent: {
          if (__DEV__) {
            const instance = returnFiber.stateNode;
            if (instance.render._isMockFunction) {
              // We allow auto-mocks to proceed as if they're returning null.
              break;
            }
          }
        }
        // Intentionally fall through to the next case, which handles both
        // functions and classes
        // eslint-disable-next-lined no-fallthrough
        case FunctionComponent: {
          const Component = returnFiber.type;
          invariant(
            false,
            "%s(...): Nothing was returned from render. This usually means a " +
              "return statement is missing. Or, to render nothing, " +
              "return null.",
            Component.displayName || Component.name || "Component"
          );
        }
      }
    }

    // Remaining cases are all treated as empty.
    return deleteRemainingChildren(returnFiber, currentFirstChild);
  }

  return reconcileChildFibers;
}
```

###### completeUnitOfWork

`location: v16.13.1/react-reconciler/src/ReactFiberWorkLoop.js`

```javascript
/**
 * Create Fiber object
 * Create an instance of each node and add it to the stateNode property
 * Collected to perform Fiber node of the DOM manipulation, forming effect chain table structure
 */
function completeUnitOfWork(unitOfWork: Fiber): void {
  // Attempt to complete the current unit of work, then move to the next
  // sibling. If there are no more siblings, return to the parent fiber.
  /*
   * 完成当前的工作单元，然后去处理下一个兄弟节点，如果没有兄弟节点，return到上级
   * Fiber节点
   * */
  let completedWork = unitOfWork;
  do {
    // The current, flushed, state of this fiber is the alternate. Ideally
    // nothing should rely on this, but relying on it here means that we don't
    // need an additional field on the work in progress.
    // 获取到current节点
    const current = completedWork.alternate;
    // 获取到父级节点
    const returnFiber = completedWork.return;

    // Check if the work completed or if something threw.
    if ((completedWork.effectTag & Incomplete) === NoEffect) {
      // 当前工作单元（Fiber）的任务已经完成
      setCurrentDebugFiberInDEV(completedWork);
      let next;
      // 是否开启React性能分析工具相关
      if (
        !enableProfilerTimer ||
        (completedWork.mode & ProfileMode) === NoMode
      ) {
        next = completeWork(current, completedWork, subtreeRenderLanes);
      } else {
        startProfilerTimer(completedWork);
        // 更新节点的属性，绑定事件等，将属性作为updateQueue挂载到WIP节点上
        // 结构为：[ 'style', { color: 'blue' }, onClick, this.handleClick ]
        next = completeWork(current, completedWork, subtreeRenderLanes);
        // Update render duration assuming we didn't error.
        stopProfilerTimerIfRunningAndRecordDelta(completedWork, false);
      }
      resetCurrentDebugFiberInDEV();

      if (next !== null) {
        // Completing this fiber spawned new work. Work on that next.
        workInProgress = next;
        return;
      }
      // 收集WIP节点的lanes，不漏掉被跳过的update的lanes，便于再次发起调度
      resetChildLanes(completedWork);

      if (
        returnFiber !== null &&
        // Do not append effects to parents if a sibling failed to complete
        (returnFiber.effectTag & Incomplete) === NoEffect
      ) {
        // Append all the effects of the subtree and this fiber onto the effect
        // list of the parent. The completion order of the children affects the
        // side-effect order.
        /*
         * effectList是一条单向链表，每完成一个工作单元上的任务，都要将它产生的effect链表并入
         * 上级工作单元。
         * */
        // 链表的并入操作
        // 将当前节点的effectList并入到父节点的effectList
        if (returnFiber.firstEffect === null) {
          returnFiber.firstEffect = completedWork.firstEffect;
        }
        if (completedWork.lastEffect !== null) {
          if (returnFiber.lastEffect !== null) {
            returnFiber.lastEffect.nextEffect = completedWork.firstEffect;
          }
          returnFiber.lastEffect = completedWork.lastEffect;
        }

        // If this fiber had side-effects, we append it AFTER the children's
        // side-effects. We can perform certain side-effects earlier if needed,
        // by doing multiple passes over the effect list. We don't want to
        // schedule our own side-effect on our own list because if end up
        // reusing children we'll schedule this effect onto itself since we're
        // at the end.
        /*
         * 如果这个Fiber有副作用，我们会在子Fiber的副作用之后添加它。如果需要，我们可以通过对效果列
         * 表进行多次传递，提前执行某些副作用。我们不想把自己的副作用放在自己的effect-list上，因为如
         * 果最终复用了子Fiber，我们将把这个效果安排在自己身上，因为我们已经到了最后。
         * */
        const effectTag = completedWork.effectTag;

        // Skip both NoWork and PerformedWork tags when creating the effect
        // list. PerformedWork effect is read by React DevTools but shouldn't be
        // committed.
        // 链表元素的实际添加，添加时跳过NoWork 和 PerformedWork的effectTag，因为真正
        // 的commit用不到
        if (effectTag > PerformedWork) {
          if (returnFiber.lastEffect !== null) {
            returnFiber.lastEffect.nextEffect = completedWork;
          } else {
            returnFiber.firstEffect = completedWork;
          }
          returnFiber.lastEffect = completedWork;
        }
      }
    } else {
      // This fiber did not complete because something threw. Pop values off
      // the stack without entering the complete phase. If this is a boundary,
      // capture values if possible.
      /*
       * 这Fiber不完整，因为抛出了一些错误。在进入完成阶段之前从堆栈中弹出。如果这是一个边界，
       * 尽可能捕获错误。
       * */
      // 因为出错后会被添加上Incomplete 类型的effectTag，所以会被执行到这里
      // unwindWork主要是将WIP节点上的ShouldCapture去掉，换成 DidCapture
      const next = unwindWork(completedWork, subtreeRenderLanes);

      // Because this fiber did not complete, don't reset its expiration time.
      // 逐层往上标记effectTag |= Incomplete
      if (next !== null) {
        // If completing this work spawned new work, do that next. We'll come
        // back here again.
        // Since we're restarting, remove anything that is not a host effect
        // from the effect tag.
        // 如果完成此工作产生了新任务，则执行下一步，之后会再回到这里，由于需要重启，所以从effect
        // 标签中删除所有不是host effect的effectTag。
        next.effectTag &= HostEffectMask;
        workInProgress = next;
        return;
      }

      if (
        enableProfilerTimer &&
        (completedWork.mode & ProfileMode) !== NoMode
      ) {
        // Record the render duration for the fiber that errored.
        stopProfilerTimerIfRunningAndRecordDelta(completedWork, false);

        // Include the time spent working on failed children before continuing.
        let actualDuration = completedWork.actualDuration;
        let child = completedWork.child;
        while (child !== null) {
          actualDuration += child.actualDuration;
          child = child.sibling;
        }
        completedWork.actualDuration = actualDuration;
      }

      if (returnFiber !== null) {
        // Mark the parent fiber as incomplete and clear its effect list.
        // 将父Fiber的effect list清除，effectTag标记成未完成，便于它的父节点再completeWork的时候
        // 被unwindWork
        returnFiber.firstEffect = returnFiber.lastEffect = null;
        returnFiber.effectTag |= Incomplete;
      }
    }

    const siblingFiber = completedWork.sibling;
    if (siblingFiber !== null) {
      // If there is more work to do in this returnFiber, do that next.
      workInProgress = siblingFiber;
      return;
    }
    // Otherwise, return to the parent
    completedWork = returnFiber;
    // Update the next thing we're working on in case something throws.
    workInProgress = completedWork;
  } while (completedWork !== null);

  // We've reached the root.
  if (workInProgressRootExitStatus === RootIncomplete) {
    workInProgressRootExitStatus = RootCompleted;
  }
}
```

###### completeWork

`location: v16.13.1/react-reconciler/src/ReactFiberCompleteWork.js`

```javascript
function completeWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes
): Fiber | null {
  const newProps = workInProgress.pendingProps;

  switch (workInProgress.tag) {
    case IndeterminateComponent:
    case LazyComponent:
    case SimpleMemoComponent:
    case FunctionComponent:
    case ForwardRef:
    case Fragment:
    case Mode:
    case Profiler:
    case ContextConsumer:
    case MemoComponent:
      return null;
    case ClassComponent: {
      const Component = workInProgress.type;
      if (isLegacyContextProvider(Component)) {
        popLegacyContext(workInProgress);
      }
      return null;
    }
    case HostRoot: {
      popHostContainer(workInProgress);
      popTopLevelLegacyContextObject(workInProgress);
      resetMutableSourceWorkInProgressVersions();
      const fiberRoot = (workInProgress.stateNode: FiberRoot);
      if (fiberRoot.pendingContext) {
        fiberRoot.context = fiberRoot.pendingContext;
        fiberRoot.pendingContext = null;
      }
      if (current === null || current.child === null) {
        // If we hydrated, pop so that we can delete any remaining children
        // that weren't hydrated.
        const wasHydrated = popHydrationState(workInProgress);
        if (wasHydrated) {
          // If we hydrated, then we'll need to schedule an update for
          // the commit side-effects on the root.
          markUpdate(workInProgress);
        } else if (!fiberRoot.hydrate) {
          // Schedule an effect to clear this container at the start of the next commit.
          // This handles the case of React rendering into a container with previous children.
          // It's also safe to do for updates too, because current.child would only be null
          // if the previous render was null (so the the container would already be empty).
          workInProgress.effectTag |= Snapshot;
        }
      }
      updateHostContainer(workInProgress);
      return null;
    }
    case HostComponent: {
      popHostContext(workInProgress);
      const rootContainerInstance = getRootHostContainer();
      const type = workInProgress.type;
      if (current !== null && workInProgress.stateNode != null) {
        updateHostComponent(
          current,
          workInProgress,
          type,
          newProps,
          rootContainerInstance
        );

        if (enableDeprecatedFlareAPI) {
          const prevListeners = current.memoizedProps.DEPRECATED_flareListeners;
          const nextListeners = newProps.DEPRECATED_flareListeners;
          if (prevListeners !== nextListeners) {
            markUpdate(workInProgress);
          }
        }

        if (current.ref !== workInProgress.ref) {
          markRef(workInProgress);
        }
      } else {
        if (!newProps) {
          invariant(
            workInProgress.stateNode !== null,
            "We must have new props for new mounts. This error is likely " +
              "caused by a bug in React. Please file an issue."
          );
          // This can happen when we abort work.
          return null;
        }

        const currentHostContext = getHostContext();
        // TODO: Move createInstance to beginWork and keep it on a context
        // "stack" as the parent. Then append children as we go in beginWork
        // or completeWork depending on whether we want to add them top->down or
        // bottom->up. Top->down is faster in IE11.
        const wasHydrated = popHydrationState(workInProgress);
        if (wasHydrated) {
          // TODO: Move this and createInstance step into the beginPhase
          // to consolidate.
          if (
            prepareToHydrateHostInstance(
              workInProgress,
              rootContainerInstance,
              currentHostContext
            )
          ) {
            // If changes to the hydrated node need to be applied at the
            // commit-phase we mark this as such.
            markUpdate(workInProgress);
          }
          if (enableDeprecatedFlareAPI) {
            const listeners = newProps.DEPRECATED_flareListeners;
            if (listeners != null) {
              updateDeprecatedEventListeners(
                listeners,
                workInProgress,
                rootContainerInstance
              );
            }
          }
        } else {
          // Create node instance object
          const instance = createInstance(
            type,
            newProps,
            rootContainerInstance,
            currentHostContext,
            workInProgress
          );

          // Append all children to the parent
          // instance is parent
          // workInProgress.child is children
          appendAllChildren(instance, workInProgress, false, false);

          // This needs to be set before we mount Flare event listeners
          // Add stateNode attribute to Fiber object
          workInProgress.stateNode = instance;

          if (enableDeprecatedFlareAPI) {
            const listeners = newProps.DEPRECATED_flareListeners;
            if (listeners != null) {
              updateDeprecatedEventListeners(
                listeners,
                workInProgress,
                rootContainerInstance
              );
            }
          }

          // Certain renderers require commit-time effects for initial mount.
          // (eg DOM renderer supports auto-focus for certain elements).
          // Make sure such renderers get scheduled for later work.
          // 处理props等过程，回调注册等
          if (
            finalizeInitialChildren(
              instance,
              type,
              newProps,
              rootContainerInstance,
              currentHostContext
            )
          ) {
            markUpdate(workInProgress);
          }
        }

        if (workInProgress.ref !== null) {
          // If there is a ref on a host node we need to schedule a callback
          markRef(workInProgress);
        }
      }
      return null;
    }
    case HostText: {
      const newText = newProps;
      if (current && workInProgress.stateNode != null) {
        const oldText = current.memoizedProps;
        // If we have an alternate, that means this is an update and we need
        // to schedule a side-effect to do the updates.
        updateHostText(current, workInProgress, oldText, newText);
      } else {
        if (typeof newText !== "string") {
          invariant(
            workInProgress.stateNode !== null,
            "We must have new props for new mounts. This error is likely " +
              "caused by a bug in React. Please file an issue."
          );
          // This can happen when we abort work.
        }
        const rootContainerInstance = getRootHostContainer();
        const currentHostContext = getHostContext();
        const wasHydrated = popHydrationState(workInProgress);
        if (wasHydrated) {
          if (prepareToHydrateHostTextInstance(workInProgress)) {
            markUpdate(workInProgress);
          }
        } else {
          workInProgress.stateNode = createTextInstance(
            newText,
            rootContainerInstance,
            currentHostContext,
            workInProgress
          );
        }
      }
      return null;
    }
    case SuspenseComponent: {
      popSuspenseContext(workInProgress);
      const nextState: null | SuspenseState = workInProgress.memoizedState;

      if (enableSuspenseServerRenderer) {
        if (nextState !== null && nextState.dehydrated !== null) {
          if (current === null) {
            const wasHydrated = popHydrationState(workInProgress);
            invariant(
              wasHydrated,
              "A dehydrated suspense component was completed without a hydrated node. " +
                "This is probably a bug in React."
            );
            prepareToHydrateHostSuspenseInstance(workInProgress);
            if (enableSchedulerTracing) {
              markSpawnedWork(OffscreenLane);
            }
            return null;
          } else {
            // We should never have been in a hydration state if we didn't have a current.
            // However, in some of those paths, we might have reentered a hydration state
            // and then we might be inside a hydration state. In that case, we'll need to exit out of it.
            resetHydrationState();
            if ((workInProgress.effectTag & DidCapture) === NoEffect) {
              // This boundary did not suspend so it's now hydrated and unsuspended.
              workInProgress.memoizedState = null;
            }
            // If nothing suspended, we need to schedule an effect to mark this boundary
            // as having hydrated so events know that they're free to be invoked.
            // It's also a signal to replay events and the suspense callback.
            // If something suspended, schedule an effect to attach retry listeners.
            // So we might as well always mark this.
            workInProgress.effectTag |= Update;
            return null;
          }
        }
      }

      if ((workInProgress.effectTag & DidCapture) !== NoEffect) {
        // Something suspended. Re-render with the fallback children.
        // 被挂起了，用之前的子节点重新渲染
        workInProgress.lanes = renderLanes;
        // Do not reset the effect list.
        if (
          enableProfilerTimer &&
          (workInProgress.mode & ProfileMode) !== NoMode
        ) {
          transferActualDuration(workInProgress);
        }
        return workInProgress;
      }

      const nextDidTimeout = nextState !== null;
      let prevDidTimeout = false;
      if (current === null) {
        if (workInProgress.memoizedProps.fallback !== undefined) {
          popHydrationState(workInProgress);
        }
      } else {
        const prevState: null | SuspenseState = current.memoizedState;
        prevDidTimeout = prevState !== null;
      }

      if (nextDidTimeout && !prevDidTimeout) {
        // If this subtreee is running in blocking mode we can suspend,
        // otherwise we won't suspend.
        // TODO: This will still suspend a synchronous tree if anything
        // in the concurrent tree already suspended during this render.
        // This is a known bug.
        if ((workInProgress.mode & BlockingMode) !== NoMode) {
          // TODO: Move this back to throwException because this is too late
          // if this is a large tree which is common for initial loads. We
          // don't know if we should restart a render or not until we get
          // this marker, and this is too late.
          // If this render already had a ping or lower pri updates,
          // and this is the first time we know we're going to suspend we
          // should be able to immediately restart from within throwException.
          const hasInvisibleChildContext =
            current === null &&
            workInProgress.memoizedProps.unstable_avoidThisFallback !== true;
          if (
            hasInvisibleChildContext ||
            hasSuspenseContext(
              suspenseStackCursor.current,
              (InvisibleParentSuspenseContext: SuspenseContext)
            )
          ) {
            // If this was in an invisible tree or a new render, then showing
            // this boundary is ok.
            renderDidSuspend();
          } else {
            // Otherwise, we're going to have to hide content so we should
            // suspend for longer if possible.
            renderDidSuspendDelayIfPossible();
          }
        }
      }

      if (supportsPersistence) {
        // TODO: Only schedule updates if not prevDidTimeout.
        if (nextDidTimeout) {
          // If this boundary just timed out, schedule an effect to attach a
          // retry listener to the promise. This flag is also used to hide the
          // primary children.
          workInProgress.effectTag |= Update;
        }
      }
      if (supportsMutation) {
        // TODO: Only schedule updates if these values are non equal, i.e. it changed.
        if (nextDidTimeout || prevDidTimeout) {
          // If this boundary just timed out, schedule an effect to attach a
          // retry listener to the promise. This flag is also used to hide the
          // primary children. In mutation mode, we also need the flag to
          // *unhide* children that were previously hidden, so check if this
          // is currently timed out, too.
          workInProgress.effectTag |= Update;
        }
      }
      if (
        enableSuspenseCallback &&
        workInProgress.updateQueue !== null &&
        workInProgress.memoizedProps.suspenseCallback != null
      ) {
        // Always notify the callback
        workInProgress.effectTag |= Update;
      }
      return null;
    }
    case HostPortal:
      popHostContainer(workInProgress);
      updateHostContainer(workInProgress);
      if (current === null) {
        preparePortalMount(workInProgress.stateNode.containerInfo);
      }
      return null;
    case ContextProvider:
      // Pop provider fiber
      popProvider(workInProgress);
      return null;
    case IncompleteClassComponent: {
      // Same as class component case. I put it down here so that the tags are
      // sequential to ensure this switch is compiled to a jump table.
      const Component = workInProgress.type;
      if (isLegacyContextProvider(Component)) {
        popLegacyContext(workInProgress);
      }
      return null;
    }
    case SuspenseListComponent: {
      popSuspenseContext(workInProgress);

      const renderState: null | SuspenseListRenderState =
        workInProgress.memoizedState;

      if (renderState === null) {
        // We're running in the default, "independent" mode.
        // We don't do anything in this mode.
        return null;
      }

      let didSuspendAlready =
        (workInProgress.effectTag & DidCapture) !== NoEffect;

      const renderedTail = renderState.rendering;
      if (renderedTail === null) {
        // We just rendered the head.
        if (!didSuspendAlready) {
          // This is the first pass. We need to figure out if anything is still
          // suspended in the rendered set.

          // If new content unsuspended, but there's still some content that
          // didn't. Then we need to do a second pass that forces everything
          // to keep showing their fallbacks.

          // We might be suspended if something in this render pass suspended, or
          // something in the previous committed pass suspended. Otherwise,
          // there's no chance so we can skip the expensive call to
          // findFirstSuspended.
          const cannotBeSuspended =
            renderHasNotSuspendedYet() &&
            (current === null || (current.effectTag & DidCapture) === NoEffect);
          if (!cannotBeSuspended) {
            let row = workInProgress.child;
            while (row !== null) {
              const suspended = findFirstSuspended(row);
              if (suspended !== null) {
                didSuspendAlready = true;
                workInProgress.effectTag |= DidCapture;
                cutOffTailIfNeeded(renderState, false);

                // If this is a newly suspended tree, it might not get committed as
                // part of the second pass. In that case nothing will subscribe to
                // its thennables. Instead, we'll transfer its thennables to the
                // SuspenseList so that it can retry if they resolve.
                // There might be multiple of these in the list but since we're
                // going to wait for all of them anyway, it doesn't really matter
                // which ones gets to ping. In theory we could get clever and keep
                // track of how many dependencies remain but it gets tricky because
                // in the meantime, we can add/remove/change items and dependencies.
                // We might bail out of the loop before finding any but that
                // doesn't matter since that means that the other boundaries that
                // we did find already has their listeners attached.
                const newThennables = suspended.updateQueue;
                if (newThennables !== null) {
                  workInProgress.updateQueue = newThennables;
                  workInProgress.effectTag |= Update;
                }

                // Rerender the whole list, but this time, we'll force fallbacks
                // to stay in place.
                // Reset the effect list before doing the second pass since that's now invalid.
                if (renderState.lastEffect === null) {
                  workInProgress.firstEffect = null;
                }
                workInProgress.lastEffect = renderState.lastEffect;
                // Reset the child fibers to their original state.
                resetChildFibers(workInProgress, renderLanes);

                // Set up the Suspense Context to force suspense and immediately
                // rerender the children.
                pushSuspenseContext(
                  workInProgress,
                  setShallowSuspenseContext(
                    suspenseStackCursor.current,
                    ForceSuspenseFallback
                  )
                );
                return workInProgress.child;
              }
              row = row.sibling;
            }
          }
        } else {
          cutOffTailIfNeeded(renderState, false);
        }
        // Next we're going to render the tail.
      } else {
        // Append the rendered row to the child list.
        if (!didSuspendAlready) {
          const suspended = findFirstSuspended(renderedTail);
          if (suspended !== null) {
            workInProgress.effectTag |= DidCapture;
            didSuspendAlready = true;

            // Ensure we transfer the update queue to the parent so that it doesn't
            // get lost if this row ends up dropped during a second pass.
            const newThennables = suspended.updateQueue;
            if (newThennables !== null) {
              workInProgress.updateQueue = newThennables;
              workInProgress.effectTag |= Update;
            }

            cutOffTailIfNeeded(renderState, true);
            // This might have been modified.
            if (
              renderState.tail === null &&
              renderState.tailMode === "hidden" &&
              !renderedTail.alternate &&
              !getIsHydrating() // We don't cut it if we're hydrating.
            ) {
              // We need to delete the row we just rendered.
              // Reset the effect list to what it was before we rendered this
              // child. The nested children have already appended themselves.
              const lastEffect = (workInProgress.lastEffect =
                renderState.lastEffect);
              // Remove any effects that were appended after this point.
              if (lastEffect !== null) {
                lastEffect.nextEffect = null;
              }
              // We're done.
              return null;
            }
          } else if (
            // The time it took to render last row is greater than time until
            // the expiration.
            now() * 2 - renderState.renderingStartTime >
              renderState.tailExpiration &&
            renderLanes !== OffscreenLane
          ) {
            // We have now passed our CPU deadline and we'll just give up further
            // attempts to render the main content and only render fallbacks.
            // The assumption is that this is usually faster.
            workInProgress.effectTag |= DidCapture;
            didSuspendAlready = true;

            cutOffTailIfNeeded(renderState, false);

            // Since nothing actually suspended, there will nothing to ping this
            // to get it started back up to attempt the next item. If we can show
            // them, then they really have the same priority as this render.
            // So we'll pick it back up the very next render pass once we've had
            // an opportunity to yield for paint.
            workInProgress.lanes = renderLanes;
            if (enableSchedulerTracing) {
              markSpawnedWork(renderLanes);
            }
          }
        }
        if (renderState.isBackwards) {
          // The effect list of the backwards tail will have been added
          // to the end. This breaks the guarantee that life-cycles fire in
          // sibling order but that isn't a strong guarantee promised by React.
          // Especially since these might also just pop in during future commits.
          // Append to the beginning of the list.
          renderedTail.sibling = workInProgress.child;
          workInProgress.child = renderedTail;
        } else {
          const previousSibling = renderState.last;
          if (previousSibling !== null) {
            previousSibling.sibling = renderedTail;
          } else {
            workInProgress.child = renderedTail;
          }
          renderState.last = renderedTail;
        }
      }

      if (renderState.tail !== null) {
        // We still have tail rows to render.
        if (renderState.tailExpiration === 0) {
          // Heuristic for how long we're willing to spend rendering rows
          // until we just give up and show what we have so far.
          const TAIL_EXPIRATION_TIMEOUT_MS = 500;
          renderState.tailExpiration = now() + TAIL_EXPIRATION_TIMEOUT_MS;
          // TODO: This is meant to mimic the train model or JND but this
          // is a per component value. It should really be since the start
          // of the total render or last commit. Consider using something like
          // globalMostRecentFallbackTime. That doesn't account for being
          // suspended for part of the time or when it's a new render.
          // It should probably use a global start time value instead.
        }
        // Pop a row.
        const next = renderState.tail;
        renderState.rendering = next;
        renderState.tail = next.sibling;
        renderState.lastEffect = workInProgress.lastEffect;
        renderState.renderingStartTime = now();
        next.sibling = null;

        // Restore the context.
        // TODO: We can probably just avoid popping it instead and only
        // setting it the first time we go from not suspended to suspended.
        let suspenseContext = suspenseStackCursor.current;
        if (didSuspendAlready) {
          suspenseContext = setShallowSuspenseContext(
            suspenseContext,
            ForceSuspenseFallback
          );
        } else {
          suspenseContext = setDefaultShallowSuspenseContext(suspenseContext);
        }
        pushSuspenseContext(workInProgress, suspenseContext);
        // Do a pass over the next row.
        return next;
      }
      return null;
    }
    case FundamentalComponent: {
      if (enableFundamentalAPI) {
        const fundamentalImpl = workInProgress.type.impl;
        let fundamentalInstance: ReactFundamentalComponentInstance<
          any,
          any
        > | null = workInProgress.stateNode;

        if (fundamentalInstance === null) {
          const getInitialState = fundamentalImpl.getInitialState;
          let fundamentalState;
          if (getInitialState !== undefined) {
            fundamentalState = getInitialState(newProps);
          }
          fundamentalInstance = workInProgress.stateNode =
            createFundamentalStateInstance(
              workInProgress,
              newProps,
              fundamentalImpl,
              fundamentalState || {}
            );
          const instance = ((getFundamentalComponentInstance(
            fundamentalInstance
          ): any): Instance);
          fundamentalInstance.instance = instance;
          if (fundamentalImpl.reconcileChildren === false) {
            return null;
          }
          appendAllChildren(instance, workInProgress, false, false);
          mountFundamentalComponent(fundamentalInstance);
        } else {
          // We fire update in commit phase
          const prevProps = fundamentalInstance.props;
          fundamentalInstance.prevProps = prevProps;
          fundamentalInstance.props = newProps;
          fundamentalInstance.currentFiber = workInProgress;
          if (supportsPersistence) {
            const instance = cloneFundamentalInstance(fundamentalInstance);
            fundamentalInstance.instance = instance;
            appendAllChildren(instance, workInProgress, false, false);
          }
          const shouldUpdate =
            shouldUpdateFundamentalComponent(fundamentalInstance);
          if (shouldUpdate) {
            markUpdate(workInProgress);
          }
        }
        return null;
      }
      break;
    }
    case ScopeComponent: {
      if (enableScopeAPI) {
        if (current === null) {
          const scopeInstance: ReactScopeInstance = createScopeInstance();
          workInProgress.stateNode = scopeInstance;
          if (enableDeprecatedFlareAPI) {
            const listeners = newProps.DEPRECATED_flareListeners;
            if (listeners != null) {
              const rootContainerInstance = getRootHostContainer();
              updateDeprecatedEventListeners(
                listeners,
                workInProgress,
                rootContainerInstance
              );
            }
          }
          prepareScopeUpdate(scopeInstance, workInProgress);
          if (workInProgress.ref !== null) {
            markRef(workInProgress);
            markUpdate(workInProgress);
          }
        } else {
          if (enableDeprecatedFlareAPI) {
            const prevListeners =
              current.memoizedProps.DEPRECATED_flareListeners;
            const nextListeners = newProps.DEPRECATED_flareListeners;
            if (
              prevListeners !== nextListeners ||
              workInProgress.ref !== null
            ) {
              markUpdate(workInProgress);
            }
          } else {
            if (workInProgress.ref !== null) {
              markUpdate(workInProgress);
            }
          }
          if (current.ref !== workInProgress.ref) {
            markRef(workInProgress);
          }
        }
        return null;
      }
      break;
    }
    case Block:
      if (enableBlocksAPI) {
        return null;
      }
      break;
    case OffscreenComponent:
    case LegacyHiddenComponent: {
      popRenderLanes(workInProgress);
      if (current !== null) {
        const nextState: OffscreenState | null = workInProgress.memoizedState;
        const prevState: OffscreenState | null = current.memoizedState;

        const prevIsHidden = prevState !== null;
        const nextIsHidden = nextState !== null;
        if (
          prevIsHidden !== nextIsHidden &&
          newProps.mode !== "unstable-defer-without-hiding"
        ) {
          workInProgress.effectTag |= Update;
        }
      }
      return null;
    }
  }
  invariant(
    false,
    "Unknown unit of work tag (%s). This error is likely caused by a bug in " +
      "React. Please file an issue.",
    workInProgress.tag
  );
}
```

###### appendAllChildren

`location: v16.13.1/react-reconciler/src/ReactFiberCompleteWork.js`

```javascript
let appendAllChildren = function (
  parent: Instance,
  workInProgress: Fiber,
  needsVisibilityToggle: boolean,
  isHidden: boolean
) {
  // We only have the top Fiber that was created but we need recurse down its
  // children to find all the terminal nodes.
  let node = workInProgress.child;
  while (node !== null) {
    if (node.tag === HostComponent || node.tag === HostText) {
      appendInitialChild(parent, node.stateNode);
    } else if (enableFundamentalAPI && node.tag === FundamentalComponent) {
      appendInitialChild(parent, node.stateNode.instance);
    } else if (node.tag === HostPortal) {
      // If we have a portal child, then we don't want to traverse
      // down its children. Instead, we'll get insertions from each child in
      // the portal directly.
    } else if (node.child !== null) {
      node.child.return = node;
      node = node.child;
      continue;
    }
    if (node === workInProgress) {
      return;
    }
    while (node.sibling === null) {
      if (node.return === null || node.return === workInProgress) {
        return;
      }
      node = node.return;
    }
    node.sibling.return = node.return;
    node = node.sibling;
  }
};
```

#### Commit

The commit phase can be divided into three sub-phases:

- before mutation phase (before DOM operations are performed)
- mutation phase (when DOM operations are performed)
- layout phase (after DOM operations are performed)

###### commitRoot

`location: v16.13.1/react-reconciler/src/ReactFiberWorkLoop.js`

```javascript
function commitRoot(root) {
  // Acquisition task priority - > common priority
  const renderPriorityLevel = getCurrentPriorityLevel();

  // Use the highest priority to execute the current task, because the commit phase cannot be interrupted
  // ImmediateSchedulerPriority - > Highest Priority
  runWithPriority(
    ImmediateSchedulerPriority,
    commitRootImpl.bind(null, root, renderPriorityLevel)
  );
  return null;
}
```

###### commitRootImpl

`location: v16.13.1/react-reconciler/src/ReactFiberWorkLoop.js`

```javascript
function commitRootImpl(root, renderPriorityLevel) {
  // https://zh-hans.reactjs.org/docs/hooks-reference.html#timing-of-effects
  // effect会保证在任何新的渲染前执行。React 将在组件更新前刷新上一轮渲染的 effect。
  do {
    // `flushPassiveEffects` will call `flushSyncUpdateQueue` at the end, which
    // means `flushPassiveEffects` will sometimes result in additional
    // passive effects. So we need to keep flushing in a loop until there are
    // no more pending effects.
    // TODO: Might be better if `flushPassiveEffects` did not automatically
    // flush synchronous work at the end, to avoid factoring hazards like this.
    flushPassiveEffects();
  } while (rootWithPendingPassiveEffects !== null);
  flushRenderPhaseStrictModeWarningsInDEV();

  invariant(
    (executionContext & (RenderContext | CommitContext)) === NoContext,
    "Should not already be working."
  );

  const finishedWork = root.finishedWork;
  const lanes = root.finishedLanes;
  if (finishedWork === null) {
    return null;
  }
  root.finishedWork = null;
  root.finishedLanes = NoLanes;

  invariant(
    finishedWork !== root.current,
    "Cannot commit the same tree as before. This error is likely caused by " +
      "a bug in React. Please file an issue."
  );

  // commitRoot never returns a continuation; it always finishes synchronously.
  // So we can clear these now to allow a new callback to be scheduled.
  /*
   * commitRoot是同步完成的，所以现在能清理掉这些，便于让一个新的回调被调度
   * */
  root.callbackNode = null;
  root.callbackId = NoLanes;

  // Update the first and last pending times on this root. The new first
  // pending time is whatever is left on the root fiber.
  let remainingLanes = mergeLanes(finishedWork.lanes, finishedWork.childLanes);
  markRootFinished(root, remainingLanes);
  console.log(
    "更新完成后的渲染优先级（root.pendingLanes）：",
    root.pendingLanes
  );
  // Clear already finished discrete updates in case that a later call of
  // `flushDiscreteUpdates` starts a useless render pass which may cancels
  // a scheduled timeout.
  /*
   * 清除已经完成的离散的更新，万一flushDiscreteUpdates被延迟调用启动了一次不必要的渲染，
   * 这可能会取消timeout
   * */
  if (rootsWithPendingDiscreteUpdates !== null) {
    if (
      !hasDiscreteLanes(remainingLanes) &&
      rootsWithPendingDiscreteUpdates.has(root)
    ) {
      rootsWithPendingDiscreteUpdates.delete(root);
    }
  }

  if (root === workInProgressRoot) {
    // We can reset these now that they are finished.
    workInProgressRoot = null;
    workInProgress = null;
    workInProgressRootRenderLanes = NoLanes;
  } else {
    // This indicates that the last root we worked on is not the same one that
    // we're committing now. This most commonly happens when a suspended root
    // times out.
  }

  // Get the list of effects.
  let firstEffect;
  if (finishedWork.effectTag > PerformedWork) {
    // A fiber's effect list consists only of its children, not itself. So if
    // the root has an effect, we need to add it to the end of the list. The
    // resulting list is the set that would belong to the root's parent, if it
    // had one; that is, all the effects in the tree including the root.
    /*
     * fiber的effectList只包含子节点的effect，不包含它自身的，所以如果root节点有
     * 一个effect，需要把它放到effectList 的末尾，保证effectList里包含了所有的有
     * effect的fiber
     * */
    if (finishedWork.lastEffect !== null) {
      finishedWork.lastEffect.nextEffect = finishedWork;
      firstEffect = finishedWork.firstEffect;
    } else {
      firstEffect = finishedWork;
    }
  } else {
    // There is no effect on the root.
    firstEffect = finishedWork.firstEffect;
  }

  if (firstEffect !== null) {
    const prevExecutionContext = executionContext;
    executionContext |= CommitContext;
    const prevInteractions = pushInteractions(root);

    // Reset this to null before calling lifecycles
    ReactCurrentOwner.current = null;

    // The commit phase is broken into several sub-phases. We do a separate pass
    // of the effect list for each phase: all mutation effects come before all
    // layout effects, and so on.

    // The first phase a "before mutation" phase. We use this phase to read the
    // state of the host tree right before we mutate it. This is where
    // getSnapshotBeforeUpdate is called.
    focusedInstanceHandle = prepareForCommit(root.containerInfo);
    shouldFireAfterActiveInstanceBlur = false;

    nextEffect = firstEffect;
    do {
      if (__DEV__) {
        invokeGuardedCallback(null, commitBeforeMutationEffects, null);
        if (hasCaughtError()) {
          invariant(nextEffect !== null, "Should be working on an effect.");
          const error = clearCaughtError();
          captureCommitPhaseError(nextEffect, error);
          nextEffect = nextEffect.nextEffect;
        }
      } else {
        try {
          // 调用getSnapshotBeforeUpdate，在即将更新前获取DOM信息
          commitBeforeMutationEffects();
        } catch (error) {
          invariant(nextEffect !== null, "Should be working on an effect.");
          captureCommitPhaseError(nextEffect, error);
          nextEffect = nextEffect.nextEffect;
        }
      }
    } while (nextEffect !== null);

    // We no longer need to track the active instance fiber
    focusedInstanceHandle = null;

    if (enableProfilerTimer) {
      // Mark the current commit time to be shared by all Profilers in this
      // batch. This enables them to be grouped later.
      recordCommitTime();
    }

    // The next phase is the mutation phase, where we mutate the host tree.
    // 更新DOM阶段
    nextEffect = firstEffect;
    do {
      if (__DEV__) {
        invokeGuardedCallback(
          null,
          commitMutationEffects,
          null,
          root,
          renderPriorityLevel
        );
        if (hasCaughtError()) {
          invariant(nextEffect !== null, "Should be working on an effect.");
          const error = clearCaughtError();
          captureCommitPhaseError(nextEffect, error);
          nextEffect = nextEffect.nextEffect;
        }
      } else {
        try {
          commitMutationEffects(root, renderPriorityLevel);
        } catch (error) {
          invariant(nextEffect !== null, "Should be working on an effect.");
          captureCommitPhaseError(nextEffect, error);
          nextEffect = nextEffect.nextEffect;
        }
      }
    } while (nextEffect !== null);

    if (shouldFireAfterActiveInstanceBlur) {
      afterActiveInstanceBlur();
    }
    resetAfterCommit(root.containerInfo);

    // The work-in-progress tree is now the current tree. This must come after
    // the mutation phase, so that the previous tree is still current during
    // componentWillUnmount, but before the layout phase, so that the finished
    // work is current during componentDidMount/Update.
    root.current = finishedWork;

    // The next phase is the layout phase, where we call effects that read
    // the host tree after it's been mutated. The idiomatic use case for this is
    // layout, but class component lifecycles also fire here for legacy reasons.
    nextEffect = firstEffect;
    do {
      if (__DEV__) {
        invokeGuardedCallback(null, commitLayoutEffects, null, root, lanes);
        if (hasCaughtError()) {
          invariant(nextEffect !== null, "Should be working on an effect.");
          const error = clearCaughtError();
          captureCommitPhaseError(nextEffect, error);
          nextEffect = nextEffect.nextEffect;
        }
      } else {
        try {
          commitLayoutEffects(root, lanes);
        } catch (error) {
          invariant(nextEffect !== null, "Should be working on an effect.");
          captureCommitPhaseError(nextEffect, error);
          nextEffect = nextEffect.nextEffect;
        }
      }
    } while (nextEffect !== null);

    nextEffect = null;

    // Tell Scheduler to yield at the end of the frame, so the browser has an
    // opportunity to paint.
    // 通知Scheduler在帧的结尾让出执行权，来让浏览器有机会去绘制
    requestPaint();

    if (enableSchedulerTracing) {
      popInteractions(((prevInteractions: any): Set<Interaction>));
    }
    executionContext = prevExecutionContext;
  } else {
    // No effects.
    root.current = finishedWork;
    // Measure these anyway so the flamegraph explicitly shows that there were
    // no effects.
    // TODO: Maybe there's a better way to report this.
    if (enableProfilerTimer) {
      recordCommitTime();
    }
  }

  const rootDidHavePassiveEffects = rootDoesHavePassiveEffects;

  if (rootDoesHavePassiveEffects) {
    // This commit has passive effects. Stash a reference to them. But don't
    // schedule a callback until after flushing layout work.
    rootDoesHavePassiveEffects = false;
    rootWithPendingPassiveEffects = root;
    pendingPassiveEffectsLanes = lanes;
    pendingPassiveEffectsRenderPriority = renderPriorityLevel;
  } else {
    // We are done with the effect chain at this point so let's clear the
    // nextEffect pointers to assist with GC. If we have passive effects, we'll
    // clear this in flushPassiveEffects.
    nextEffect = firstEffect;
    while (nextEffect !== null) {
      const nextNextEffect = nextEffect.nextEffect;
      nextEffect.nextEffect = null;
      if (nextEffect.effectTag & Deletion) {
        detachFiberAfterEffects(nextEffect);
      }
      nextEffect = nextNextEffect;
    }
  }

  // Read this again, since an effect might have updated it
  remainingLanes = root.pendingLanes;

  // Check if there's remaining work on this root
  if (remainingLanes !== NoLanes) {
    if (enableSchedulerTracing) {
      if (spawnedWorkDuringRender !== null) {
        const expirationTimes = spawnedWorkDuringRender;
        spawnedWorkDuringRender = null;
        for (let i = 0; i < expirationTimes.length; i++) {
          scheduleInteractions(
            root,
            expirationTimes[i],
            root.memoizedInteractions
          );
        }
      }
      schedulePendingInteractions(root, remainingLanes);
    }
  } else {
    // If there's no remaining work, we can clear the set of already failed
    // error boundaries.
    legacyErrorBoundariesThatAlreadyFailed = null;
  }

  if (enableSchedulerTracing) {
    if (!rootDidHavePassiveEffects) {
      // If there are no passive effects, then we can complete the pending interactions.
      // Otherwise, we'll wait until after the passive effects are flushed.
      // Wait to do this until after remaining work has been scheduled,
      // so that we don't prematurely signal complete for interactions when there's e.g. hidden work.
      finishPendingInteractions(root, lanes);
    }
  }

  if (remainingLanes === SyncLane) {
    // Count the number of times the root synchronously re-renders without
    // finishing. If there are too many, it indicates an infinite update loop.
    if (root === rootWithNestedUpdates) {
      nestedUpdateCount++;
    } else {
      nestedUpdateCount = 0;
      rootWithNestedUpdates = root;
    }
  } else {
    nestedUpdateCount = 0;
  }

  onCommitRootDevTools(finishedWork.stateNode, renderPriorityLevel);

  if (__DEV__) {
    onCommitRootTestSelector();
  }

  // Always call this before exiting `commitRoot`, to ensure that any
  // additional work on this root is scheduled.
  /*
   * 每次commit阶段完成后，再执行一遍ensureRootIsScheduled，确保是否还有任务需要被调度。
   * 例如，高优先级插队的更新完成后，最后commit掉，还会再执行一遍，保证之前跳过的低优先级任务
   * 重新执行
   *
   * */
  ensureRootIsScheduled(root, now());

  if (hasUncaughtError) {
    hasUncaughtError = false;
    const error = firstUncaughtError;
    firstUncaughtError = null;
    throw error;
  }

  if ((executionContext & LegacyUnbatchedContext) !== NoContext) {
    // This is a legacy edge case. We just committed the initial mount of
    // a ReactDOM.render-ed root inside of batchedUpdates. The commit fired
    // synchronously, but layout updates should be deferred until the end
    // of the batch.
    return null;
  }

  // If layout work was scheduled, flush it now.
  flushSyncCallbackQueue();

  return null;
}
```

##### Before Mutation

###### commitBeforeMutationEffects

`location: v16.13.1/react-reconciler/src/ReactFiberWorkLoop.js`

```javascript
// The first sub-phase of Commit
// Call the getSnapshotBeforeUpdate life cycle function of the class component
function commitBeforeMutationEffects() {
  while (nextEffect !== null) {
    const current = nextEffect.alternate;

    if (!shouldFireAfterActiveInstanceBlur && focusedInstanceHandle !== null) {
      if ((nextEffect.effectTag & Deletion) !== NoEffect) {
        if (doesFiberContain(nextEffect, focusedInstanceHandle)) {
          shouldFireAfterActiveInstanceBlur = true;
          beforeActiveInstanceBlur();
        }
      } else {
        // TODO: Move this out of the hot path using a dedicated effect tag.
        if (
          nextEffect.tag === SuspenseComponent &&
          isSuspenseBoundaryBeingHidden(current, nextEffect) &&
          doesFiberContain(nextEffect, focusedInstanceHandle)
        ) {
          shouldFireAfterActiveInstanceBlur = true;
          beforeActiveInstanceBlur();
        }
      }
    }

    const effectTag = nextEffect.effectTag;
    if ((effectTag & Snapshot) !== NoEffect) {
      setCurrentDebugFiberInDEV(nextEffect);
      // 调用getSnapshotBeforeUpdate
      commitBeforeMutationEffectOnFiber(current, nextEffect);

      resetCurrentDebugFiberInDEV();
    }
    if ((effectTag & Passive) !== NoEffect) {
      // 调度useEffect
      // If there are passive effects, schedule a callback to flush at
      // the earliest opportunity.
      if (!rootDoesHavePassiveEffects) {
        rootDoesHavePassiveEffects = true;
        scheduleCallback(NormalSchedulerPriority, () => {
          flushPassiveEffects();
          return null;
        });
      }
    }
    nextEffect = nextEffect.nextEffect;
  }
}
```

###### commitBeforeMutationEffectOnFiber

`location: v16.13.1/react-reconciler/src/ReactFiberCommitWork.js`

```javascript
function commitBeforeMutationLifeCycles(
  current: Fiber | null,
  finishedWork: Fiber
): void {
  switch (finishedWork.tag) {
    case FunctionComponent:
    case ForwardRef:
    case SimpleMemoComponent:
    case Block: {
      return;
    }
    case ClassComponent: {
      if (finishedWork.effectTag & Snapshot) {
        if (current !== null) {
          const prevProps = current.memoizedProps;
          const prevState = current.memoizedState;
          const instance = finishedWork.stateNode;
          // We could update instance props and state here,
          // but instead we rely on them being set during last render.
          // TODO: revisit this when we implement resuming.
          if (__DEV__) {
            if (
              finishedWork.type === finishedWork.elementType &&
              !didWarnAboutReassigningProps
            ) {
              if (instance.props !== finishedWork.memoizedProps) {
                console.error(
                  "Expected %s props to match memoized props before " +
                    "getSnapshotBeforeUpdate. " +
                    "This might either be because of a bug in React, or because " +
                    "a component reassigns its own `this.props`. " +
                    "Please file an issue.",
                  getComponentName(finishedWork.type) || "instance"
                );
              }
              if (instance.state !== finishedWork.memoizedState) {
                console.error(
                  "Expected %s state to match memoized state before " +
                    "getSnapshotBeforeUpdate. " +
                    "This might either be because of a bug in React, or because " +
                    "a component reassigns its own `this.state`. " +
                    "Please file an issue.",
                  getComponentName(finishedWork.type) || "instance"
                );
              }
            }
          }
          const snapshot = instance.getSnapshotBeforeUpdate(
            finishedWork.elementType === finishedWork.type
              ? prevProps
              : resolveDefaultProps(finishedWork.type, prevProps),
            prevState
          );
          if (__DEV__) {
            const didWarnSet =
              ((didWarnAboutUndefinedSnapshotBeforeUpdate: any): Set<mixed>);
            if (snapshot === undefined && !didWarnSet.has(finishedWork.type)) {
              didWarnSet.add(finishedWork.type);
              console.error(
                "%s.getSnapshotBeforeUpdate(): A snapshot value (or null) " +
                  "must be returned. You have returned undefined.",
                getComponentName(finishedWork.type)
              );
            }
          }
          instance.__reactInternalSnapshotBeforeUpdate = snapshot;
        }
      }
      return;
    }
    case HostRoot: {
      if (supportsMutation) {
        if (finishedWork.effectTag & Snapshot) {
          const root = finishedWork.stateNode;
          clearContainer(root.containerInfo);
        }
      }
      return;
    }
    case HostComponent:
    case HostText:
    case HostPortal:
    case IncompleteClassComponent:
      // Nothing to do for these component types
      return;
  }
  invariant(
    false,
    "This unit of work tag should not have side-effects. This error is " +
      "likely caused by a bug in React. Please file an issue."
  );
}
```

##### Mutation

###### commitMutationEffects

`location: v16.13.1/react-reconciler/src/ReactFiberWorkLoop.js`

```javascript
// The second sub-stage of Commit
// Perform DOM operations based on effectTag
function commitMutationEffects(root: FiberRoot, renderPriorityLevel) {
  // TODO: Should probably move the bulk of this function to commitWork.
  while (nextEffect !== null) {
    setCurrentDebugFiberInDEV(nextEffect);

    const effectTag = nextEffect.effectTag;

    if (effectTag & ContentReset) {
      commitResetTextContent(nextEffect);
    }

    if (effectTag & Ref) {
      const current = nextEffect.alternate;
      if (current !== null) {
        commitDetachRef(current);
      }
    }

    // The following switch statement is only concerned about placement,
    // updates, and deletions. To avoid needing to add a case for every possible
    // bitmap value, we remove the secondary effects from the effect tag and
    // switch on that value.
    const primaryEffectTag =
      effectTag & (Placement | Update | Deletion | Hydrating);
    switch (primaryEffectTag) {
      case Placement: {
        commitPlacement(nextEffect);
        // Clear the "placement" from effect tag so that we know that this is
        // inserted, before any life-cycles like componentDidMount gets called.
        // TODO: findDOMNode doesn't rely on this any more but isMounted does
        // and isMounted is deprecated anyway so we should be able to kill this.
        nextEffect.effectTag &= ~Placement;
        break;
      }
      case PlacementAndUpdate: {
        // Placement
        commitPlacement(nextEffect);
        // Clear the "placement" from effect tag so that we know that this is
        // inserted, before any life-cycles like componentDidMount gets called.
        nextEffect.effectTag &= ~Placement;

        // Update
        const current = nextEffect.alternate;
        commitWork(current, nextEffect);
        break;
      }
      case Hydrating: {
        nextEffect.effectTag &= ~Hydrating;
        break;
      }
      case HydratingAndUpdate: {
        nextEffect.effectTag &= ~Hydrating;

        // Update
        const current = nextEffect.alternate;
        commitWork(current, nextEffect);
        break;
      }
      case Update: {
        const current = nextEffect.alternate;
        commitWork(current, nextEffect);
        break;
      }
      case Deletion: {
        commitDeletion(root, nextEffect, renderPriorityLevel);
        break;
      }
    }

    resetCurrentDebugFiberInDEV();
    nextEffect = nextEffect.nextEffect;
  }
}
```

###### commitPlacement

`location: v16.13.1/react-reconciler/src/ReactFiberCommitWork.js`

```javascript
// 插入DOM
function commitPlacement(finishedWork: Fiber): void {
  if (!supportsMutation) {
    return;
  }

  // Recursively insert all host nodes into the parent.
  // 往上找到最近的有dom节点的fiber，作为parentFiber
  const parentFiber = getHostParentFiber(finishedWork);

  // Note: these two variables *must* always be updated together.
  let parent;
  let isContainer;
  const parentStateNode = parentFiber.stateNode;
  switch (parentFiber.tag) {
    case HostComponent:
      parent = parentStateNode;
      isContainer = false;
      break;
    case HostRoot:
      parent = parentStateNode.containerInfo;
      isContainer = true;
      break;
    case HostPortal:
      parent = parentStateNode.containerInfo;
      isContainer = true;
      break;
    case FundamentalComponent:
      if (enableFundamentalAPI) {
        parent = parentStateNode.instance;
        isContainer = false;
      }
    // eslint-disable-next-line-no-fallthrough
    default:
      invariant(
        false,
        "Invalid host parent fiber. This error is likely caused by a bug " +
          "in React. Please file an issue."
      );
  }
  if (parentFiber.effectTag & ContentReset) {
    // Reset the text content of the parent before doing any insertions
    resetTextContent(parent);
    // Clear ContentReset from the effect tag
    parentFiber.effectTag &= ~ContentReset;
  }

  const before = getHostSibling(finishedWork);
  // We only have the top Fiber that was inserted but we need to recurse down its
  // children to find all the terminal nodes.
  if (isContainer) {
    insertOrAppendPlacementNodeIntoContainer(finishedWork, before, parent);
  } else {
    insertOrAppendPlacementNode(finishedWork, before, parent);
  }
}
```

###### insertOrAppendPlacementNodeIntoContainer

`location: v16.13.1/react-reconciler/src/ReactFiberCommitWork.js`

```javascript
function insertOrAppendPlacementNodeIntoContainer(
  node: Fiber,
  before: ?Instance,
  parent: Container
): void {
  const { tag } = node;
  const isHost = tag === HostComponent || tag === HostText;
  if (isHost || (enableFundamentalAPI && tag === FundamentalComponent)) {
    const stateNode = isHost ? node.stateNode : node.stateNode.instance;
    if (before) {
      insertInContainerBefore(parent, stateNode, before);
    } else {
      appendChildToContainer(parent, stateNode);
    }
  } else if (tag === HostPortal) {
    // If the insertion itself is a portal, then we don't want to traverse
    // down its children. Instead, we'll get insertions from each child in
    // the portal directly.
  } else {
    const child = node.child;
    if (child !== null) {
      insertOrAppendPlacementNodeIntoContainer(child, before, parent);
      let sibling = child.sibling;
      while (sibling !== null) {
        insertOrAppendPlacementNodeIntoContainer(sibling, before, parent);
        sibling = sibling.sibling;
      }
    }
  }
}
```

###### insertInContainerBefore

`location: v16.13.1/react-dom/src/client/ReactDOMHostConfig.js`

```javascript
export function insertInContainerBefore(
  container: Container,
  child: Instance | TextInstance,
  beforeChild: Instance | TextInstance | SuspenseInstance
): void {
  if (container.nodeType === COMMENT_NODE) {
    (container.parentNode: any).insertBefore(child, beforeChild);
  } else {
    container.insertBefore(child, beforeChild);
  }
}
```

###### appendChildToContainer

`location: v16.13.1/react-dom/src/client/ReactDOMHostConfig.js`

```javascript
export function appendChildToContainer(
  container: Container,
  child: Instance | TextInstance
): void {
  let parentNode;
  if (container.nodeType === COMMENT_NODE) {
    parentNode = (container.parentNode: any);
    parentNode.insertBefore(child, container);
  } else {
    parentNode = container;
    parentNode.appendChild(child);
  }
  // This container might be used for a portal.
  // If something inside a portal is clicked, that click should bubble
  // through the React tree. However, on Mobile Safari the click would
  // never bubble through the *DOM* tree unless an ancestor with onclick
  // event exists. So we wouldn't see it and dispatch it.
  // This is why we ensure that non React root containers have inline onclick
  // defined.
  // https://github.com/facebook/react/issues/11918
  const reactRootContainer = container._reactRootContainer;
  if (
    (reactRootContainer === null || reactRootContainer === undefined) &&
    parentNode.onclick === null
  ) {
    // TODO: This cast may not be sound for SVG, MathML or custom elements.
    trapClickOnNonInteractiveElement(((parentNode: any): HTMLElement));
  }
}
```

##### Layout

###### commitLayoutEffects

`location: v16.13.1/react-reconciler/src/ReactFiberWorkLoop.js`

```javascript
/**
 * The third sub-stage of Commit
 * 1. Handle life cycle functions of class components
 * 2. Handling hook functions for function components
 * 3. Execute the callback function after rendering is completed, if any
 */
function commitLayoutEffects(root: FiberRoot, committedLanes: Lanes) {
  // TODO: Should probably move the bulk of this function to commitWork.
  while (nextEffect !== null) {
    setCurrentDebugFiberInDEV(nextEffect);

    const effectTag = nextEffect.effectTag;

    if (effectTag & (Update | Callback)) {
      const current = nextEffect.alternate;
      commitLayoutEffectOnFiber(root, current, nextEffect, committedLanes);
    }

    if (effectTag & Ref) {
      commitAttachRef(nextEffect);
    }

    resetCurrentDebugFiberInDEV();
    nextEffect = nextEffect.nextEffect;
  }
}
```

###### commitLayoutEffectOnFiber

`location: v16.13.1/react-reconciler/src/ReactFiberCommitWork.js`

```javascript
function commitLifeCycles(
  finishedRoot: FiberRoot,
  current: Fiber | null,
  finishedWork: Fiber,
  committedLanes: Lanes
): void {
  switch (finishedWork.tag) {
    case FunctionComponent:
    case ForwardRef:
    case SimpleMemoComponent:
    case Block: {
      // At this point layout effects have already been destroyed (during mutation phase).
      // This is done to prevent sibling component effects from interfering with each other,
      // e.g. a destroy function in one component should never override a ref set
      // by a create function in another component during the same commit.
      if (
        enableProfilerTimer &&
        enableProfilerCommitHooks &&
        finishedWork.mode & ProfileMode
      ) {
        try {
          startLayoutEffectTimer();
          commitHookEffectListMount(HookLayout | HookHasEffect, finishedWork);
        } finally {
          recordLayoutEffectDuration(finishedWork);
        }
      } else {
        commitHookEffectListMount(HookLayout | HookHasEffect, finishedWork);
      }

      schedulePassiveEffects(finishedWork);
      return;
    }
    case ClassComponent: {
      const instance = finishedWork.stateNode;
      if (finishedWork.effectTag & Update) {
        if (current === null) {
          // We could update instance props and state here,
          // but instead we rely on them being set during last render.
          // TODO: revisit this when we implement resuming.
          if (__DEV__) {
            if (
              finishedWork.type === finishedWork.elementType &&
              !didWarnAboutReassigningProps
            ) {
              if (instance.props !== finishedWork.memoizedProps) {
                console.error(
                  "Expected %s props to match memoized props before " +
                    "componentDidMount. " +
                    "This might either be because of a bug in React, or because " +
                    "a component reassigns its own `this.props`. " +
                    "Please file an issue.",
                  getComponentName(finishedWork.type) || "instance"
                );
              }
              if (instance.state !== finishedWork.memoizedState) {
                console.error(
                  "Expected %s state to match memoized state before " +
                    "componentDidMount. " +
                    "This might either be because of a bug in React, or because " +
                    "a component reassigns its own `this.state`. " +
                    "Please file an issue.",
                  getComponentName(finishedWork.type) || "instance"
                );
              }
            }
          }
          if (
            enableProfilerTimer &&
            enableProfilerCommitHooks &&
            finishedWork.mode & ProfileMode
          ) {
            try {
              startLayoutEffectTimer();
              instance.componentDidMount();
            } finally {
              recordLayoutEffectDuration(finishedWork);
            }
          } else {
            instance.componentDidMount();
          }
        } else {
          const prevProps =
            finishedWork.elementType === finishedWork.type
              ? current.memoizedProps
              : resolveDefaultProps(finishedWork.type, current.memoizedProps);
          const prevState = current.memoizedState;
          // We could update instance props and state here,
          // but instead we rely on them being set during last render.
          // TODO: revisit this when we implement resuming.
          if (__DEV__) {
            if (
              finishedWork.type === finishedWork.elementType &&
              !didWarnAboutReassigningProps
            ) {
              if (instance.props !== finishedWork.memoizedProps) {
                console.error(
                  "Expected %s props to match memoized props before " +
                    "componentDidUpdate. " +
                    "This might either be because of a bug in React, or because " +
                    "a component reassigns its own `this.props`. " +
                    "Please file an issue.",
                  getComponentName(finishedWork.type) || "instance"
                );
              }
              if (instance.state !== finishedWork.memoizedState) {
                console.error(
                  "Expected %s state to match memoized state before " +
                    "componentDidUpdate. " +
                    "This might either be because of a bug in React, or because " +
                    "a component reassigns its own `this.state`. " +
                    "Please file an issue.",
                  getComponentName(finishedWork.type) || "instance"
                );
              }
            }
          }
          if (
            enableProfilerTimer &&
            enableProfilerCommitHooks &&
            finishedWork.mode & ProfileMode
          ) {
            try {
              startLayoutEffectTimer();
              instance.componentDidUpdate(
                prevProps,
                prevState,
                instance.__reactInternalSnapshotBeforeUpdate
              );
            } finally {
              recordLayoutEffectDuration(finishedWork);
            }
          } else {
            instance.componentDidUpdate(
              prevProps,
              prevState,
              instance.__reactInternalSnapshotBeforeUpdate
            );
          }
        }
      }

      // TODO: I think this is now always non-null by the time it reaches the
      // commit phase. Consider removing the type check.
      const updateQueue: UpdateQueue<*> | null =
        (finishedWork.updateQueue: any);
      if (updateQueue !== null) {
        if (__DEV__) {
          if (
            finishedWork.type === finishedWork.elementType &&
            !didWarnAboutReassigningProps
          ) {
            if (instance.props !== finishedWork.memoizedProps) {
              console.error(
                "Expected %s props to match memoized props before " +
                  "processing the update queue. " +
                  "This might either be because of a bug in React, or because " +
                  "a component reassigns its own `this.props`. " +
                  "Please file an issue.",
                getComponentName(finishedWork.type) || "instance"
              );
            }
            if (instance.state !== finishedWork.memoizedState) {
              console.error(
                "Expected %s state to match memoized state before " +
                  "processing the update queue. " +
                  "This might either be because of a bug in React, or because " +
                  "a component reassigns its own `this.state`. " +
                  "Please file an issue.",
                getComponentName(finishedWork.type) || "instance"
              );
            }
          }
        }
        // We could update instance props and state here,
        // but instead we rely on them being set during last render.
        // TODO: revisit this when we implement resuming.
        commitUpdateQueue(finishedWork, updateQueue, instance);
      }
      return;
    }
    case HostRoot: {
      // TODO: I think this is now always non-null by the time it reaches the
      // commit phase. Consider removing the type check.
      const updateQueue: UpdateQueue<*> | null =
        (finishedWork.updateQueue: any);
      if (updateQueue !== null) {
        let instance = null;
        if (finishedWork.child !== null) {
          switch (finishedWork.child.tag) {
            case HostComponent:
              instance = getPublicInstance(finishedWork.child.stateNode);
              break;
            case ClassComponent:
              instance = finishedWork.child.stateNode;
              break;
          }
        }
        commitUpdateQueue(finishedWork, updateQueue, instance);
      }
      return;
    }
    case HostComponent: {
      const instance: Instance = finishedWork.stateNode;

      // Renderers may schedule work to be done after host components are mounted
      // (eg DOM renderer may schedule auto-focus for inputs and form controls).
      // These effects should only be committed when components are first mounted,
      // aka when there is no current/alternate.
      if (current === null && finishedWork.effectTag & Update) {
        const type = finishedWork.type;
        const props = finishedWork.memoizedProps;
        commitMount(instance, type, props, finishedWork);
      }

      return;
    }
    case HostText: {
      // We have no life-cycles associated with text.
      return;
    }
    case HostPortal: {
      // We have no life-cycles associated with portals.
      return;
    }
    case Profiler: {
      if (enableProfilerTimer) {
        const { onCommit, onRender } = finishedWork.memoizedProps;
        const { effectDuration } = finishedWork.stateNode;

        const commitTime = getCommitTime();

        if (typeof onRender === "function") {
          if (enableSchedulerTracing) {
            onRender(
              finishedWork.memoizedProps.id,
              current === null ? "mount" : "update",
              finishedWork.actualDuration,
              finishedWork.treeBaseDuration,
              finishedWork.actualStartTime,
              commitTime,
              finishedRoot.memoizedInteractions
            );
          } else {
            onRender(
              finishedWork.memoizedProps.id,
              current === null ? "mount" : "update",
              finishedWork.actualDuration,
              finishedWork.treeBaseDuration,
              finishedWork.actualStartTime,
              commitTime
            );
          }
        }

        if (enableProfilerCommitHooks) {
          if (typeof onCommit === "function") {
            if (enableSchedulerTracing) {
              onCommit(
                finishedWork.memoizedProps.id,
                current === null ? "mount" : "update",
                effectDuration,
                commitTime,
                finishedRoot.memoizedInteractions
              );
            } else {
              onCommit(
                finishedWork.memoizedProps.id,
                current === null ? "mount" : "update",
                effectDuration,
                commitTime
              );
            }
          }

          // Schedule a passive effect for this Profiler to call onPostCommit hooks.
          // This effect should be scheduled even if there is no onPostCommit callback for this Profiler,
          // because the effect is also where times bubble to parent Profilers.
          enqueuePendingPassiveProfilerEffect(finishedWork);

          // Propagate layout effect durations to the next nearest Profiler ancestor.
          // Do not reset these values until the next render so DevTools has a chance to read them first.
          let parentFiber = finishedWork.return;
          while (parentFiber !== null) {
            if (parentFiber.tag === Profiler) {
              const parentStateNode = parentFiber.stateNode;
              parentStateNode.effectDuration += effectDuration;
              break;
            }
            parentFiber = parentFiber.return;
          }
        }
      }
      return;
    }
    case SuspenseComponent: {
      commitSuspenseHydrationCallbacks(finishedRoot, finishedWork);
      return;
    }
    case SuspenseListComponent:
    case IncompleteClassComponent:
    case FundamentalComponent:
    case ScopeComponent:
    case OffscreenComponent:
    case LegacyHiddenComponent:
      return;
  }
  invariant(
    false,
    "This unit of work tag should not have side-effects. This error is " +
      "likely caused by a bug in React. Please file an issue."
  );
}
```

###### commitHookEffectListMount

`location: v16.13.1/react-reconciler/src/ReactFiberCommitWork.js`

```javascript
// Handling hook functions for function components
function commitHookEffectListMount(tag: number, finishedWork: Fiber) {
  const updateQueue: FunctionComponentUpdateQueue | null =
    (finishedWork.updateQueue: any);
  const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next;
    let effect = firstEffect;
    do {
      if ((effect.tag & tag) === tag) {
        // Mount
        const create = effect.create;
        // create is the first parameter of the useEffect method
        // The return value is the cleanup function
        effect.destroy = create();

        if (__DEV__) {
          const destroy = effect.destroy;
          if (destroy !== undefined && typeof destroy !== "function") {
            let addendum;
            if (destroy === null) {
              addendum =
                " You returned null. If your effect does not require clean " +
                "up, return undefined (or nothing).";
            } else if (typeof destroy.then === "function") {
              addendum =
                "\n\nIt looks like you wrote useEffect(async () => ...) or returned a Promise. " +
                "Instead, write the async function inside your effect " +
                "and call it immediately:\n\n" +
                "useEffect(() => {\n" +
                "  async function fetchData() {\n" +
                "    // You can await here\n" +
                "    const response = await MyAPI.getData(someId);\n" +
                "    // ...\n" +
                "  }\n" +
                "  fetchData();\n" +
                `}, [someId]); // Or [] if effect doesn't need props or state\n\n` +
                "Learn more about data fetching with Hooks: https://fb.me/react-hooks-data-fetching";
            } else {
              addendum = " You returned: " + destroy;
            }
            console.error(
              "An effect function must not return anything besides a function, " +
                "which is used for clean-up.%s",
              addendum
            );
          }
        }
      }
      effect = effect.next;
    } while (effect !== firstEffect);
  }
}
```

###### commitUpdateQueue

`location: v16.13.1/react-reconciler/src/ReactUpdateQueue.js`

```javascript
// Execute the callback function after rendering
export function commitUpdateQueue<State>(
  finishedWork: Fiber,
  finishedQueue: UpdateQueue<State>,
  instance: any
): void {
  // Commit the effects
  const effects = finishedQueue.effects;
  finishedQueue.effects = null;
  if (effects !== null) {
    for (let i = 0; i < effects.length; i++) {
      const effect = effects[i];
      const callback = effect.callback;
      if (callback !== null) {
        effect.callback = null;
        callCallback(callback, instance);
      }
    }
  }
}
```
