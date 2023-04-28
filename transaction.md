# Transactional 의 동작 원리

## Transactional은 proxy로 동작한다

여기서 `proxy` 방식이란? 객체를 직접 참조하는 것이 아니라, <br>
객체를 대행하는 객체를 통해 대상 객체를 접근하는 방식을 말합니다.

### JDK Proxy, CGLib
프록시 객체를 생성하는 방법에 따라 JDK Proxy 와 CGLib를 만들 수 있습니다. <br>
먼저 둘의 가장 다른 점은 `타겟의 어떤 부분을 상속받아서 프록시를 구현하느냐` 입니다. <br>
`JDK Proxy`: 구현된 클래스의 인터페이스를 프록세 객체로 구현하여 코드를 끼워 넣는 방식 <br>
```java
public class ExamDynamicHandler implements InvocationHandler {
    private ExamInterface target; // 타깃 객체에 대한 클래스를 직접 참조하는것이 아닌 Interface를 이용

    public ExamDynamicHandler(ExamInterface target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args)
            throws Throwable {
        // TODO Auto-generated method stub
        // 메소드에 대한 명세, 파라미터등을 가져오는 과정에서 Reflection 사용
        String ret = (String)method.invoke(target, args); //타입 Safe하지 않는 단점이 있다.
        return ret.toUpperCase(); //메소드 기능에 대한 확장
    }
}
```
다음과 같이 invoke 메소드를 오버라이딩 하여 Proxy 위임 기능을 수행하는데, <br>
이 때 메소드에 대한 명세와 파라미터를 가져오는 과정에서 리플렉션을 사용합니다.<br>

CGLib: 타켓 클래스를 상송받아 프록시를 만든다.
```java
// 1. Enhancer 객체를 생성
Enhancer enhancer = new Enhancer();
// 2. setSuperclass() 메소드에 프록시할 클래스 지정
enhancer.setSuperclass(BoardServiceImpl.class);
enhancer.setCallback(NoOp.INSTANCE);
// 3. enhancer.create()로 프록시 생성
Object obj = enhancer.create();
// 4. 프록시를 통해서 간접 접근
BoardServiceImpl boardService = (BoardServiceImpl)obj;
boardService.writePost(postDTO);
```
위에 코드 처럼 상속을 통해 객체가 생성되기에 성능상의 이점을 누릴수 있습니다.<br>
내부적으로 reflection을 사용하지 않습니다.<br>
또한 반드시 인터페이스를 구현하지 않아도 됩니다. <br>
이렇듯 성능 상 이점으로 인해 Spring Boot에서는 CGLib를 사용한 방식을 기본으로 채택하고 있습니다.<br>

<img width="928" alt="스크린샷 2023-04-28 오후 5 06 41" src="https://user-images.githubusercontent.com/80631952/235091594-3ba87faa-e69e-4319-83bb-e13d773a259d.png">
RacingGameService를 디버깅 해보면 다른 객체가 injection 되어있다.

@Transactional이 동작하는 RacingGameService.java
```java
@Service
public class RacingGameService {

    private final CarDao carDao;
    private final GameResultDao gameResultDao;

    public RacingGameService(CarDao carDao, GameResultDao gameResultDao) {
        this.carDao = carDao;
        this.gameResultDao = gameResultDao;
    }

    @Transactional
    public Long addGameResultAndCars(UserInputDto inputDto) {
        RacingGame racingGame = getRacingGame(inputDto);
        TryCount tryCount = new TryCount(inputDto.getCount());

        Long gameResultId = gameResultDao.insert(new GameResultEntity(tryCount.getCount()));
        return gameResultId;
    }
}
```
proxy로 생성된 RacingGameService@28fa06ee.java

```java
@Service
public class RacingGameService extends RacingGameService{

    private final CarDao carDao;
    private final GameResultDao gameResultDao;
    private final EntityManager em;

    @Transactional
    public Long addGameResultAndCars(UserInputDto inputDto) {
        EntityTransaction tx = em.getTransaction(); // EntityTransaction를 받아
        tx.begin(); //사용전

        Long gameResultId = super.createUserListWithTrans();

        tx.commit(); // 사용 후
    }
}
```
정확한 RacingGameService@28fa06ee 의 코드는 아니지만 <br>
다음과 같이 EntityTransaction 을 injection 받아
매소드 사용 전 후에 코드를 동작시키도록 만들어져 있습니다.

그렇기에 실수하기 좋은 두 부분
1. private은 트랜잭션 처리를 할 수 없다.
   @Transactional은 proxy 객체를 이용하여 접근하기 때문에 외부 접근이 불가능한 private 합수에는 
   적용할 수 없습니다. 인텔리제이에서는 다음과 같이 에러표시를 해줍니다.
2. 트랜잭션은 객체 외부에서 처음 진입한 매서드를 기준으로 동작한다.
