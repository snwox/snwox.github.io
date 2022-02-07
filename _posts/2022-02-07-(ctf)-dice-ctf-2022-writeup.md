---
title: (CTF) Dice CTF 2022 writeup
categories: [CTF]
tags : [2022, ctf, writeup]
date : 2022-02-07 13:16:37 +0900
author:
    name: 
    link: 
toc: false
comments: false
mermaid: false
math: false
---

# (REV) flagle

![Untitled](/assets/img/2022-02-07-(ctf)-dice-ctf-2022-writeup/Untitled.png)

한 줄에 `5*6` 글자씩 넣어보면, 맞는지 아닌지 출력해준다.

## analysis

```jsx
const submit_guess = () => {
        let correct = 0;
        let input_text = '';
        for (let i = 1; i <= 6; ++i) {
            const el = get_input(current_guess, i);

            const guess_val = el.value;
            input_text += guess_val;

            const result = guess(i, guess_val);
            console.log(result, guess_val, i, guess_val);
            if (result === CORRECT) {
                el.classList.add('correct');
                if (current_guess < 6) {
                    const next = get_input(current_guess + 1, i);
                    next.value = guess_val;
                    next.classList.add('correct');
                }
                correct++;
            } else if (result === WRONG_LOCATION) {
                el.classList.add('partial');
            } else if (result === INCORRECT) {
                el.classList.add('incorrect');
            }
            el.disabled = true;
            el.removeEventListener('keydown', keydown_listener);
        }
        current_guess++;

        if (correct === 6) {
            prompt('Congrats! Here\'s your flag:', input_text);
        }
```

`current guess` 는 행을 , `i` 는 열을 의미한다.

`guess` 함수를 통해 해당 5글자가 맞는지 확인한다.

```jsx
const guess = Module.cwrap('guess', 'number', ['number', 'string']);

// flag-checker.js
Module["ccall"] = ccall;
Module["cwrap"] = cwrap;

function cwrap(ident, returnType, argTypes, opts) {
    return function () {
        return ccall(ident, returnType, argTypes, arguments, opts);
    }
}

```

guess 함수는 Module 의 cwrap 을 통해 함수를 호출합니다.

cwrap 에서 호출되는 함수인 ccall 은 `Module` 에 등록되어있는 `flag-checker.wasm` 의 모듈 중 해당 `ident` (여기서는 guess) 를 가진 함수를 호출합니다.

```jsx
var asm = createWasm();
/** @type {function(...*):?} */
var ___wasm_call_ctors = Module["___wasm_call_ctors"] = createExportWrapper("__wasm_call_ctors");

/** @type {function(...*):?} */
var _a = Module["_a"] = createExportWrapper("a");

/** @type {function(...*):?} */
var _streq = Module["_streq"] = createExportWrapper("streq");

/** @type {function(...*):?} */
var _validate_1 = Module["_validate_1"] = createExportWrapper("validate_1");

/** @type {function(...*):?} */
var _validate_2 = Module["_validate_2"] = createExportWrapper("validate_2");

/** @type {function(...*):?} */
var _validate_3 = Module["_validate_3"] = createExportWrapper("validate_3");

/** @type {function(...*):?} */
var _validate_5 = Module["_validate_5"] = createExportWrapper("validate_5");

/** @type {function(...*):?} */
var _validate_6 = Module["_validate_6"] = createExportWrapper("validate_6");
```

이런게 있는걸 봐서는 각 열마다 검증하는게 다를 것을 알 수 있습니다.

---

`eunjo` 님이 wasm 파일을 바이너리로 바꿔주셔서 바이너리를 분석해보면

if 문으로 각 열에 대해 문자가 올바르지 판단하는것을 볼 수 있었습니다.

### row 1

![Untitled](/assets/img/2022-02-07-(ctf)-dice-ctf-2022-writeup/Untitled%201.png)

문자열 비교

```python
flag[0][0] = ord('d')
flag[0][1] = ord('i')
flag[0][2] = ord('c')
flag[0][3] = ord('e')
flag[0][4] = ord('{')
```

---

### row 2

![Untitled](/assets/img/2022-02-07-(ctf)-dice-ctf-2022-writeup/Untitled%202.png)

각 문자들을 메모리에서 로드한 후, 

![Untitled](/assets/img/2022-02-07-(ctf)-dice-ctf-2022-writeup/Untitled%203.png)

옮기는 과정의 흐름을 잘 따라가 보면,

![Untitled](/assets/img/2022-02-07-(ctf)-dice-ctf-2022-writeup/Untitled%204.png)

