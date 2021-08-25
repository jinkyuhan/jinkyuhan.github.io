<h1 style="text-align: center; ">JS와 Java8의 클로저 지원 비교</h1>
<p style="text-align: right;"> updated 2020.08 </p>

## 💡 클로저란?

---

&nbsp;어떤 스코프 S(Lexical environment) 내에서 함수 F를 선언(생성)할 때, 함수 F안에 스코프 S에서만 접근 할 수 있는 지역 변수 V를 사용 한 경우, 이를 기억해놨다가 스코프 S가
종료되더라도 함수 F의 실행 시점에 기억된 변수 V에 접근할 수 있는 함수 F를 클로저 라고 한다

**&nbsp;즉, 함수를 정적인 하나의 값으로 다루는 함수형 프로그래밍에서, 선언시에만 존재하는 변수를 어떤식으로 생성되는 함수안에 담을 것인가에 대한 논의의 결과이다.**

## 자바스크립트의 클로저 원리와 주의사항

---

자바스크립트에서는 모든 함수가 클로저이다. 즉, 모든 함수는 자기자신이 선언된 lexical environment를 저장한다. 아래는 **Javascript** 에서 보여지는 클로저의 예시이다.

```javascript
function functionMaker() {
    // F를 선언하는 스코프 S
    let count = 0;
    const localVariable = "example";
    const localObject = {name: "closure"};
    return (name) => { // 함수 F
        count += 1;
        localObject.name = name;
        console.log(count);
        console.log(localVariable);
        console.log(localObject);
        console.log(localObject.name);
    }
}

/* 
함수 생성 1, 
functionMaker가 실행될때 생성된 lexical environment정보가 F1.[[environment]]에 저장됨.
*/
const F1 = functionMaker();

/* 
함수 생성 2, 
functionMaker가 실행될때 생성된 lexical environment정보가
F2.[[environment]]에 저장됨.
*/
const F2 = functionMaker();

// functionMaker가 종료 되었지만 함수 F1, F2를 실행할때 localVariable와 localObject에 접근가능.
F1('F1_A');
// output:
//   1
//   example
//   { "name": "F1_A" }
//   F1_A
F1('F1_B');
// output:
//   2
//   example
//   { "name": "F1_B" }
//   F1_B

F2('F2_A');
// output:
//   1
//   example
//   { "name": "F2_A" }
//   F2_A

```

Javascript가 closure를 지원하지 않는다면 functionMaker() 호출이 종료되면서 localVariable과 localObject가 스택에서 제외되기 때문에 접근이 불가능할 것이다. 하지만
자바스크립트는 closure를 지원한다. 즉, 위 코드에서 **F1()이 실행되면서 F1함수의 내용이 evaluate 될 때, 함수가 생성될때 선언 되었던 지역변수들이 메모리에 살아 있다는 뜻이 된다.**

왜냐하면 Javascript의 GC는 접근할 수 없는(refrence 될 수 없는) 메모리들을 비워내는데, functionMaker()가 첫번째로 실행될때의 **lexical environment가
F1.[[environment]]에 저장되어 지역변수에 접근이 가능한 상황이므로 메모리에서 사라지지 않는다.**

주의할점은 바로 여기에 있다. 메모리의 lexical environment에 살아있는 지역변수들은 매 함수의 실행 시점마다 공유된다. 예컨데 함수 F1의 입장에서 보면 count, localVariable,
localObject들은 마치 전역변수 처럼 매 함수 실행 시 마다 공유하게 되는 변수이다. **F1의 첫번째 실행(F1_A)와 F1의 두번째 시행(F1_B)가 실행순서를 고려하지 않은 상태에서 사용되었다면, 함수형
프로그래밍의 관점에서 이야기하는 SideEffect가 일어나고, RaceHazard가 발생함으로** **클로저에서 외부 lexical environment의 변수에 쓰기 명령을 수행하는 것은 주의하여야 하고
개인적으로 바람직하지 않다고 생각한다.**

# Java8의 클로저 지원

--- 

&nbsp; [내가 공부한 바에 따르면](./Java8의_람다_함수형인터페이스.md), Java8에서 함수형 프로그래밍을 위해 사용하는 람다표현식은 FunctionalInterface의 익명객체를 생성하는
것이다.<br>

&nbsp; JVM에서 객체는 Heap영역에 생성된다. 하지만 메소드가 실행되면서 선언된 람다 밖의 변수는 Stack 영역에 생성된다.<br>
따라서 만약 람다표현식으로 생성된 객체가 Heap에 사라지기 전에 지역변수가 속한 해당 Stack 영역이 Pop된다면, **람다 표현식 내부에서 값을 변경하려고 할때 해당 변수가 메모리에 살아있지 않음으로
NullPointerException을 피할 수 없다.**<br>

&nbsp;그래서 **Java8은 클로저를 읽기에 한해서만 부분적으로 지원한다.** 람다표현식 안에서 외부 lexical environment의 변수에 쓰기를 시도하면 해당 변수는 final로 선언 되었다고
경고문구가 뜬다. 이를 봤을때, 람다표현식으로 새로운 함수형 객체를 생성할 때, **lexical environment의 변수를 final로 복사하여 함수형 인터페이스의 요소로 저장함을 알 수 있다.** <br>
&nbsp;즉, 메소드 안에서 사용된 람다표현식에서 Lexical environment의 변수에 접근할 때는, 읽기가능한 복사본만을 갖기 때문에, 읽기에 한해서만 접근이 가능하다.

```java
public class lamdaClosure {

    public static void main(String[] args) {
        int[] nums = {1, 2, 3};
        int count = 0;
        Arrays.stream(nums).forEach((num) -> {
            // lambda A
            count++; // lambda A 안에서 count는 final임으로 수정 불가, Error in Compile time!
            System.out.println(num);
        });
    }
}
```