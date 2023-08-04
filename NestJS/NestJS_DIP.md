이번 포스팅에서 DIP/OCP 원칙을 공부하고 팀 프로젝트에 적용했던 기록을 남기려고한다. 특히 NestJS에 Dependency Injection 활용 방안을 정리하고자한다!

* 예시는 typescript 로 작성되었습니다.

* 목차
	1. DIP? OCP?
    2. NestJS DI 사용하기
    3. 반드시 DIP패턴을 지켜야 하나?


# 1. DIP? OCP?
> ### OCP(Open/closed prinsiple): SW는 확장에는 열려있으나 변경에는 닫혀 있어야 한다.   
> ### DIP(Dependecy inversion principle): 상위 수준의 모듈은 하위 수준의 모듈에 의존하면 안된다.

위는 DIP 와 OCP 에 사전적인 정의이다. 그러나 이렇게 봐서는 이해가 가지 않는다. 좀더 단순하게 바꾸자면 
* OCP는 __코드를 추가(확장)는 허용하나 변경하는 것은 허용하지않는다.__ 
* DIP 는 __상위 클래스(상위 수준 모듈)가 구현체(하위 수준 모듈)에 직접 의존하지 않는다.__

이 의미는 어떤 객체의 변경 이미 사용중인 객체에 영향을 주지 않기 위해(OCP) 객체간의 결합을 느슨하게(DIP) 하는 것이다.

예시를 한번 살펴보자.

## 코드 예시

## 1) 문제상황

``` typescript
class Circle {
    private radius: number;

    constructor(radius: number) {
        this.radius = radius;
    }

    public calculateArea(): number {
        return Math.PI * Math.pow(this.radius, 2);
    }
}

class AreaCalculator {
    private circle: Circle = new Circle(5);

    public calculateArea() {
        return this.circle.calculateArea();
    }
}
```

원의 넓이를 구하는 코드이다. 언뜻 보면 문제가 없어보이지만, 문제는 코드 변경시 일어나게 된다. 만약 여기서 원이 아닌 직사각형 넓이를 구해야하는 상황이 발생했다고 가정해보자. 그렇다면 코드는
다음과 같이 변경되어야 한다.

``` typescript
class Rectangle {
    private width: number;
    private height: number;

    constructor(width: number, height: number) {
        this.width = width;
        this.height = heightl
    }

    calculateArea(): number {
        return this.width * this.height;
    }
}

class AreaCalculator {
    //private circle: Circle = new Circle(5); 
    private rectangle: Rectangle = new Rectangle(4, 5);
    public calculateArea() {
        //return this.circle.calculateArea();
        return this.rectangle.calculateArea
    }
}
```


자 원에서 사각형넓이를 구하는 코드로 __변경__ 하기 위해 AreaCalculator가 의존하던 Circle 을 전부 Rectangle 로 의존하게끔 바꿔야한다. 실제 프로젝트에서 훨씬 복잡한 코드에서 이러한 변경을 하나하나 바꿔준다는것은 너무 고달픈 일일 것이다.


