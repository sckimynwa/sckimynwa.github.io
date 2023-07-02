---  
share: true  
toc: true  
categories: [Weekly Frontends]  
tags: [frontend, browser, qanda, hook, typescript]  
layout: post  
title: "Browser rendering, Typescript structural typing"  
date: "2023-07-02"  
github_title: "2023-07-02-Browser rendering, Typescript structural typing"  
image:  
  path: /assets/img/Pasted%20image%2020230702094411.png  
---  
  
## TL; DR  
QANDA Adaptive Content Frontend Part에서 주간 회고 시간에 각자 읽고 싶었지만 시간이 없어서 읽지 못했던 아티클들을 모아서 읽고 서로 공유한 내용들을 정리합니다.   
  
  
## How Browser Works  
브라우저에 대해 굉장히 많은 부분들을 상세하게 커버하고 있는 글이라 짧은 시간안에 읽기가 쉽지 않았다. 그래서 흥미로운 부분들 위주로 읽었다.  
  
### Context Free Grammar  
지난학기에 Database 수업을 들으면서 SQL Parser를 구현한 적이 있었다. 다음과 같은 SQL 문법을 DBMS는 어떻게 처리하는가?에 대해서 EBNF(Extended Backus-Naur Form)의 문법을 사용해서 언어의 문법을 기술하고 이를 [Lark](https://github.com/lark-parser/lark/blob/master/docs/grammar.md)라는 Python Parsing Toolkit을 사용해서 파싱했던 기억이 난다.  
  
```SQL  
SELECT id, name, grade FROM student LEFT JOIN student_grade;  
```  
  
EBNF는 Context-Free Grammar이기 때문에 특정 언어에 종속되지 않으며 대략적으로 다음과 같이 생겼다.  
```bnf  
// SELECT  
select_query : SELECT select_list table_expression  
select_list : "*"  
            | selected_column ("," selected_column)*  
selected_column : [table_name "."] column_name [AS column_name]  
table_expression : from_clause [where_clause]  
from_clause : FROM table_reference_list  
table_reference_list : referred_table ("," referred_table)*  
referred_table : table_name [AS table_name]  
where_clause : WHERE boolean_expr  
boolean_expr : boolean_term (OR boolean_term)*  
boolean_term : boolean_factor (AND boolean_factor)*  
boolean_factor : [NOT] boolean_test  
boolean_test : predicate  
             | parenthesized_boolean_expr  
parenthesized_boolean_expr : LP boolean_expr RP  
predicate : comparison_predicate  
          | null_predicate  
comparison_predicate : comp_operand ("<" | ">" | "=" | ">=" | "<=" | "!=") comp_operand  
comp_operand : comparable_value  
             | [table_name "."] column_name  
comparable_value : INT | STR | DATE  
null_predicate : [table_name "."] column_name null_operation  
null_operation : IS [NOT] NULL  
```  
  
글을 읽으면서 흥미로웠던 점은.CSS나 JS의 경우 이러한 BNF의 형태를 사용해서 파싱을 구현할 수 있으나 HTML5는 이러한 Context-Free Grammar를 사용해서 파싱할 수 없다는 점이었다. 이유는 HTML이 굉장히 "너그러운" 성질을 갖고 있기 때문이다. 암묵적으로 태그에 대한 생략이 가능하고, 잘못된 태그를 입력해도 오류를 일으키지 않고 적절히 추론해서 보여준다. 따라서 기존의 BNF 형태의 파서를 사용할 수 없고, 완전히 새로운 형태의 파서를 직접 구현해야 한다는 점이 흥미로웠다.  
>Unfortunately all the conventional parser topics do not apply to HTML (I didn't bring them up just for fun - they will be used in parsing CSS and JavaScript). HTML cannot easily be defined by a context free grammar that parsers need.  
  
There is a formal format for defining HTML - DTD (Document Type Definition) - but it is not a context free grammar.  
  
### Style Sheet Processing  
이 글을 읽게 된 이유가 바로 이 부분 때문이다. 프론트엔드 팀 내에서 TechTalk을 하다가 link로 불러오는 css가 FOUC를 일으키는가? 에 대한 질문이 나왔고, 이에 대해 명확하게 답하지 못해서 이 부분을 다시 챙겨보기로 했었다.  
  
결론적으로는 "브라우저에 따라 다르다"이다.  
- 웹킷 기반 브라우저의 경우 CSS가 아직 로드되지 않은 상황에서도 빠르게 렌더링하려는 경향이 있기 때문에 FOUC가 두드러지게 발생한다  
- 파이어폭스는 일반적으로 CSS가 완전히 로드될 때까지 렌더링을 차단하는 경향이 있기 떄문에 FOUC현상이 덜 두드러지지만 완전히 발생하지 않는다고는 보기 어렵다.  
  
>한편 스타일 시트는 다른 모델을 사용한다. 이론적으로 스타일 시트는 DOM 트리를 변경하지 않기 때문에 문서 파싱을 기다리거나 중단할 이유가 없다. 그러나 스크립트가 문서를 파싱하는 동안 스타일 정보를 요청하는 경우라면 문제가 된다. 스타일이 파싱되지 않은 상태라면 스크립트는 잘못된 결과를 내놓기 때문에 많은 문제를 야기한다. 이런 문제는 흔치 않은 것처럼 보이지만 매우 빈번하게 발생한다. 파이어폭스는 아직 로드 중이거나 파싱 중인 스타일 시트가 있는 경우 모든 스크립트의 실행을 중단한다. 한편 웹킷은 로드되지 않은 스타일 시트 가운데 문제가 될만한 속성이 있을 때에만 스크립트를 중단한다.  
  
  
## Typescript Structural Typing  
  
Typescript에 정의된 Object.keys interface는 다음과 같은 형태이다.   
```typescript  
// typescript/lib/lib.es5.d.ts  
interface Object {  
  keys(o: object): string[];  
}  
  
```  
  
Generic을 지원하지 않아서 다음과 같이 작성하면 오류가 난다  
```typescript  
interface Options {  
  hostName: string;  
  port: number;  
}  
  
function validateOptions (options: Options) {  
  Object.keys(options).forEach(key => {  
    if (options[key] == null) {  
        // @error Expression of type 'string' can't be used to index type 'Options'.  
      throw new Error(`Missing option ${key}`);  
    }  
  });  
}  
  
```  
  
Q. Object.keys는 왜 Generic을 지원하지 않을까? 다음과 같이 Generic을 지원한다면 참 편할 것 같다  
```typescript  
class Object {  
  keys<T extends object>(o: T): (keyof T)[];  
}  
```  
  
- 사용하다 보면 Object.keys의 key를 string이 아닌 실제 Object의 key 타입으로 추론해서 사용하고 싶을 때가 있다. (위의 예제에서는 'hostName' | 'port'). 어차피 Option이라는 인터페이스를 알고 있을텐데 왜 굳이 해당 인터페이스의 key값이 아니라 string으로 정의해버리는걸까?  
  
A. 이는 Typescript의 Structural Typing 때문에 그렇다  
> TypeScript의 타입 호환성은 구조적 서브 타이핑(subtyping)을 기반으로 합니다. 구조적 타이핑이란 오직 멤버만으로 타입을 관계시키는 방식입니다. 명목적 타이핑(nominal typing) 과는 대조적입니다. 다음 코드를 살펴보겠습니다:  
  
프로그래밍 언어에서 널리 사용되는 타이핑 방식에는 명목적 서브타이핑(Nominal Subtyping)과 구조적 서브타이핑(Structural Subtyping)이 있다.  
  
### Nominal Subtyping  
- 타입 정의 시에 상속 관계임을 명확히 명시한 경우에만 타입 호환을 허용하는 것.  
  
### Structural Subtyping  
- 상속 관계가 명시되어 있지 않더라도 객체의 프로퍼티를 기반으로 사용함에 문제가 없다면 타입 호환을 허용하는 것.   
- 이런 의미에서 Duck Typing이라고도 함  
> 만약 어떤 새가 오리처럼 걷고, 헤엄치고, 꽥꽥거리는 소리를 낸다면 나는 그 새를 오리라고 부를 것이다.  
  
Typescipt는 [일부 예외](https://github.com/Microsoft/TypeScript/pull/3823)를 제외하고는 구조적 서브타이핑을 사용하기 때문에 위의 예제에서의 Options 타입에 "호환"된다고 여겨지는 인터페이스가 'hostName', 'port'이외의 다른 property를 가지고 있을 수 있다. 따라서 Options의 interface만을 가지고 Object.keys의 타입을 추론할 수 없는 것이다.  
>They key takeaway is that when we have an object of type `T`, all we know about that object is that it contains _at least_ the properties in `T`.  
>  
>We do **not** know whether we have _exactly_ `T`, which is why `Object.keys` is typed the way it is. Let's take an example.  
  
따라서 의도적으로 string으로 타이핑함으로써, 의도하지 않은 프로퍼티가 인터페이스 안에 정의되어 있을 수 있음을 알려주는 것이라 할 수 있다.   
>We now have our answer for why `Object.keys` is typed the way it is. It forces us to acknowledge that objects may contain properties that the type system is not aware of.  
  
### Making use of Structrual Typing  
Typescript의 Structural Typing을 잘 이용하면 오히려 편리한 타입 추론을 사용할 수 있다. 글에서는 다음과 같은 예시를 제안한다  
  
```typescript  
function getKeyboardShortcut(e: KeyboardEvent) {  
  if (e.key === "s" && e.metaKey) {  
    return "save";  
  }  
  if (e.key === "o" && e.metaKey) {  
    return "open";  
  }  
  return null;  
}  
  
getKeyboardShortcut({ key: "s", metaKey: true });  
                    // @error Type '{ key: string; metaKey: true; }' is missing the following properties from type 'KeyboardEvent': altKey, charCode, code, ctrlKey, and 37 more.  
  
  
```  
  
실제 Keyboard Event는 저것 이외의 수십가지의 property가 정의되기 때문에 아래와 같이 사용하면 타입 에러를 내지만 Typescript의 Structural Typing을 사용해서 다음과 같이 정의하면 `KeyboardShortcutEvent`에 대해 `KeyboardEvent`가 구조적으로 호환되기 때문에 편리하게 Unit Test를 작성할 수 있다.  
  
```typescript  
interface KeyboardShortcutEvent {  
  key: string;  
  metaKey: boolean;  
}  
  
function getKeyboardShortcut(e: KeyboardShortcutEvent) {}  
  
```  
  
## Reference  
- [How Browser Works](https://d2.naver.com/helloworld/59361)  
	- [Original Article](https://web.dev/howbrowserswork/)  
- [React Use Hook](https://yceffort.kr/2023/06/react-use-hook)  
- [Typescript Structural Typing](https://alexharri.com/blog/typescript-structural-typing)  
	- [Structural Typing - Toss](https://toss.tech/article/typescript-type-compatibility)