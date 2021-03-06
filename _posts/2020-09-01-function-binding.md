---
layout: post
title:  "JS interop - 함수 바인딩"
date:   2020-09-01 17:53:57 +0900
categories: jekyll update
comments: true
---

어제에 이어서 함수 바인딩에 대해서 끝까지 알아보겠습니다.
## Variadic 함수 바인딩
- JS 함수중에서는 여러개의 argument를 인자로 받는 경우가 있습니다. ReScript에서도 그런 경우를 처리해주는데요, 이 때 각 인자들은 같은 타입이어야 한다는 조건이 붙습니다. 아래는 예시입니다.

```javascript
@bs.module("path") @bs.variadic
external join: array<string> => string = "join"
let v = join(["a", "b"]) //a와 b가 같은 타입입니다. (string)
```

```javascript
var Path = require("path");
var v = Path.join("a", "b");
```

## 폴리몰픽 함수 바인딩
- 하나의 함수가 여러 형태로 오버로딩(overloading)되는 경우는 어떻게 바인딩할까요? 여기엔 두가지 방법이 존재합니다. 첫째, 여러개의 externals를 사용하는 방법입니다.

```javascript
@bs.module("MyGame") external drawCat: unit => unit = "draw"
@bs.module("MyGame") external drawDog: (~giveName: string) => unit = "draw"
@bs.module("MyGame") external draw: (string, ~useRandomAnimal: bool) => unit = "draw"
```
- 위의 경우는 **draw** 라는 함수가 각기 다른 인자를 가지도록 externals를 이용해 여러번 overloading 된 경우입니다. 예상을 벗어나지 않는 직관적 형태였는데요, 그럼 이번엔 아래와 같은 경우를 살펴봅시다.

```javascript
function padLeft(value, padding) {
  if (typeof padding === "number") {
    return Array(padding + 1).join(" ") + value;
  }
  if (typeof padding === "string") {
    return padding + value;
  }
  throw new Error(`Expected string or number, got '${padding}'.`);
}
```
- padding의 타입이 정해져있지 않기 때문에 함수 안에서 직접 타입체킹을 한 후 값을 리턴하는 경우입니다. ReScript의 패턴매칭을 사용하고 싶은 욕구가 듭니다. 실제로 아래와 같이 바인딩할 수 있습니다.

```javascript
@bs.val
external padLeft: (
  string,
  @bs.unwrap [
    | #Str(string)
    | #Int(int)
  ])
  => string = "padLeft"
padLeft("Hello World", #Int(4))
padLeft("Hello World", #Str("Message from ReScript: "))
```
- 큰 틀에서 살펴보자면, 
1. @bs.val 을 통해 JS에 존재하는 padLeft를 바인딩함을 알려줍니다.
2. @bs.unwrap으로 두번째 인자의 타입에 대한 패턴매칭을 해줍니다. @bs.unwrap은 variant 생성자를 벗겨내서 실제 padding의 값으로 컴파일하는 역할을 합니다.

## 인자를 더 타이트하게 바인딩하기
- 인자의 타입을 string으로 제한함과 동시에, string에서 특정한 값들만 들어오게 하고 싶은 경우가 있을 수 있습니다. fs.readFileSync(*.text, utf8 | ascii)의 경우과 같이요. 그럴때는 아래와 같이 할 수 있습니다.
```javascript
@bs.module("fs")
external readFileSync: (
  ~name: string,
  @bs.string [ // (1)
    | #utf8
    | @bs.as("ascii") #useAscii // (2)
  ],
) => string = "fs"
```

- 우선 fs 안의 함수를 바인딩한 것이긱 때문에  @bs.module("fs")로 시작합니다.
1. @bs.string 은 [] 안의 결과가 string임을 나타내고, 
2. @bs.as 는 #useAscii 가 매칭되었을 경우 ascii 값 @bs.as안의 값을 반환하도록 합니다. @bs.as는 여러 쓰임새가 있는데 한가지 예로는 record 타입의 **대문자로 시작하는 키를 갖지 못하는 규칙** 을 우회할 때도 사용할 수 있습니다.

```javascript
type test = {
    @bs.as("Capitalized")
    unCapitalized: string
}
```
이렇게요.