## 2) 해결방법
![](https://velog.velcdn.com/images/mingyu/post/f9f8abea-49e0-4b52-b5fb-177cf09808c3/image.png)



위 같은 코드의 문젠점은 결국 AreaCalculator 가 하위모듈(Circle)에 의존하여 너무 강하게 결합된것이 원인이다. 

AreaCalcuator는 도형의 넓이를 구하는 calculateArea() 메소드를 포함하고있다. 즉 어떤 도형이든 calculateArea() 메소드를 실행해서 넓이를 구하기만 하면된다. 그렇다면 여러 도형들의 넓이를 추상화(Abstraction) 할 수 있지 않을까?

다이어그램으로 다시 나타내보면 다음과 같다.

![](https://velog.velcdn.com/images/mingyu/post/d43c7e8b-a26f-4ad6-8c25-adc406d8a492/image.png)

즉 AreaCalculate은 이제 Shape interface에 의존하기 때문에 도형에 변경은 관심사가 아니다.
interface에 의존함으로써 __느슨하게 결합__ 할수 있게된다.
```typescript
interface Shape {
    calculateArea(): number;
}

class Circle implements Shape {
    private radius: number;

    constructor(radius: number) {
        this.radius = radius;
    }

    public calculateArea(): number {
        return Math.PI * Math.pow(this.radius, 2);
    }
}

class Rectangle implements Shape {
    private width: number;
    private height: number;

    constructor(width: number, height: number) {
        this.width = width;
        this.height = height;
    }

    public calculateArea(): number {
        return this.width * this.height;
    }
}

class AreaCalculator {
    constructor(private shape: Shape) {}

    public calculateArea() {
        return this.shape.calculateArea();
    }
}
```

여러 도형에 대한 calculateArea() 메소드를 abstraction 한 __Shape Interface__ 를 작성하였다.
그리고 AreaCalculator를 Shape에 의존하도록 변경하였다. 

이제 AreaCalculator는 도형이 변경되도 코드를 변경하지 않아도 된다! 단지 새로운 도형의 넓이를 구하고 싶으면 Class 를 추가 하면 된다.(OCP)

하지만 아직 문제가있다. Shape 은 interface이다. 즉 구현체가 아니므로 new() 를 할수가 없다.
그렇기 때문에 __외부에서 의존성(Dependency)을 주입(Injection) 해줘야한다__


```typescript
class Application{

    public app():void{
        const rectangle = new Rectangle(5, 10);
        const areaCalculatorForRectangle = new AreaCalculator(rectangle);
        console.log(areaCalculatorForRectangle.calculateArea());

        const circle = new Circle(5);
        const areaCalculatorForCircle = new AreaCalculator(circle);
        console.log(areaCalculatorForCircle.calculateArea());
    }
}
```

즉 이런식으로 __외부__ 에서 의존성을 관리를 해줘야한다. 그리고 NestJS는 이러한 DI를 편하게 관리하는 기능을 제공한다.


# 2. NestJS DI 적용하기

직접 진행한 프로젝트인 Singing-Runner에서 적용한 내용을 바탕으로 설명 하고자 한다. 

Singing-Runner을 잠깐 소개하자면 노래와 레이싱을 접목한 게임이다. 매치메이킹을 통해 만난 상대와 레이싱을 하는데, 여러 아이템을 통해 상대에게 공격/방어 할수 있다.

## 1) 문제 상황
프로젝트를 진행할 당시 정말 짧은 기간이 주여졌고, 기획이 정말 많이 변경되었다. 특히 중간발표는 어느정도 프로젝트를 구현해서 시연까지 해야했다. 

하지만 당시 기획상 정해지지 않았던 것이 몇가지 있었는데 그 중 하나는 매치메이킹 이었다.

>  매치메이킹: 유저 점수(티어)에 따라 매치메이킹이 되야하지만, 아직 로그인 시스템이 완성되지 않았다.

즉 기획이 여러번 바뀔것으로 예상되었고, 나는 공부한 Dependency Injection을 활용해 보기로 했다. 

## 2) 해결 방법

'매치 메이킹'이 구현해야할 요구사항은 다음과 같았다.
* 유저가 매치메이킹을 요청하면 유저를 대키큐에 삽입할것(joinQueue)
* 유저가 매치메이킹을 취소하면 유저를 대키큐에서 제거할것(leaveQueue)
* 매치메이킹을 승락했으나, 다른 유저가 매치를 취소할 경우 승락한 유저를 대기큐에서 우선순위로 둘것(joinQueueAtFront)
* 매치메이킹이 조건이 만족했는지 확인할 수 있을것(isQueueReady)
* 모든 유저가 메치메이킹을 승락하면, 유저들을 대키큐에서 제거할것(getAvailableUsers)

즉, 이러한 요구사항은 매치 메이킹 정책에 상관없이 '매치메이킹'이 제공해야할 동작이다. 먼저 위 요구사항에 따라 interface를 작성하였다. 

```typescript
export interface MatchMakingPolicy {
  joinQueue(userGameDto: UserGameDto);
  joinQueueAtFront(userGameDto: UserGameDto);
  leaveQueue(userId: string);
  isQueueReady(userGameDto: UserGameDto): boolean;
  getAvailableUsers(userGameDto: UserGameDto): Array<UserGameDto>;
}
```
그리고 아직 로그인시스템이 완성되지 않았기때문에 발표 시연을 하기 위해 간단하게 매치메이킹을 누른 순서대로 큐에들어가고, 먼저 모인 3명이 매치메이킹 되는 간단한 정책을 구현했다,


```typescript
export class SimpleMatchMaking implements MatchMakingPolicy {
  private readyQueue: Array<UserGameDto> = [];

  public joinQueue(userGameDto: UserGameDto) {
    this.readyQueue.push(userGameDto);
  }
  public joinQueueAtFront(UserGameDto: UserGameDto) {
    this.readyQueue.unshift(UserGameDto);
  }
  public leaveQueue(userId: string) {
    this.readyQueue = this.readyQueue.filter(
      (userInQueue) => userInQueue.getUserMatchDto().userId !== userId
    );
  }

  public isQueueReady(): boolean {
    return this.readyQueue.length >= 2;
  }

  public getAvailableUsers(): UserGameDto[] {
    return this.readyQueue.splice(0, 2);
  }
}
```
> 참고로 당시에 typescript 에 대한 지식이 부족해 Array가 dequeue와 같은 구현인줄알고 그냥 Array를 사용했다.

그리고 Module을 설정해야하는데

```typescript
// game.module.ts 
@Module({
  imports: [
    UserModule,
    SongModule,
    SocialModule,
  ],
  providers: [
    {
      provide: "MatchMakingPolicy",
      useClass: SimpleMatchMaking, 
    },
    GameGateway,
  ],
  exports: [GameReplayService],
})
export class GameModule {}
```

위코드는 game module 의 일부이다. MatchMakingPolicy를 포함하는 module에다가 'provider'를 등록해야한다.

provider를 등록하면 NestJS는 @injectable 로 사용한 코드의 의존성을 관리해준다.

여기서 interface에 Dependency Injection 할때 다른점은 바로 토큰을 사용해야한다는것이다.
```typescript
    {
      provide: "MatchMakingPolicy",
      useClass: SimpleMatchMaking, 
    },
```
위처럼 provide에 문자열인 "MatchMakingPolicy" 이 토큰인데, 이제 위에 토큰을 사용하면 useClass에 'SimpleMatchMaking' 을 주입시킬 수 있다는 의미이다.

이제 MatchMakingPolicy를 의존하는 MatchService에다가 다음과 같이 주입한다.


```typescript
@Injectable()
export class MatchService {
  constructor(
    @Inject("MatchMakingPolicy")
    private matchMakingPolicy: MatchMakingPolicy
  ) {}
  `
  `
  `
}
  ```

여기서 의존성 주입을 사용하기 위해 @injectable() 데코레이터를 달아준다.
이 데코레이터를 사용하면, 해당 클래스는 다르 클래스에서 주입 받아 사용 할수 있게 된다.

@inject 를 사용하고 괄호안에 토큰을 넣어주면 matchMakingPoliccy interface에 useclass로 등록된 SimpleMatchMaking을 주입할 수 있다.

### MatchMakingPolicy 변경하기

중간 발표이후 매치메이킹 정책이 같은 점수에 사람끼리 만날 수 있도록 하는 것으로 기획이 확정 되었다.

즉 다시 MMRMatchPolicy를 구현했다.
```typescript
export class MMRMatchPolicy implements MatchMakingPolicy {
  private tierQueueMap: Map<UserMatchTier, UserGameDto[]> = new Map();
  constructor() {
    this.tierQueueMap.set(UserMatchTier.BRONZE, []);
    this.tierQueueMap.set(UserMatchTier.SILVER, []);
    this.tierQueueMap.set(UserMatchTier.GOLD, []);
    this.tierQueueMap.set(UserMatchTier.PLATINUM, []);
    this.tierQueueMap.set(UserMatchTier.DIAMOND, []);
  }
  public joinQueue(userGameDto: UserGameDto) {
    const userTier: UserMatchTier = this.transformMMRtoTier(
      userGameDto.getUserMatchDto().userMmr
    );
    if (this.tierQueueMap.get(userTier)?.includes(userGameDto)) {
      this.leaveQueue(userGameDto.getUserMatchDto().userId);
    }
    this.tierQueueMap.get(userTier)?.push(userGameDto);
  }

  public joinQueueAtFront(userGameDto: UserGameDto) {
    const userTier: UserMatchTier = this.transformMMRtoTier(
      userGameDto.getUserMatchDto().userMmr
    );
    this.tierQueueMap.get(userTier)?.unshift(userGameDto);
  }

  public leaveQueue(userId: string) {
    for (const key of this.tierQueueMap.keys()) {
      let usersInQueue = this.tierQueueMap.get(key);

      if (usersInQueue !== undefined) {
        usersInQueue = usersInQueue.filter(
          (userInQueue) => userInQueue.getUserMatchDto().userId !== userId
        );

        this.tierQueueMap.set(key, usersInQueue);
      }
    }
  }

  public isQueueReady(userGameDto: UserGameDto): boolean {
    const userTier: UserMatchTier = this.transformMMRtoTier(
      userGameDto.getUserMatchDto().userMmr
    );
    const readyQueue: UserGameDto[] | undefined =
      this.tierQueueMap.get(userTier);
    if (readyQueue === undefined) {
      return false;
    }
    for (const readyUser of readyQueue) {
      if (
        readyUser.getUserMatchDto().userId ===
        userGameDto.getUserMatchDto().userId
      ) {
        this.leaveQueue(readyUser.getUserMatchDto().userId);
        return false;
      }
    }
    if (readyQueue.length >= 2) {
      return true;
    }
    return false;
  }

  public getAvailableUsers(userGameDto: UserGameDto): UserGameDto[] {
    if (
      !userGameDto ||
      !userGameDto.getUserMatchDto() ||
      userGameDto.getUserMatchDto().userMmr == null
    ) {
      throw new HttpException(
        "Invalid userGameDto or userMMR",
        HttpStatus.BAD_REQUEST
      );
    }

    const userMMR = userGameDto.getUserMatchDto().userMmr;
    const userTier: UserMatchTier = this.transformMMRtoTier(userMMR);

    const matchQueue = this.tierQueueMap.get(userTier);
    if (!matchQueue || matchQueue.length < 2) {
      throw new HttpException(
        "Invalid userGameDto or userMMR",
        HttpStatus.BAD_REQUEST
      );
    }

    return matchQueue.splice(0, 2);
  }
}
```

이후에 game module 설정을 다음과 같이 변경해줬다.
```typescript
    {
      provide: "MatchMakingPolicy",
      useClass: MMRMatchPolicy,
    },
```

즉 다른 코드를 건드릴것없이 useclass를 simple matchmakingPolicy 에서 MMRMatchPolicy로 변경하기만 하면된다.

이렇듯 NestJS는 의존성을 관리해주기 떄문에 편리하게 DI을 사용할 수 있다!

# 3. 반드시 DIP 패턴을 지켜야하나?

이전 예시와 같은 경우에 나는 DI를 사용한 느슨한 결합으로 효과를 봤다고 생각한다. 그러나 프로젝트를 진행하면서 매번 interface를 만들어 추상화하고 DI를 하는 패턴을 지키지는 못했다.

1. 먼저 시간이 별로 없었다. 나는 이 프로젝트를 진행할때, NestJS와 typescript를 처음 다뤄봤고. 매우 짧은기간이 주어졌다. 그래서 모든 class를 interface화 하지 못했다.

2. 변경되지 않을 코드를 꼭 Interface로 만들어야하나? 에 대한 의문을 갖고있었다.
 

이러한 궁금증을 해결하기 위해 검색을 하다. 배달의 민족 기술이사인 김영한님이 관련질문에 대답한 내용이 있었다.


![](https://velog.velcdn.com/images/mingyu/post/2f0d2a8e-9404-4196-8d15-10c977681c89/image.png)
![](https://velog.velcdn.com/images/mingyu/post/7bc953e6-a495-4852-b6f1-80afa61c22da/image.png)


즉 김영한님 답변을 토대로 생각해보면, 

1. 변경되지 않을 코드라는 타당한 이유가 있을것
    *  판단을 위해 경험에 영역이 필요한것 같기도 하다.

2. 시간이 부족했다면, 이후 리팩토링을 통해 필요할때 인터페이스를 도입

이러한 기준을 갖고 판단을 해야할 것 같다.

즉 모든 패턴은 결국 내가 사용하기 나름! 엔지니어는 늘 Best 상황을 기대해서는 안된다. 절대적인 바이블은 없다는것을 늘 기억하자.


