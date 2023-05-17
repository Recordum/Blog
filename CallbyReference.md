Python 공부를 시작하면서 Java와는 다른 방식으로 함수를 호출함(Call by Object Reference)을 알게 되었다.    

좀더 자세히 알고 싶어 자료를 찾다보니 Call by Reference 라는 용어가 Call by Address와 혼용되어 사용됨을 알게되었고, 이번 기회에 헷갈렸던 용어들을 다시 정리하고자 한다.

* 목차
1. Call by Value? Call By Reference?
2. Java가 함수를 호출하는 방식
3. Python이 함수를 호출하는 방식


# Call by Value? Call by Reference?

## 1. Call by Value
> Call by Value : '값에 의한 호출' 즉, 값을 복사해서 함수에 전달하는 것을 의미한다.

[위키에 나온 정의] (https://en.wikipedia.org/wiki/Evaluation_strategy#Call_by_value)를 요약해보면 결국 함수에 전달된 변수가 호출자의 변수에 영향을 끼치지 않을떄 Call by Value 임을 알 수 있다.
즉. 매개변수는 전달인자를 복사한 지역변수의 형태로 동작하게 된다.

### Call by Value 예시
```
#include <stdio.h>

void swap(int a, int b) {
    int temp = a;
    a = b;
    b = temp;
}

int main() {
    int x = 5;
    int y = 10;
    printf("Before swap: x=%d, y=%d\n", x, y);
    swap(x, y);
    printf("After swap: x=%d, y=%d\n", x, y);
    return 0;
}
```
```
출력값 
Before swap: x=5, y=10
After swap: x=5, y=10
```
![](https://velog.velcdn.com/images/mingyu/post/644ee607-ab1b-43e4-8ccf-7d751cfebc58/image.png)

1. 지역변수 x, y 는 main함수에 스택프레임에 저장되어 있다. 

2. swap() 함수를 호출하고 매개변수 값으로 단순히 x, y의 __값(value)__ 5 와 10을 swap(x, y) 함수에 전달한다. swap 함수 스택프레임에 지역 변수 a와 b값이 저장되고 명령문을 실행한다.

3. swap() 함수가 return 되면 스택프레임에 a와 b또한 소멸된다.

즉 x와 a, y와 b는 완전히 다른 메모리 주소를 가지고 있는 변수이며. a, b가 변화해도 x, y 에 영향을 끼치지 않는다. 

## 2. Call by Reference 와 Call by Address
> Call by Reference : '참조에 의한 호출' 즉, 매개변수에 참조값을 전달하는 걸 의미한다.

 Call by Value와 달리 Call by Reference는 함수의 내부동작이 호출한 함수에 전달인자까지 영향을 주게 된다.

먼저 Call by Reference 예제로 잘못 쓰이는 코드를 먼저 봐보자.

``` 
#include <stdio.h>

void swap(int *a, int *b) {
    int temp = *a;
    *a = *b;
    *b = temp;
}

int main() {
    int x = 5, y = 10;
    printf("Before swapping, x = %d and y = %d\n", x, y);

    swap(&x, &y); // x와 y의 주소를 전달하여 값 교환
    printf("After swapping, x = %d and y = %d\n", x, y);
    return 0;
}
```
```
출력값
Before swapping, x = 5 and y = 10
After swapping, x = 10 and y = 5
```
1. main()은 swap() 함수를 호출하고 매개변수로 x, y의 __주소값__(&x, &y)을 넘긴다.

2. swap()에 매개변수 포인터(*a, *b)에 x,y값이 역참조 된다.

3. *a와 *b 값을 서로 바꾼다,

4. swap()리턴 후에 실제로 main 함수에 x,y 값이 변경됨을 확인 할 수 있다.


과정을 한번 봐보면 함수의 내부동작이 호출한 함수에 전달인자까지 영향을 주게 된다(3, 4)
그렇다면 이 예제는 Call by Reference가 맞을까?

> 엄밀히 말하면 이 예제는 Call by Reference가 아닌 __Call by Address__ 이다.

코드 진행 과정에서 1번을 자세히 봐보면 swap함수에 전달되는 것은 x의 '주소값'이다. 단순히 x의 값에서 직접 주소를 구해(&x) 주소값을 복사하여 매개변수에 넘겨주고 swap 함수가 포인터로 역참조할 뿐이다.   

다시 말하면 swap함수안에 포인터 변수에 x의 주소값을 저장한 것이다. 당연한 얘기지만 a의 주소값과 x의 주소값은 다르다.

즉 일종의 __Call by Value__ 방식이다. 그리고 이렇게 주소값을 넘기는 방식을 __Call by Address__ 라고 한다.

__사실 C언어는 모두 Call by Value방식으로만 함수를 동작시킨다__



### Call by Reference 예제


그렇다면 엄밀한 의미에 Call by Reference 예제를 확인해 보자. 앞서 언급했듯이 C언어로는 Call by Reference 방식이 안되니 C++를 사용하겠다.


```
#include <iostream>
using namespace std;

void increment(int& x) {
    x++;
}

int main() {
    int num = 10;
    cout << "Before increment, num = " << num << endl;
    increment(num); // 참조자를 이용하여 num 변수를 전달
    cout << "After increment, num = " << num << endl;
    return 0;
}
```

```
출력결과
Before increment, num = 10
After increment, num = 11
```
1. main 함수에서 increment() 함수를 호출하고 변수 num을 넘긴다.

2. increment 함수는 넘여온 num 변수를 참조자를 사용하여 num에 __직접 접근한다__

3. return 후 num 값이 1 더해진것을 확인 할 수 있다.

여기서 Call by Reference 와 Call by Address 차이가 잘 안느껴질 수 도있다.하지만 둘은 엄연히 다른 동작을 하고있다.

참조자로 선언한 x는 num과 같은 주소를 갖는다. 만약 Call by Address 였다면. x는 포인터일 것이고 num에 값을 가르키지만 포인터 자체의 주소(&x)는 다르다. 

Call by Reference 에서는 __매개변수로 num이라는 변수 자체가 넘어온것이다.__


# Java가 함수를 호출하는 방식

사실 Java가 Primitive type 은 Call by Value 로 Object 는 Call by Reference 로 동작하는 것처럼 보이지만, 사실 전부 Call by Value(pass-by-value)이다! 라는 글을 많이 접해보았을 것이다.   

그렇다면 Java가 왜 전부 Call by Value 인지 들여다 보도록 하자. 지금 까지 글을 잘 이해했다면 무리없이 이해할 수 있을 것이다. 

### 예제 코드

Car Class 필드인 value 값을 변경하는 코드이다

```java


class Car {
    private int value;
    public Car(int value) {
        this.value = value;
    }
    public int getCarValue() {
        return value;
    }
    public void setCarValue(int a) {
        this.value = value;
    }
}
public class CallBySomething {

    public static void main(String[] args) {

        Car myCar = new Car(5);
        System.out.println("Before ChangeValue : " + myCar.getCarValue());
        changeValue(myCar);
        System.out.println("After ChangeValue : " + myCar.getCarValue());
    }

    private static void changeValue(Car myCar) {
        myCar.setCarValue(100);
    	myCar = new Car(10000);
    }

}
```
```
출력결과
Before ChangeValue : 5
After ChangeValue : 100
```

과정을 살펴보자

1. main 함수에서 myCar 객체를 선언한다.(myCar 객체가 Heap영역에 생성)
![](https://velog.velcdn.com/images/mingyu/post/c2b1e677-55c5-4975-a8a1-7d9ec618e070/image.png)


2. changeValue()함수를 호출하고 myCar객체를 넘긴다.

3. changeValue()에서 myCar value값을 100으로 변경 해준다.
![](https://velog.velcdn.com/images/mingyu/post/3949ee4e-f891-415b-b643-144922f4d43a/image.png)

4. changeValue() 에서 myCar에 새로운 car객체를 할당한다.
![](https://velog.velcdn.com/images/mingyu/post/47f2c4c7-b42c-46ca-9342-7e074338611a/image.png)

5. changeValue()가 return 된다.
![](https://velog.velcdn.com/images/mingyu/post/4c3c99d1-bbd4-4a76-8bdf-bef3adaf6f12/image.png)


과정을 살펴보면 알 수 있듯이, Java는 Call by Reference가 아닌 그저 객체에 주소값을 넘기는 Call by Value 임을 알 수 있다.

> __만약 Java가 Call by Reference 였다면 changeValue()안에서 myCar에 새로운 객체(new Car(10000))이 할당 될떄 main() 에 myCar도 함께 변경되었을 것이고, 출력값이 10000이 되었을 것이다.__

# Python이 함수를 호출하는 방식

Python 호출 방식은 Call by Object Reference 라고 한다.
사실 호출방식을 이렇게 부르는 이유는 Python의 모든 자료형이 Object로 이루어져 있기 때문이다.

먼저 python에 자료형에 대해 알아보자.

## immutable/mutable

파이썬 자료형은  immutable / mutable 객체로 나뉜다.  
mutable 객체는 : dictionary/list 등이 있다.


![](https://velog.velcdn.com/images/mingyu/post/d93f3782-1b88-42b3-8b28-523688ba78bd/image.png)
>_그림 출처 https://wikidocs.net/91520_

그림과 같이 mutable 객체는 다른 객체들을 바인딩 하고 __수정이 가능하다__
즉 list에 객체를 append 하거나 pop 해도 시작주소는 변경되지 않는다.

반대로 immutable 객체는 변경이 불가능하다. 
```python
a = python2
a = python3
```
위 코드는 아래 그림과 같이 동작하는데. python2라는 immutable 객체가 저장된 a 에 python3를 할당하면 python2가 python3로 변경되는 것이 않고, python3 문자열 객체를 새로 생성하여 a에 할당한다.
![](https://velog.velcdn.com/images/mingyu/post/1526d6a6-9b8a-477d-bc50-8334ea1a3c05/image.png)
>_그림 출처 https://wikidocs.net/91520_

이러한 특징은 파이썬이 따로 변수에 자료형을 선언하지 않고 동적으로 할당할 수 있는 이유이기도하다.

## Call by Object Reference

즉 파이썬은 Java/C 에 있는 int/str 등의 자료형이 따로 없다. 모든 자료형에서 앞에 설명한 Java 에서 Object가 인자로 넘어갈때와 같은 형식으로 함수가 호출된다.

즉 Object를 인자를 넘길때는 Java와 같은 동작 방식이므로 immutable 객체인 int형을 넘길때 Java의 primitive type과 어떤 차이가 있는지 확인해보자.

### 예제코드
```
def changeValue(a):
    a = 6
    return

a = 10
changeValue(a)
print(a)
```
```
출력결과
10
```

출력결과가 마치 java에서 int 를 함수로 넘길떄와 같다.

하지만 동작과정은 다르니 과정을 살펴보자.

1. main 함수에서 a에 10 을 할당한다
![](https://velog.velcdn.com/images/mingyu/post/2bcafe40-5fae-464a-a7f6-58c4fa2fc048/image.png)

2. changeValue 인자로 a에 객체참조값이 전달된다.
![](https://velog.velcdn.com/images/mingyu/post/8547e76f-d704-4172-9076-6779e4c73cbe/image.png)

3. changeValue a에 6이 할당된다.
![](https://velog.velcdn.com/images/mingyu/post/f277b12b-8503-4324-a55e-788c2c611848/image.png)

4. chageValue가 return된다.
![](https://velog.velcdn.com/images/mingyu/post/e218a07f-0c13-4edc-bb34-967decd3e4e4/image.png)

마치 Java의 Object가 함수에 전달될떄와 비슷하다.

__즉 Python은 호출 방식이 모두 ObjectReference값을 넘기므로 Call by ObjectReference 방식이라고 한다.__

## 참고
https://wikidocs.net/91520
https://en.wikipedia.org/wiki/Evaluation_strategy#Call_by_value
https://stackoverflow.com/questions/40480/is-java-pass-by-reference-or-pass-by-value/73021#73021
