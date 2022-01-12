# 나만의 React 만들기
https://pomb.us/build-your-own-react/

위 블로그 보며 자신만의 리액트 만들어보기

학습 흐름에 맞추어 작성하였다.

## 0. preview
리액트의 기본 예시는 다음과 같다.
``` js
const element = <h1 title="foo">Hello</h1>;
const container = document.getElementById("root");
ReactDom.render(element, container);
```

이를 순수 자바 스크립트 함수로 바꾸면 다음과 같다.
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

리액트의 render는 react element를 실제 javascript dom으로 바꾸는 함수이다.
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

children이 원시값일 때를 처리하는 wrap 함수(createTextElement)를 만든다.

참고로, 실제 React는 children에 원시값이나 빈 배열을 할당하지는 않는다.
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

만약, 우리가 JSX는 계속 쓰고 싶다면,

babel에게 React의 createElement가 아닌 우리 creataeElement를 쓰라고 다음 커멘트를 달아야한다.
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
다음은 render 함수를 만들어보자.

우선, DOM에 추가하는 코드만 만들고, 차례대로 업데이트, 삭제를 구현할 것이다.

DOM 노드를 만들고 type을 정하고, container에 추가하는 함수를 만들자.

자식들도 재귀적으로 element를 생성해줘야한다.

Text element에 대한 제외 처리도 해준다.

마지막으로, element의 children 외의 props들을 dom 노드에 추가해준다.
```js
function render(element, container) {
  const dom =
    element.type == "TEXT_ELEMENT"
      ? document.createTextNode("")
      : document.createElement(element.type);
  
  const isPropety = (key) => key !== "children";
  
  Object.keys(element.props)
    .filter(isPropety)
    .forEach((name) => {
      dom[name] = element.props[name];
    });

  element.props.children.forEach((child) => render(child, dom));
  container.appendChild(dom);
}
```

## 3. concurrent Mode
다음 코드를 작성하기 하기전에 짚어야 할게 있다.

이전 render 함수의 재귀 코드는 많은 양의 트리를 처리할 때는 오랫동안 블록을 하고 있어서 다른 작업을 할 수가 없다.
```js
  Object.keys(element.props)
    .filter(isPropety)
    .forEach((name) => {
      dom[name] = element.props[name];
    });
```

그래서, 이 작업을 작은 단위로 쪼개서 브라우저가 다른 작업이 필요하면 멈출 수 있게 하도록 할 것이다.

우리는 requestIdleCallback로 루프를 만들건데, requestIdleCallback은 브라우저가 쓰레드가 작업이 없을 때 실행시키도록 하는 함수이다. 

실제 React는 이를 더 이상 사용하지 않고 스케쥴러를 사용하지만 우리 대체해서 사용하도록한다. (자세한 내용은 concurrent 모드에 대해 알아보면 좋음)

또한, requestIdleCallback은 deadline 파라미터를 제공한다. 이를 통해 브라우저가 다신 제어권을 가져갈때까지 얼마나 남았는지 알 수 있다. 

```js
function workLoop(deadline) {
  let shouldYield = false;
  while (nextUnitOfWork && !shouldYield) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    shouldYield = deadline.timeRemaining() < 1;
  }
  requestIdleCallback(workLoop);
}
```

이 workLoop를 더 완성시켜보자.

performUnitOfWork는 다음 작업 단위를 리턴하는 함수로, 일단 todo로 남겨놓자.

```js
let nextUnitOfWork = null;

function workLoop(deadline) {
  let shouldYield = false;
  while (nextUnitOfWork && !shouldYield) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    shouldYield = deadline.timeRemaining() < 1;
  }
  requestIdleCallback(workLoop);
}

requestIdleCallback(workLoop);

function performUnitOfWork(nextUnitOfWork) {
  // todo
}
```

## 4. fibers
이 실행 단위를 관리하기 위한 자료구조가 바로 fiber이다.

element 마다 fiber를 가지게 되고, 이 fiber는 하나의 실행 단위이다.

다음과 같은 html이 있다고하자
``` js
Didact.render(
  <div>
    <h1>
      <p />
      <a />
    </h1>
    <h2 />
  </div>,
  container
)
```
render 함수는 root fiber를 만들고 첫 nextUnitOfWork로 설정할 것이다. 

이후 작업들은 performUnitOfWork에서 실행될 것이고 다음과 같다.
```
1. Dom에 element 달기
2. children에 대한 fiber 생성
3. 다음 실행 단위 설정
```

이 자료구조의 핵심은 다음 실행 단위를 쉽게 찾는 것이다.

