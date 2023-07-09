---  
share: true  
toc: true  
categories: [Weekly Frontends]  
tags: [frontend, browser, qanda, svg, typescript]  
layout: post  
title: "Typescript as a Set, SVG, and Modules"  
date: "2023-07-07"  
github_title: "2023-07-07-Typescript as a Set, SVG, and Modules"  
---  
  
## TL; DR  
QANDA Adaptive Content Frontend Part에서 주간 회고 시간에 각자 읽고 싶었지만 시간이 없어서 읽지 못했던 아티클들을 모아서 읽고 서로 공유한 내용들을 정리합니다.   
  
## Typescript as a Set  
  
![center](Pasted%20image%2020230709091639.png)  
  
타입스크립트는 튜링 완전하며, 타입 시간에 평가(evaluate)되는 코드이다. (여기서 타입 시간이란, IDE에서 타이핑하는 시간을 의미한다. 즉 IDE를 통해서 Read-Eval-Print-Loop을 수행한다)  
  
Typescript에서 공리계(Axiom)에 속하는 가장 중요한 공리 중 하나는 *타입은 집합이다* 라는 것이다. 이 공리를 이해해야 다음과 같은 예제를 혼동하지 않고 타이핑할 수 있다.  
  
```typescript  
type A = { a: number }  
type B = { b: number }  
  
type C = A | B  
const c1: C = { a: 1}  
const c2: C = { b: 1}  
const c3: C = { a: 1, b:2 }  
  
type D = A & B  
const d1: D = { a: 1, b: 2}  
const d2: D = { a: 1 } // Type Error!  
  
```  
  
C의 경우, A와 B의 합집합으로 정의되었으므로 a를 가지고 있거나 b를 가지고 있으면 된다. 따라서 c1, c2, c3 모두 valid하다. 반면, D의 경우 A와 B의 교집합으로 정의되었으므로 a를 가지고 있으면서 b를 가지고 있는 집합으로 정의된다. 따라서 a만 가지고 있는 d2는 type error가 발생한다.  
  
![](Pasted%20image%2020230709092643.png)  
  
한편, unknown은 전체 집합을 의미하며, never는 공집합을 의미한다. 따라서 집합의 관점으로 해석해보면 unknown과 교집합하는 모든 집합은 자기 자신이 되며, never와 합집합하는 모든 집합도 자기 자신이 된다. 아래는 이를 사용해서 타이핑한 예시가 되겠다.  
  
```typescript  
export type Message<MODE extends 'input' | 'output',  
C, // common  
O = unknown, // output only  
I = unknown // input only  
> = C & (MODE extends 'output' ? O : I);  
  
export type OneOf<O1, O2, O3 = never, O4 = never> = O1 | O2 | O3 | O4;  
```  
  
### Chain of Thoughts  
- Infer Typing  
- Recursive Conditional Typing  
- Distributive Conditional Typing  
  
## Generic and Mapped Types  
  
### keyof Operator  
`keyof` operator는 타입스크립트에서 제공하는 함수이며,  `Object.keys`는 자바스크립트에서 제공하는 함수이다. 이를 사용해서 다음과 같이 특정 객체에 대한 엄격한 Property 검사를 수행할 수 있다.   
  
한 가지 흥미로운 점은 email, loc를 추론하는 시점에 User 객체의 프로퍼티가 완전히 정해지므로 (동적으로 바뀌지 않고, 추론 가능함.) `getObjectProps`에 명시적으로 제네릭 값을 넣어주지 않더라도 이를 추론할 수 있다는 점이다. (Typescript structural subtyping의 특징이라 하겠다.)  
  
>The keyof operator takes an object type and produces a string or numeric literal union of its keys. - Typescript Handbook  
  
```typescript  
function getObjectProps<Obj, Key extends keyof Obj>(obj: Obj, key: Key): Obj[key] {  
	return obj[key]  
}  
  
const User = {  
	name: 'Matias',  
	site: 'matiashernandez.dev',  
	location: {  
		country: 'Chile',  
		city: 'Talca'  
	}  
}  
  
const email = getObjectProps(User, 'email') //Argument of type '"email"' is not assignable to parameter of type '"name" | "site" | "location"'.  
const loc = getObjectProps(User,'location')  
```  
  
  
  