각 문자마다 상수와 비교하는 것을 볼 수 있습니다.

```python
flag[1][0] = ord("F")
flag[1][1] = ord("!")
flag[1][2] = ord("3")
flag[1][3] = ord("l")
flag[1][4] = ord("D")
```

---

### row 3

![Untitled](/assets/img/2022-02-07-(ctf)-dice-ctf-2022-writeup/Untitled%205.png)

각각 사칙연산한 결과를 비교하는데, z3 으로 풀 수 있다.

```python
s = z3.Solver()
a = [z3.Int('a_%d' % (i)) for i in range(5)]
s.add(a[0]*a[1] == 0x12c0,
      a[0]+a[2] == 178,
      a[1]+a[2] == 126,
      a[3]*a[2] == 9126,
      a[3]-a[4] == 62,
      4800*a[2]-a[3]*a[4] == 367965)
for i in range(5):
    s.add(32 <= a[i])
    s.add(a[i] <= 128)

if s.check() == sat:
    for i in range(5):
        flag[2][i] = int(str(s.model()[a[i]]))
```

---

### row 4

![Untitled](/assets/img/2022-02-07-(ctf)-dice-ctf-2022-writeup/Untitled%206.png)

대충 validate_4 를 호출하는 것 같은데, wasm 바이너리에서는 없었고,

![Untitled](/assets/img/2022-02-07-(ctf)-dice-ctf-2022-writeup/Untitled%207.png)

`flag-checker.js` 를 보면 `validate_4` 만 저렇게 등록되어있는 것을 볼 수 있다.

```python
// flag_checker.js
function validate_4(a){ return c(UTF8ToString(a)) == 0 ? 0 : 1; }

// script.js
function c(b) { 
    var e = { 
        'HLPDd': function (g, h) { return g === h; }, 
        'tIDVT': function (g, h) { return g(h); }, 
        'QIMdf': function (g, h) { return g - h; }, 
        'FIzyt': 'int', 'oRXGA': function (g, h) { return g << h; }, 
        'AMINk': function (g, h) { return g & h; } 
    }, 
    f = current_guess; 
    try { 
        let g = e['HLPDd'](
            btoa(e['tIDVT'](intArrayToString, window[b](b[e['QIMdf'](f, 0x26f4 + 0x1014 + -0x3707 * 0x1)], e['FIzyt'])()['toString'](e['oRXGA'](e['AMINk'](f, -0x1a3 * -0x15 + 0x82e * -0x1 + -0x1a2d), 0x124d + -0x1aca + 0x87f))['match'](/.{2}/g)['map'](h => parseInt(h, f * f)))), 'ZGljZQ==') ? -0x1 * 0x1d45 + 0x2110 + -0x3ca : -0x9 * 0x295 + -0x15 * -0x3 + 0x36 * 0x6d; 
    } catch { return 0x1b3c + -0xc9 * 0x2f + -0x19 * -0x63; } 
}
```

`eunjo` 님이 어떤식으로 호출되는지 정리해주셨다.

```python
function c(b) {
    var e = {
            'isSame': function (g, h) {
                return g === h;
            },
            'call': function (g, h) {
                return g(h);
            },
            'minus': function (g, h) {
                return g - h;
            },
            'Int': 'int',
            'shift2': function (g, h) {
                return g << h;
            },
            'and': function (g, h) {
                return g & h;
            }
        },
        f = current_guess;
    try {
        let g = e.isSame(btoa(e.call(intArrayToString, 
						window[b](b[e.minus(f, 1)], e.Int)().toString(
						e.shift2(e.and(f, 4), 2 )
					).match(/.{2}/g).map(h => parseInt(h, f * f)))), 'ZGljZQ==') ? 1 : 0;
    } catch {
        return 0
    }
}
```

대충 코드를 정리해봤다. atob(`ZGljZQ==`) == `dice` 이고

window 객체에서 입력한 문자열을 메소드 이름으로,

row-1 번째 문자를 인자로 호출한 뒤, row 가 4 이상이면 `toString(16)` 이렇게 함수가 호출되어

2글자씩 16진수 정수를 가져와 문자열로 만든 뒤, base64 인코딩 한다.

일단, `toString` 의 인자가 0이 되면 안되니까 row 가 4 이상이어야 하고,

window 객체의 메소드 리턴값이 정수 0x64696365 (dice) 가 되어야한다.