그래서, fiber들은 첫 번째 자녀, 바로 옆 형제, 부모와의 링크를 가지고 있고 다음과 같은 규칙을 가진다.
```
자식 있다면, 자식이 다음 실행 단위가 된다.
자식이 없다면, 형제가 실행 단위가 된다.
자식, 형제가 없다면, 삼촌(형제의 부모)가 실행 단위가 된다.
```

이를 코드로 옮겨보자. 우선 기존 render 함수에서 dom을 만드는 부분을 따로 함수화한다.
```js
function createDom(fiber) {
  const dom =
    fiber.type == "TEXT_ELEMENT"
      ? document.createTextNode("")
      : document.createElement(fiber.type);

  const isPropety = (key) => key !== "children";

  Object.keys(fiber.props)
    .filter(isPropety)
    .forEach((name) => {
      dom[name] = fiber.props[name];
    });

  return dom;
}
```

render 함수는 이제 nextUnifOfWork를 root의 fiber tree로 설정한다.
```js
function render(element, container) {
  nextUnitOfWork = {
    dom: container,
    props: {
      children: [element],
    },
  };
}
```

이제, 브라우저가 준비가 되면 workLoop를 호출해 root부터 작업을 시작할 것이다.

먼저, 새 node를 만들어 dom에 붙이고, 

자식들에 대해 fiber를 생성한다.
첫 자녀는 자녀로 연결하고, 나머지들은 서로 형제로 연결한다.

마지막으로는, 다음 실행 단위를 찾고 반환한다.

```js
function performUnitOfWork(fiber) {
  if (!fiber.dom) {
    fiber.dom = createDom(fiber);
  }

  if (fiber.parent) {
    fiber.parent.dom.appendChild(fiber.dom);
  }

  const elements = fiber.props.children;
  let index = 0;
  let prevSibling = null;

  while (index < elements.lenth) {
    const element = elements[index];
    const newFiber = {
      type: element.type,
      props: element.props,
      parent: fiber,
      dom: null,
    };

    if (index == 0) {
      fiber.child = newFiber;
    } else {
      prevSibling.sibling = newFiber;
    }

    prevSibling = newFiber;
    index++;
  }

  if (fiber.child) {
    return fiber.child;
  }
  
  let nextFiber = fiber;
  
  while (nextFiber) {
    if (nextFiber.sibling) {
      return newFiber.sibling;
    }
    nextFiber = nextFiber.parent;
  }
}
```

## 5. render and commit phases
여기서 문제가 있다.

우리는 각 element에 대하여 dom에 새 node를 달아주고 있는데, 만약 모든 dom을 그리기전에 브라우저가 제어권을 뺏어간다면 rendering이 끝나기전에 불완전한 UI를 보여주고 말 것이다.

그래서 performUnitOfWork에서 dom을 매번 추가하는 부분을 지우고, fiber tree의 root를 wiproot라고 이름 짓고 이를 추적할 것이다.

```js
// performUnitOfWork 지울부분
 if (fiber.parent) {
    fiber.parent.dom.appendChild(fiber.dom);
  }
```

``` js
function render(element, container) {
  wipRoot = {
    dom: container,
    props: {
      children: [element],
    },
  };
  nextUnitOfWork = wipRoot;
}
```

다음 실행 단위가 없어, 모든 실행 단위가 끝이 나면 비로소 dom에 commit한다. 이를 commitRoot 함수에서 실행한다.

``` js
function workLoop(deadline) {
  let shouldYield = false;
  while (nextUnitOfWork && !shouldYield) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    shouldYield = deadline.timeRemaining() < 1;
  }

  if (!nextUnitOfWork && wipRoot){
    commitRoot()
  }
  
  requestIdleCallback(workLoop);
}
```

```js
function commitRoot() {
  commitWork(wipRoot.child);
  wipRoot = null;
}

function commitWork(fiber) {
  if (!fiber) {
    return;
  }
  const domParent = fiber.parent.dom;
  domParent.appendChild(fiber.dom);
  commitWork(fiber.child);
  commitWork(fiber.sibling);
}
```
## 6. reconciliation
지금까지 dom에 추가하는 기능을 구현했다. 이제는 update와 unmount을 구현해보자.

우선 마지막 추가한 마지막 fiber tree를 currentRoot로 저장한다.

또한, alternate라는 프로퍼티를 fiber에 추가해 이를 기억하게 한다.
```js
function render(element, container) {
  wipRoot = {
    dom: container,
    props: {
      children: [element],
    },
    alternate: currentRoot,
  };

  nextUnitOfWork = wipRoot;
}
```