### Generic에서의 Extends  
interface에서의 extends와는 다르게 Typescript Generic에서의 `extends` 키워드는 스코프를 좁혀서 이 타입이 해당 타입에 "속하는가"를 판단하기 위해서 사용한다. 아래의 예시에서(react-query의 타이핑) TQueryKey는 아무 값이나 할당될 수 없고, QueryKey에 속하는(즉, 위에서 설명한 집합의 관점에서 QueryKey의 Subset에 해당되는) 타입만 할당될 수 있다는 의미이다.   
   
``` typescript  
export type QueryFunction<T = unknown, TQueryKey extends QueryKey = QueryKey> = (context: QueryFunctionContext<TQueryKey>) => T | Promise<T>  
```  
  
>Another usage of the `extends` keyword is to **narrow the scope** of a generic to make it more useful.  
>  
>This usage of `extends` _narrowing down or constraining the type of a generic_ is the corner stone to be able to implement conditional types since the `extends` will be used as condition to check if a generic is or not of a certain type.  
  
  
## SVG in JS  
  
결론적으로 SVG를 JSX로 렌더링하는 것은 매우 비싼 작업이므로 SVG는 img 태그의 src를 사용해서 렌더링하도록 하자! 가 이 글의 요지라 할 수 있겠다.  
  
>SVGs in JS have a cost and SVGs do not belong into your JS bundle. It’s about time to bring back ==SVG-in-HTML.==  Let’s take a look at better techniques for using SVG in JSX, while keeping our JS bundle small and performant.  
  
### Parsing & Compilation  
  
![](Pasted%20image%2020230709102317.png)  
  
SVG를 React Component를 사용해서 렌더링하겠다는 것은 `svgr` 같은 번들러를 사용해서 JS Bundle에 이를 포함시키겠다는 것을 의미한다. 즉, 브라우저가 바로 html 태그를 해석하는 것 보다 더 많은 절차 (파싱, 컴파일)를 밟아야 한다는 것을 의미하며, 이는 당연히 이미지를 바로 렌더링하는것 보다 비싸다.  
  
> Byte-for-byte, JavaScript is **more expensive** for the browser to process **than** the   
> **equivalently** sized **image** or Web Font (web.dev)  
  
>Parsing & Compilation is the step right before the execution. So whenever your JavaScript is downloaded and ready to run, the **time** it takes **to parse** and the time it takes **to compile**, is the **time where the user is waiting for the interactivity**.  
  
따라서 몇몇 특수한 경우들을 제외하고는 img 태그를 사용해서 svg-in-js를 가급적 피하는 것이 좋다. 주목할 만한 내용은 일반적으로 사용한다고 느껴지는 좋은 사양의 iPhone에서는 큰 문제가 없을 수 있지만, p75 user들이 사용하는 기기(즉, 절대 다수의 사용자들이 사용하는 기기를 대표하는 값)는 A50 정도의 수준이기 때문에 이 차이가 생각보다 클 수 있다는 점이다. 특히 글로벌 웹뷰나 웹 앱을 만드는 프로젝트에서는 꽤 유의미한 성능차이를 낼 수 있으니 이를 감안하는 것을 중요해보인다.  
  
>On a fast M2 laptop, the difference might not be obvious, but as [Alex Russell](https://infrequently.org/) from the Microsoft Edge team reports year-by-year, there is a [performance equality gap](https://infrequently.org/2022/12/performance-baseline-2023/). As he puts it, a Samsung Galaxy A50 and the Nokia G11 are the best devices you can buy to understand world-wide p75 users. ==Web development should be inclusive for everyone==, and not only for wealthy regions.  
  
  
(React 18의 Server Component를 사용하면 브라우저가 Javascript를 해석하는 것이 아니기 때문에 비교적 괜찮다. 하지만 drawback이 있을 수 있으니 확인이 필요하긴 하다)  
  
>Very recent, but also a viable solution that does not require too many changes, would be adopting the upcoming Server Components from React, which can be used in e.g. [NextJS 13.4](https://nextjs.org/blog/next-13-4). They are especially helpful when you need to change behaviour of components at runtime and allow you to write JSX that is only executed on the server. This way, the SVGs are not part of the JavaScript bundle shipped to browsers anymore. In order to keep them on the server only, it is as simple as _not_ adding `'use client'` to the file.  
  
  
  
## Reference  
  
- [You Dont Know Type](https://velog.io/@gomjellie/You-dont-know-type)  
- [Generic and Mapped Types](https://egghead.io/blog/learn-the-key-concepts-of-typescript-s-powerful-generic-and-mapped-types)  
- [SVG in JS](https://kurtextrem.de/posts/svg-in-js)  
- [CJS & ESM](https://toss.tech/article/commonjs-esm-exports-field)