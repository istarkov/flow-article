# Flow support in recompose

## History

One month ago official flow library definition was landed in recompose.
It was a long way before we added it having that [PR](https://github.com/acdlite/recompose/pull/241) was added by @GiulioCanti one year ago.

There was one problem, I got stuck with one of simplest enhancers in recompose library `withProps`.

Type inference had worked but not in a way I expected, errors were not detected or were cryptic, readability of intersection types also didn't make me happy.
The only way was to declare almost every input-output type for each enhancer but I'm too lazy to do it well.
Playing with type definitions didn't give me a better result so I gave up.

The game changer was object type spread landed in [flow v0.42.0](https://github.com/facebook/flow/releases/tag/v0.42.0)

A small output type definition change of `withProps` enhacer gave me an expected result

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

I was able to declare just Enhanced component props type definition, all types were infered properly and error was readable.

It was clear that if it would be possible to make same idea to work for most recompose enhancers there will be really easy to work with flow and recompose.

And thanks to flow type inference and @GiulioCanti, for most enhancers it already worked.

## Meet recompose + flow

In most cases all you need is to declare a props type of enhanced Component.
Flow will infer all other types you need.

```javascript
import type { HOC } from 'recompose';

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

You don't need to provide types for enhancers arguments, flow will infer them automatically.

Just remove `defaultProps` in the example above and you would immediately get an error inside `withProps`
`[flow] undefined (This type cannot be coerced to string)`

The other wonderful feature is the ability of flow to infer types automatically,
this makes expirience with recompose even more better than ever.

See this in action.

![recompose-flow](https://user-images.githubusercontent.com/5077042/28116959-0c96ae2c-6714-11e7-930e-b1454c629908.gif)

If you want to play with this there is a fully typed [example project](https://github.com/acdlite/recompose/tree/master/types/flow-example) of this [menu app](https://grader-meets-16837.netlify.com/)

A lot of usefull information you can also get from  [tests](https://github.com/acdlite/recompose/tree/master/types/flow-typed/recompose_v0.24.x/flow_v0.53.x-)


Sometimes it's needed to use recompose with React Class Components. You can use following helper to extract property type from enhancer.

```javascript
// Extract type from any enhancer
type HOCBase_<A, B, C: HOC<A, B>> = A
type HOCBase<C> = HOCBase_<*, *, C>

```

and use it within your Component declaration

```javascript
type MyComponentProps = HOCBase<typeof myEnhancer>

class MyComponent extends React.Component<MyComponentProps> {
  render() ...
}

const MyEnhancedComponent = myEnhancer(MyComponent)

```

What if you need to write your own enhancer.
For simple enhancers just see [flow-documentation](https://flow.org/en/docs/react/hoc/)

For class based enhacers see the example below.

```javascript
// Example of very dirty written fetcher enhancer
function fetcher<Response: {}, Base: {}>(
  dest: string,
  obj: ?Response
): HOC<{ ...$Exact<Base>, data: Response }, Base> {
  return BaseComponent =>
    class Fetcher extends React.Component<Base, { data: Response }> {
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

// Enhanced Component props type
type EnhancedCompProps = { b: number }

// response type
type FetchResponseType = { hello: string }

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


