performUnitOfWork에서 자식 fiber를 그리는 부분을 추출해서 reconcileChildren() 함수로 다시 만든다. 

이제는 자식 fiber를 그리는 동안 이전 oldFiber와 달라진 점이 없는지 체크를 할 것이다.

비교를 위해서는 타입을 사용한다. 

같은 타입이라면 기존 노드를 사용하고 props만 업데이트 해준다. 

타입이 다르고 새 element가 있다면 새로운 dom 노드를 생성한다. 

타입이 다르고 기존 fiber만 있다면 예전 node를 지워야한다. 

이 과정에서 실제 리액트는 key를 가지고 비교를 한다. (element 배열에서 효과적일 것이다.)

이를 effectTag라는 새로운 프로퍼티를 추가해 표시해 둘 것이다.

props만 업데이트 될 경우에는 UPDATE를, 새 dom 노드 추가는 PLACEMENT 표기를 해두어 commit시에 활용한다.

삭제는 DELETION list를 따로 만들어 관리한다.

```js
function reconcileChildren(wipFiber, elements) {
  let index = 0;
  let oldFiber = wipFiber.alternate && wipFiber.alternate.child;
  let prevSibling = null;

  while (index < elements.lenth || oldFiber != null) {
    const element = elements[index];
    let newFiber = null;
    const sameType = oldFiber && element && element.type == oldFiber.type;

    if (sameType) {
      newFiber = {
        type: oldFiber.type,
        props: element.props,
        dom: oldFiber.dom,
        parent: wipFiber,
        alternate: oldFiber,
        effectTag: "UPDATE",
      };
    }
    
    if (element && !sameType) {
      newFiber = {
        type: element.type,
        props: element.props,
        dom: null,
        parent: wipFiber,
        alternate: null,
        effectTag: "PLACEMENT",
      };
    }
    
    if (oldFiber && !sameType) {
      oldFiber.effectTag = "DELETION";
      deletions.push(oldFiber);
    }

    if (oldFiber) {
      oldFiber = oldFiber.sibling;
    }

    if (index == 0) {
      fiber.child = newFiber;
    } else {
      prevSibling.sibling = newFiber;
    }

    prevSibling = newFiber;
    index++;
  }
}
```

이제 commitwork 에서 effectTags에 따라 반영한다.
``` js
function commitRoot() {
  deletions.forEach(commitWork);
  commitWork(wipRoot.child);
  currentRoot = wipRoot;
  wipRoot = null;
}

function commitWork(fiber) {
  if (!fiber) {
    return;
  }
  
  const domParent = fiber.parent.dom;
  
  if (fiber.effectTag === "PLACEMENT" && fiber.dom != null) {
    domParent.appendChild(fiber.dom);
  } else if (fiber.effectTag === "UPDATE" && fiber.dom != null) {
    updateDom(fiber.dom, fiber.alternate.props, fiber.props);
  } else if (fiber.effectTag === "DELETION") {
    domParent.removeChild(fiber.dom);
  }
  
  commitWork(fiber.child);
  commitWork(fiber.sibling);
}
```
update tag에 대해서는 기존 propety들을 교체해주는 작업을 해주는 updateDom 함수를 만든다. 

이벤트 리스터 property는 "on" 으로 시작하는 특수한 케이스로 예외처리해준다.

``` js
const isEvent = (key) => key.startsWith("on");
const isProperty = (key) => key !== "children" && !isEvent(key);
const isNew = (prev, next) => (key) => prev[key] !== next[key];
const isGone = (prev, next) => (key) => !(key in next);

function updateDom(dom, prevProps, nextProps) {
  // remove old or changed event listeners
  Object.keys(prevProps)
    .filter(isEvent)
    .filter((key) => {
      !(key in nextProps) || isNew(prevProps, nextProps)(key);
    })
    .forEach((name) => {
      const eventType = name.toLowerCase().substring(2);
      dom.removeEventListener(eventType, prevProps[name]);
    });

  // remove old properties
  Object.keys(prevProps)
    .filter(isProperty)
    .filter(isGone(prevProps, nextProps))
    .forEach((name) => {
      dom[name] = "";
    });

  // add new properties
  Object.keys(nextProps)
    .filter(isProperty)
    .filter(isNew(prevProps, nextProps))
    .forEach((name) => {
      dom[name] = nextProps[name];
    });

  // add new event listeners
  Object.keys(prevProps)
    .filter(isEvent)
    .filter(isNew(prevProps, nextProps))
    .forEach((name) => {
      const eventType = name.toLowerCase().substring(2);
      dom.addEventListener(eventType, prevProps[name]);
    });
}
```

