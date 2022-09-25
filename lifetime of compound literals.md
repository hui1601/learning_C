# Compound Literals의 lifetime은?

전 코드를 길게 작성하는걸 ~~손 아파서~~ 귀찮아하고, 변수명 짓기 귀찮기 때문에 ISO C99부터 추가된 Compound Literals(복합 리터럴)을 자주 사용합니다

```c
#include <stdlib.h>
struct Point{
    int x, y;
    struct Point *next;
};
struct Point* do_something(){
    return &(struct Point){10, 2, NULL};
}
int main(void){
    struct Point* something = do_something();
    something->x; // 펑!
}
```

이 코드는 얼마전 만든 코드의 일부입니다.

그런데 리턴값에 접근하려니 바로 터져버리더군요.

당연하겠지만, 지역변수의 값을 안넘기고 포인터로 넘겼기 때문입니다.

컴파일할 때 이런 오류를 뱉습니다.

```log
main.c: In function ‘do_something’:
main.c:7:12: warning: function returns address of local variable [-Wreturn-local-addr]
    7 |     return &(struct Point){10, 2, NULL};
      |            ^~~~~~~~~~~~~~~~~~~~~~~~~~~~
main.c:7:27: note: declared here
    7 |     return &(struct Point){10, 2, NULL};
      |            
```

복합 리터럴을 쓰면 그 메모리 영역은 블록 스코프를 가집니다.

> In C, a compound literal designates an unnamed object with **static or automatic storage duration**.
> 
> C에서 복합 리터럴은 **정적 저장 기간 또는 자동 저장 기간**을 가진 이름 없는 객체를 지정합니다.
> 
> [GCC 6.28 Compound Literals](https://gcc.gnu.org/onlinedocs/gcc/Compound-Literals.html)

여기서 설명하는 동작은 C기준이므로 C++에서는 다소 다르니 주의하세요.

복합 리터럴은 C에서 지역변수와 같은 automatic storage duration을 가집니다.

```c
int main(void){
    (struct Point){10, 2, NULL};
    (struct Point){11, 2, NULL};
    (struct Point){12, 2, NULL};
    (struct Point){13, 2, NULL};
    (struct Point){14, 2, NULL};
    (struct Point){15, 2, NULL};
}
```

이런 짓은 어떨까요?

C에서 복합 리터럴은 지역변수와 같은 lifetime을 지닙니다. 따라서 main함수가 끝날때까지 6개의 복합 리터럴의 할당은 해제되지 않습니다.

~~그러니 메모리를 극단적으로 아낄려면 저처럼 익명 임시 변수마냥 막쓰지 말라는겁니다~~

C++에서는 어떨까요?

안타깝지만 C++ 표준에는 없고 clang과 gcc에서 일종의 언어 확장으로 지원해줍니다. 다만 동작이 약간 다릅니다.

```cpp
#include <cstdlib>
struct Point{
    int x, y;
    struct Point *next;
};
struct Point* do_something(){
    struct Point* a = &(struct Point){10, 2, NULL}; // ERROR!!
    return a;
}
int main(void){
    struct Point* something = do_something();
    something->x;
}
```

```log
main.cpp: In function ‘Point* do_something()’:
main.cpp:7:39: error: taking address of rvalue [-fpermissive]
    7 |     struct Point * a = &(struct Point){10, 2, NULL};
      |                                       ^~~~~~~~~~~~~
```

아니, 왜 접근하기도 전에 포인터 선언부터 오류가 나는 걸까요?

이유는 간단합니다. C++에서 복합 리터럴은 lifetime이 전체구문의 끝이기 때문입니다.

> In C++, a compound literal designates a temporary object that only **lives until the end of its full-expression**. As a result, well-defined C code that takes the address of a subobject of a compound literal can be undefined in C++, so G++ rejects the conversion of a temporary array to a pointer.
> 
> C++에서 복합 리터럴은 **전체 표현이 끝날 때까지만** 지속되는 임시 개체를 지정합니다. 결과적으로 복합 리터럴의 하위 개체 주소를 사용하는 잘 정의된 C 코드는 C++에서 정의되지 않을 수 있으므로 G++에서는 임시 배열을 포인터로 변환하는 것을 거부합니다.
> 
> [GCC 6.28 Compound Literals](https://gcc.gnu.org/onlinedocs/gcc/Compound-Literals.html)

그러니 `(struct Point*)a`가 선언된 후 복합리터럴의 생명이 끝나서 ~~죽음~~ 메모리의 할당이 해제될 수 있기 때문입니다. 즉, UB(undefined behavior)입니다.

Q: 아니 구문의 끝이 lifetime이면 어떻게 쓰는데요?

A: 이렇게 임시변수처럼 쓸 수 있답니다.

```cpp
for (auto i : {2.7, 3.1}) cout << i << endl;
```

~~C++에서는 익명 임시 변수처럼 써도 좋습니다.~~