window 객체에서 길이 5인 메소드를 다 넣어봤는데도, row 가 4 이상이어야하는것을 몰라서 안됐었는데, `Ainsetin` 님이 `cwrap` 인걸 찾아주셧다.

```python
window['cwrap']( ? , 'int')
```

이런식으로 호출하게 되는데, 등록된 함수중에서 이름이 한 글자인 함수를 찾아야한다.

```jsx
// flag-checker.js
/** @type {function(...*):?} */
var _a = Module["_a"] = createExportWrapper("a");

// wasm binary
__int64 w2c_a()
{
  if ( ++wasm_rt_call_stack_depth > 0x1F4u )
    wasm_rt_trap(7LL);
  --wasm_rt_call_stack_depth;
  return 0x64696365LL;
}
```

a 라는 함수가 있는것을 알 수 있는데, 바이너리에서 보면 0x64696365 를 리턴한다. 야호

f-1 == 3 (a 의 인덱스)

그러면 4번째 행에 `cwrap` 를 입력하면 된다.

---

### row 5

![Untitled](/assets/img/2022-02-07-(ctf)-dice-ctf-2022-writeup/Untitled%208.png)

row 5 도 일련의 과정을 거쳐 메모리에서 열심히 옮기면서 비교하게 되는데,

특정 값을 더하거나 빼는 것을 볼 수 있었다. 다시 빼주거나 더해주면 된다.

```jsx
flag[4][0] = ord('y')-12
flag[4][1] = ord('D')-4
flag[4][2] = ord('~')-6
flag[4][3] = 33
flag[4][4] = ord("M")
```

---

### row 6

![Untitled](/assets/img/2022-02-07-(ctf)-dice-ctf-2022-writeup/Untitled%209.png)

마지막 스테이지로, 조건이 적어서 z3 으로 풀리긴 풀리는데, 좀 오래걸렸당.

```jsx
s2 = z3.Solver()
a2 = [z3.Int('a2_%d' % i) for i in range(5)]
s2.add(
    (a2[0]+0x6e3) * (a2[1]+0xb75) == 0x53acdf,
    (a2[4] == 125),
    (a2[2]+0xf49)*(a2[3]+0x60a) == 0x62218f
)
for i in range(5):
    s2.add(32 <= a2[i])
    s2.add(a2[i] <= 128)
if s2.check() == sat:
    for i in range(5):
        flag[5][i] = int(str(s2.model()[a2[i]]))
```

---

## `solve.py`

```python
import z3
from z3 import sat
flag = [list([0]*5) for i in range(6)]

# row 1,2
flag[0][0] = ord('d')
flag[0][1] = ord('i')
flag[0][2] = ord('c')
flag[0][3] = ord('e')
flag[0][4] = ord('{')
flag[1][0] = ord("F")
flag[1][1] = ord("!")
flag[1][2] = ord("3")
flag[1][3] = ord("l")
flag[1][4] = ord("D")

# row 3
s = z3.Solver()
a = [z3.Int('a_%d' % (i)) for i in range(5)]
s.add(a[0]*a[1] == 0x12c0,
      a[0]+a[2] == 178,
      a[1]+a[2] == 126,
      a[3]*a[2] == 9126,
      a[3]-a[4] == 62,
      4800*a[2]-a[3]*a[4] == 367965)
for i in range(5):
    s.add(32 <= a[i])
    s.add(a[i] <= 128)

if s.check() == sat:
    for i in range(5):
        flag[2][i] = int(str(s.model()[a[i]]))

# row 4
flag[4][0] = ord('y')-12
flag[4][1] = ord('D')-4
flag[4][2] = ord('~')-6
flag[4][3] = 33
flag[4][4] = ord("M")

# row 6
s2 = z3.Solver()
a2 = [z3.Int('a2_%d' % i) for i in range(5)]
s2.add(
    (a2[0]+0x6e3) * (a2[1]+0xb75) == 0x53acdf,
    (a2[4] == 125),
    (a2[2]+0xf49)*(a2[3]+0x60a) == 0x62218f
)
for i in range(5):
    s2.add(32 <= a2[i])
    s2.add(a2[i] <= 128)
if s2.check() == sat:
    for i in range(5):
        flag[5][i] = int(str(s2.model()[a2[i]]))

    for x in flag:
        print(''.join(list(map(chr, x))))

"""output
dice{
F!3lD
d0Nu7
                   <-- cwrap   
m@x!M
T$r3}
"""

```

![Untitled](/assets/img/2022-02-07-(ctf)-dice-ctf-2022-writeup/Untitled%2010.png)