## 7. function components
다음 추가할 기능은 함수형 컴포넌트이다.

함수형 컴포넌트의 차이점은, fiber가 dom node(container)를 갖지 않는다는 점이고, childeren props에서 가져오는 것이 아니라 함수에서 리턴한다는 점이다.

우리는 fiber의 타입을 체크해서 함수라면 다른 update 함수를 사용하도록 할 것이다. 

updateHostComponent는 기존 함수와 같고, updateFunctionComponent는 함수로 자식을 가져오도록 할것이다.

```js
function performUnitOfWork(fiber) {
  const isFunctionComponent = fiber.type instanceof Function;
  if (isFunctionComponent) {
    updateFunctionComponent(fiber);
  } else {
    updateHostComponent(fiber);
  }

  if (fiber.child) {
    return fiber.child;
  }
  
  let nextFiber = fiber;
  
  while (nextFiber) {
    if (nextFiber.sibling) {
      return newFiber.sibling;
    }
    nextFiber = nextFiber.parent;
  }
}

function updateFunctionComponent(fiber) {
  const children = [fiber.type(fiber.props)];
  reconcileChildren(fiber, children);
}

function updateHostComponent(fiber) {
  if (!fiber.dom) {
    fiber.dom = createDom(fiber);
  }
  reconcileChildren(fiber, fiber.props.children);
}
```

commitWork에서 바꿔야할 부분이 있다. 부모를 찾는 부분에서는 dom 노드를 가진 fiber를 찾을때까지 찾도록 하는 것이다.

삭제하는 부분도 dom 노들를 가진 자식을 찾을때까지 찾도록 해야 한다.

```js
function commitWork(fiber) {
  if (!fiber) {
    return;
  }

  let domParentFiber = fiber.parent;
  while (!domParentFiber.dom) {
    domParentFiber = domParentFiber.parent;
  }
  const domParent = domParentFiber;

  if (fiber.effectTag === "PLACEMENT" && fiber.dom != null) {
    domParent.appendChild(fiber.dom);
  } else if (fiber.effectTag === "UPDATE" && fiber.dom != null) {
    updateDom(fiber.dom, fiber.alternate.props, fiber.props);
  } else if (fiber.effectTag === "DELETION") {
    commitDeletion(fiber, domParent);
  }
  commitWork(fiber.child);
  commitWork(fiber.sibling);
}

function commitDeletion(fiber, domParent) {
  if (fiber.dom) {
    domParent.removeChild(fiber.dom);
  } else {
    commitDeletion(fiber.child, domParent);
  }
}
```
## 8. hooks
마지막으로 함수형 컴포넌트에 state를 추가하자.

먼저 useState에 필요한 전역변수를 선언한다.

useState가 한 컴포넌트에서 여러번 호출되었을 때를 위해 hooks 배열도 만든다.

```js
let wipFiber = null
let hookIndex = null

function updateFunctionComponent(fiber) {
  wipFiber = fiber
  hookIndex = 0
  wipFiber.hooks = []
  const children = [fiber.type(fiber.props)];
  reconcileChildren(fiber, children);
}
```

useState가 호출되면, 이전에 호출된 hook이 있는지 확인한다. 

hook이 있다면 이전 hook에서 state를 복사해 온다. 

새로운 hook을 fiber에 넣어주고 state를 반환한다.

useState는 상태를 변환시키기 위한 함수를 반환해야한다. 

그래서 setState를 선언해 action을 받도록 선언해야하고, 이 action을 hook에 큐에 삽입한다. 

그 다음은 마치 render가 작동하듯이 차근차근 다음 실행단위를 실행시키도록 한다.

이 action들은 우리가 다음 랜더링을 할 때 old hook 에 쌓여있던 action들이 실행되며 실행된다. 
```js
function useState(initial) {
  const oldHook =
    webfiber.alternate &&
    webFiber.alternate.hooks &&
    webFiber.alternate.hooks[hookIndex];

  const hook = { status: oldHook ? oldHook.state : initial, queue: [] };

  const action = oldHook ? oldHook.queue : []
  action.forEach(action => {
    hook.state = action(hook.state)
  })

  const setState = (action) => {
    hook.queue.push(action);
    wipRoot = {
      dom: currnetRoot.Dom,
      props: currnetRoot.props,
      alternate: currnetRoot,
    };
    (nextUnitOfWork = wipRoot), (deletions = []);
  };

  wipFiber.hooks.push(hook);
  hookIndex++;
  return [hook.state, setState];
}
```
