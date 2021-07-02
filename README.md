# 나만의 React 만들기
https://pomb.us/build-your-own-react/

위 블로그 보며 자신만의 리액트 만들어보기

학습 흐름에 맞추어 작성하였다.

## 0 preview
리액트의 기본 예시는 다음과 같다.

``` js
const element = <h1 title="foo">Hello</h1>;
const container = document.getElementById("root");
ReactDom.render(element, container);
```

이를 순수 자바 스크립트 함수롤 바꾸면 다음과 같다.
```js
const element = {
  type: "h1",
  props: {
    title: "foo",
    children: "Hello",
  },
};
const container = document.getElementById("root");
```

jsx는 보통 babel을 통해서 js로 트랜스파일링 된다.
js로 사용하려면 createElement로 대체 한다. 이 element는 type과 props로 이루어져있다.(사실 더 많은 것이 있지만)
type은 dom의 타입을 나타내며, 함수가 될 수도 있다.
props는 jsx의 어트리뷰트로 받은 키와 값들이다.
children은 자식으로 보통은 또 다른 엘리먼트들의 배열이다.

리액트의 render는 실제 dom을 바꾸는 함수 함수이다. js로 바꿔보자.
```js
const node = document.createElement(element.type);
node["title"] = element.props.title;

const text = document.createTextNode("");
text["nodeValue"] = element.props.children;

node.appendChild(text);
container.appendChild(node);
```

## 1. createElement
이제 우리만의 React를 만들어보자.
먼저 앞서 보았듯이 type, props, children을 갖는 element를 만드는 함수를 만들자.
```js
function createElement(type, props, ...children) {
  return {
    type,
    props: {
      ...props,
      children: children.map((child) => {
        typeof child == "object" ? child : createTextElement(child);
      }),
    },
  };
}
```
children이 원시값일 때를 처리하는 wrap 함수를 만든다.
React는 빈 children 에 원시값이나 빈 배열을 할당하지는 않지만, 우리 코드에서는 해둔다.
```js
function createTextElement(text) {
  return {
    type: "TEXT_ELEMENT",
    props: {
      nodeValue: text,
      children: [],
    },
  };
}
```

우리 라이브러리의 이름은 myReact 라고 해두자
```js
const myReact = {
  createElement,
};
```
element를 생성한다면 다음과 같을 것이다.
```js
const element = myReact.createElement(
  "div",
  { id: "foo" },
 myReact.createElement("a", null, "bar"),
 myReact.createElement("b")
);
```
만약, 우리가 JSX는 계속 쓰고 싶은데,
babel에게 React의 createElement가 아닌 우리 creataeElement를 쓰라고 하고 싶다면 다음 커멘트를 달아야한다.
```js
/** @jsx myReact.createElement */
const element = (
  <div id="foo">
    <a>bar</a>
    <b />
  </div>
);
```

## 2. render
