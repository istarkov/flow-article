# Flow Support in Recompose

## History

One month ago the official Flow library definition was landed in Recompose.
It was a long time coming, considering the original [PR](https://github.com/acdlite/recompose/pull/241) was created by @GiulioCanti a year ago.

There was one problem. I got stuck on one of simplest enhancers in the Recompose library: `withProps`.

Type inference worked, but not in a way I expected; errors were not detected or error messages were cryptic, and the readability of intersection types didn't make me happy.
The only fix was to declare almost every input--output type for each enhancer --- but I'm too lazy to do that well.
Playing with type definitions didn't give me a better result, so I gave up.

The game changer was object type spread, which landed in [Flow v0.42.0](https://github.com/facebook/flow/releases/tag/v0.42.0).

A small output type definition change to the `withProps` enhancer gave me the expected result:

```javascript
// change output type of withProps
// from `HOC<A & B, B>` to `HOC<{ ...$Exact<B>, ...A }, B>`
type EnhancedCompProps = { b: number }

const enhancer2: HOC<*, EnhancedCompProps> = compose(
  withProps(({ b }) => ({
    b: `${b}`,
  })),
  withProps(({ b }) => ({
    // $ExpectError The operand of an arithmetic operation must be a number
    c: 1 * b,
  }))
)
```

I was able to declare just the enhanced component props type definition; then all types were inferred properly, and the error was readable.

It was clear that if it were possible to make the same idea work for most Recompose enhancers, it would be easy to work with Flow and Recompose.

And thanks to Flow type inference and @GiulioCanti, for most enhancers it already worked.

## Meet Recompose + Flow

In most cases all you need to do is to declare a props type of enhanced component.
Flow will infer all the other types you need.

```javascript
/* @flow */
import React from 'react'
import { compose, defaultProps, withProps } from 'recompose'
import type { HOC } from 'recompose';

// type of Enhanced component props
type EnhancedComponentProps = {
  text?: string,
};

const baseComponent = ({ text }) => <div>{text}</div>;

const enhance:HOC<*, EnhancedComponentProps> = compose(
  defaultProps({
    text: 'world',
  }),
  withProps(({ text }) => ({
    text: `Hello ${text}`
  }))
);

const EnhancedComponent = enhance(baseComponent);

export default EnhancedComponent;
```

You don't need to provide types for arguments to enhancers; Flow will infer them automatically.

Remove `defaultProps` in the example above and you would immediately get a Flow error:

![[flow] undefined (This type cannot be coerced to string)](./error.png?raw=true)

The other wonderful feature is Flow's ability to infer types automatically,
which makes the Recompose experience even better.  See it in action:

![recompose-flow](https://user-images.githubusercontent.com/5077042/28116959-0c96ae2c-6714-11e7-930e-b1454c629908.gif)

For me this feature is much cooler than error detection, as
it allows you to read and understand the code much faster.

The magic above works for most Recompose enhancers, but not all. Any type system has its limitations, so for some enhancers you will need to provide type information for every special case (no automatic type inference), or use the following recommendations:

- `flattenProp`, `renameProp`, `renameProps` --- replace with `withProps` 

- `withReducer`, `withState` --- use `withStateHandlers`

- `lifecycle` --- write your own enhancer instead; see [this test](https://github.com/acdlite/recompose/blob/master/types/flow-typed/recompose_v0.24.x/flow_v0.53.x-/test_mapPropsStream.js) for an example

- `mapPropsStream` --- see the [test](https://github.com/acdlite/recompose/blob/master/types/flow-typed/recompose_v0.24.x/flow_v0.53.x-/test_mapPropsStream.js) for an example

For an example of how to type enhancers like this, see [this test](https://github.com/acdlite/recompose/blob/master/types/flow-typed/recompose_v0.24.x/flow_v0.53.x-/test_voodoo.js).

There is also a problem with the `mapProps` enhancer: type inference works well, but errors are not detected. See [test_mapProps](https://github.com/acdlite/recompose/blob/5c7f4d4ff2ccf1b71bb3089bca16d324d1249723/types/flow-typed/recompose_v0.24.x/flow_v0.53.x-/test_mapProps.js#L29) for details.

## Use Recompose and Flow with React class components

Sometimes you need to use Recompose with React class components. You can use the following helper to extract the property type from an enhancer:

```javascript
// Extract type from any enhancer
type HOCBase_<A, B, C: HOC<A, B>> = A
type HOCBase<C> = HOCBase_<*, *, C>

```

And use it within your component declaration:

```javascript
type MyComponentProps = HOCBase<typeof myEnhancer>

class MyComponent extends React.Component<MyComponentProps> {
  render() ...
}

const MyEnhancedComponent = myEnhancer(MyComponent)

```

## Write your own enhancers

To write your own simple enhancer
refer to the [Flow documentation](https://flow.org/en/docs/react/hoc/) and you will end up with something like:

```javascript
/* @flow */
import * as React from 'react'
import { compose, withProps } from 'recompose'
import type { HOC } from 'recompose'

function mapProps<BaseProps: {}, EnhancedProps>(
  mapperFn: EnhancedProps => BaseProps
): (React.ComponentType<BaseProps>) => React.ComponentType<EnhancedProps> {
  return Component => props => <Component {...mapperFn(props)} />
}

// Test that all is fine
type EnhancedProps = { hello: string }

const enhancer: HOC<*, EnhancedProps> = compose(
  mapProps(({ hello }) => ({
    hello: `${hello} world`,
    len: hello.length,
  })),
  withProps(props => ({
    helloAndLen: `${props.hello} ${props.len}`,
  }))
)
```

For class-based enhancers:

```javascript
/* @flow */
import * as React from 'react'
import { compose, withProps } from 'recompose'
import type { HOC } from 'recompose'
// Example of very dirty written fetcher enhancer
function fetcher<Response: {}, Base: {}>(
  dest: string,
  nullRespType: ?Response
): HOC<{ ...$Exact<Base>, data?: Response }, Base> {
  return BaseComponent =>
    class Fetcher extends React.Component<Base, { data?: Response }> {
      state = { data: undefined }
      componentDidMount() {
        fetch(dest)
          .then(r => r.json())
          .then((data: Response) => this.setState({ data }))
      }
      render() {
        return <BaseComponent {...this.props} {...this.state} />
      }
    }
}

// Test that all is fine
// Enhanced Component props type
type EnhancedCompProps = { b: number }
// response type
type FetchResponseType = { hello: string, world: number }

// Now you can use it, let's check
const enhancer: HOC<*, EnhancedCompProps> = compose(
  // pass response type via typed null
  fetcher('http://endpoint.ep', (null: ?FetchResponseType)),
  // see here fully typed data
  withProps(({ data }) => ({
    data,
  }))
)
```

Now Flow will infer the type of `data` properly, so you can safely use it.

![data-type-example](./dataExample.png?raw=true)

## Links

- A fully-typed [example project](https://github.com/acdlite/recompose/tree/master/types/flow-example) of this [menu app](https://grader-meets-16837.netlify.com/)

- Usage examples can be found in [tests](https://github.com/acdlite/recompose/tree/master/types/flow-typed/recompose_v0.24.x/flow_v0.53.x-)

## P.S.

Your move, Caleb ;-)

![twitter](./twitter.png?raw=true)