- 놀랍지 않게도, string 뿐만 아니라 int도 타이트한 바인딩이 가능합니다.
```javascript
@bs.val
external testIntType: (
  @bs.int [
    | #onClosed
    | @bs.as(20) #onOpen
    | #inBinary
    | #inTertiary
  ])
  => int = "testIntType"

testIntType(#inTertiary) //testIntType(22)
```
- 유추할 수 있듯이, onOpen이 20으로 세팅되고 난 이후의 옵션들에 대해서는 숫자각 1씩 커집니다.

## 이벤트 리스너
- 이벤트 리스터는 어떻게 바인딩할까요? 이것도 위의 방법들과 다르지 않습니다. 먼저 JS코드를 보면 

```javascript
function register(rl) {
  return rl
    .on("close", function($$event) {})
    .on("line", function(line) {
      console.log(line);
    });
}
```
- register라는 함수는 readline(rl)을 받아서 close 일때 실행하는 함수와 line일때 실행하는 함수가 각각 구현되어 있습니다. 이 'on' 이라는 함수를 바인딩 할 때는

1. string close 와 line에 대한 패턴매칭이 있어야하고
2. rl의 인자와 리턴값에 대한 정의를 알아야 합니다.
3. rl.on().on() 의 형식으로 체이닝이 되고 있으니 .on()의 리턴값은 rl임을 유추할 수 있습니다.
4. 호출자는 rl 입니다.

```javascript
let register = rl =>
  rl
  ->on(#close(event => ()))
  ->on(#line(line => Js.log(line)));
```
- 위 처럼 rl을 인자로 받아 close 일때는 유닛=>유닛 을, line일 때는 스트링=>유닛 의 함수를 호출해주도록 하면 됩니다. on은 pipe operator로 연결됩니다. 위의 1,2,3,4를 유의해서 바인딩해보면,

```javascript
@bs.send
external on: (
    readline,
    @bs.string [
      | #close(unit => unit)
      | #line(string => unit)
    ]
  )
  => readline = "on"
```
- 호출자가 rl 이기 때문에 (readline, ()=>()) => readline 이 형식이며,
- 함수는 @bs.string으로 패턴 매칭을 해줬습니다.
- 상당히 어려운데, 많이 익숙해져야 할 부분같습니다 ^.^;;

## this를 가진 콜백 바인딩하기
- 이런 함수가 있다고 가정합시다.
```javascript
x.onload = function(v) {
  console.log(this.response + v)
}
```
- 사실 이 함수는 이렇게 풀이될 수 있습니다.

```javascript
x.onload = function (v) {
  let o = this;
  console.log((o.response + v) | 0);
};
```
- onload를 ReScript에서 사용하고 싶다고 할 때, 차근차근 바인딩 해야 할 부분을 살펴보자면,
1. x가 onload를 호출자입니다. (첫번째 인자로 들어갑니다) a.b = c 오퍼레이션이 있기 때문에 @bs.set을 해줘야 할 것 같습니다. 
2. **this 를 o에 할당합니다** //감이 안잡히는군요
3. o.response는 o에서 response 값을 꺼내오는 것이기 때문에 @bs.get을 해줍니다. (response가 int라 가정합니다)
- 결과는 아래와 같습니다.

```javascript
type x
@bs.val external x: x = "x"
@bs.set external setOnload: (x, @bs.this ((x, int) => unit)) => unit = "onload"
@bs.get external resp: x => int = "response"
setOnload(x, @bs.this ((o, v) => Js.log(resp(o) + v)))
```

- 먼저 x type을 지정해주고,
- x.onload = function에 해당하는 부분을 작성합니다. 이 때, console.log는 unit를 리턴하므로 크게 볼 때 @bs.this() => unit의 형태가 됩니다.

1. @bs.this를 눈여겨 봅시다. 결국 x에 this가 바인딩하므로 this()의 첫번째 인자로 x가 옵니다. this(x)
2. function(v) 에서 v가 정수이므로 this(x, int)가 되는데, 이 this 바인딩은 아무 리턴이 없으므로 unit을 리턴한다고 되어 있습니다.
3. 마지막으로 큰 틀에서 보면 console.log 또한 리턴이 없으므로, unit을 리턴합니다.

- response는 x를 받아 int를 돌려주는 형태 (this.response) 임으로 @bs.get으로 처리해줍니다. 
- 마지막으로 setOnload를 호출합니다.

- 으으.. 너무 복잡해서 한번에 잘 이해가 가지 않는데요, ~~내일의 TIL을 하며 내용을 보완해보겠습니다.~~ 보완해버렸습니다!!