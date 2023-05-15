OS 는 Dual-Mode-Operation으로 불리는 Kernel mode 와 User mode 를 지원한다.

__user application 이 실행되는 영역(User mode) 과 kernel 이 실행되는 영역(Kernel mode)을 나누고 privilege 차등을 두어 안전성을 보안을 유지한다.__

이번 포스팅에서 Dual-Mode-Operation에 대해 알아보려 한다. 특히 PintOS project2의 코드를 통해, 추상적으로 설명되던 Kernel mode 와 User mode 사이 전환(transition) 과정을 자세히 살펴보자.

* 참고사항
	1. __KAIST PintOS Project2베이스로 작성된 글입니다. 하지만 Project 2 정답코드에 대한 설명은 포함 되어있지 않습니다.__ 
  	2. PintOS 프로젝트를 수행하지 않아도, PintOS 내용을 제외하고는 이해할 수 있는 글을 작성하는게 목표입니다.

* 목차
		1. Protection ring
		2. System Call
	    3. PintOS System Call(transition) 과정


# 1. Protection ring
![](https://velog.velcdn.com/images/mingyu/post/0d73f9b6-e8fe-4646-930e-7b3774c9736a/image.png)


> 보호 링(protection rings)은 결함 (결함 내성) 및 악성 행동 (컴퓨터 보안)으로부터 데이터와 기능을 보호하는 매커니즘이다 -[위키백과(보호링)](https://ko.wikipedia.org/wiki/%EB%B3%B4%ED%98%B8_%EB%A7%81)
[<이미지출처>](https://www.javatpoint.com/dual-mode-operations-in-operating-system)


위 그림을 보면 __권한(privileaged)__에 따라 layer를 나눈것을 확인 할 수 있다. Ring 3에 위치한 Application(User mode)는 Ring 0에 위치한 Kernel(Kernel mode) 보다 적은 권한을 갖고있다._(참고로 Ring 1, 2는 option 이고 Ring 0, 3은 모든 OS가 갖고있다)_

예를들어 CPU자원 혹은 메모리를 사용하는 것은 Ring0의 privileged를 갖고있어야 하기 때문에 User mode 에서는 CPU 자원 및 메모리를 사용하지 못한다.

privileaged 에 따라 접근을 제한 한 이유는 뭘까? 간단한 예를 들어보자 현재 2개의 process가 실행되고 있고, Kernel mode로 전환하지 않고 RAM에 직접 접근 할 수 있다고 가정해보자.


![](https://velog.velcdn.com/images/mingyu/post/a983e3c4-efe4-49af-be81-67d65aab5a21/image.png)

 위 그림 처럼 Application 2개가 실행되어 RAM에 맵핑되어있다.(Virtual memory로 lazy 하게 맵핑되는 경우는 지금 생각하지 않기로 한다.)

![](https://velog.velcdn.com/images/mingyu/post/847d178b-c67c-4772-be2d-6e3341d8fb90/image.png)

Application(1)이 메모리를 더 할당 하기 위해 Applicatoin(2)가 맵핑되어있는 주소값에 자신의 메모리를 맵핑시킨다. 이렇게 되면 Application(1)과 Application(2)가 충돌을 일으킬 수 있다.


이러한 경우(실제로는 여러 문제점 중 하나)를 방지하기 위해 User mode는 메모리 및 CPU자원을 사용하려면 Kernel에 요청하고, Kernel mode에서 __요청이 유효한지__ 확인후 적절한 __Policy에 따라__ 자원을 할당한다.

그렇다면 User가 Kernel 에게 자원을 요청할 수 있도록, Kernel은 Userd에게 Interface를 제공해야한다.

Kernel이 User에게 제공하는 API를 System Call 이라고 한다.

# 2. System Call

> Application의 요청에 따라 kernel 로 접근하기 위해 Kernel이 제공하는 API


 Application은 원하는 커널 서비스에 따라 System Call을 요청한다. 간단히 Application이 printf() 호출해 System Call을 요청하는 과정을 살펴보자.
 * printf()는 system call중 하나인 write()을 호출한다.
 
 ```c
 int main(){
 	
 	printf("System Call Example")
    return -1;
    
      }
  ```
  
1. Application 이 printf()를 호출한다.

2. printf()는 write() System Call을 호출한다.

3. System Call 을 호출하면, Interrupt 가 발생하고 Syscall handler(kerel mode) 를 호출한다.
	* 참고로 System call도 일종의 interrupt이기 때문에 interrupt handler를 호출한다. 그러나 PintOS는 msr 레지스터를 사용해 interrutp handler를 거치지 않고 직접 Syscall handler를 호출한다.(Fast System Call)
    
  4. handler는 syscall number(write()은 10번)를 확인하여 해당 함수 를 호출한다.
  5. kernel mode에서 결과를 반환하고, 다시 User mode로 돌아간다.
  
 
 
위와 같은 과정을 거쳐서 system Call은 실행된다. 하지만 여기서 3  ~ 5에서 의문점이 생길거라고 생각한다.

* Kernel mode로 진입한다는게 어떤 의미인지
* 어떻게 transition 이 일어날때 Application 에 context를 저장하는지
* System Call 결과를 어떻게 User mode에 반환하는지  

결과적으로 또 interrupt frame을 사용한다! 이제 PintOS 코드를 살펴보며 좀 더 자세히 살펴보자. (

# 3. PintOS System Call(transition) 과정

