---
title: "[수한] 영화 예매 사이트 - 좌석 동시 선택 처리(1)"
author:
  name: 수한
  link: https://github.com/sem1308
date: 2024-06-11 04:15:00 +0900
categories: [Suhan, TIL, Spring]
tags: [Suhan, TIL, Backend, Java, Spring, Test]
---

# 좌석 동시 선택 처리

좌석 예매 방식을 변경하려고 한다.

#### 기존 예매 방법
1. 좌석 여러개 선택
2. 결제하기 버튼 클릭 시 티켓 생성 API 요청
     - 상영일정 번호(schedNum)와 좌석 번호 리스트(seatNumList)를 요청 파라미터로 보냄
     - 예시)
       ```
       {
         schedNum : 1,
         seatNumList : [1,2,3]
       }
       ```
3. 상영일정 번호와 좌석 번호를 통해 해당 상영일정의 좌석이 이미 예매가 된 상태인지 아닌지 검사
   - 이때, 낙관적 락을 걸어 동시성 체크를 함
4. 모든 좌석이 예매가 안되어있다면 티켓 생성

기존 예매 방식은 좌석을 여러개 선택하고 결제하기 버튼을 눌렀음에도 결제 화면으로 넘어가지 못하는 현상이 발생한다. 그러면 어떤 좌석이 이미 예매되었는지 사용자가 직접 확인한 후 그 좌석만 체크 해제하거나 다시 처음부터 좌석을 선택해야하는 번거로움이 발생할 수 있다.

그러므로 **좌석을 선택하는 순간에 동시성 처리**를 하며 가장 빠르게 좌석을 선택한 사람에게 그 좌석을 소유할 수 있도록 함으로써 해당 좌석은 무조건 예매할 수 있다는 신뢰성을 사용자에게 보장하여 불편함을 줄이려고 한다. 

#### 변경될 예매 방법
1. 좌석 선택 시 좌석 선택 API 요청
   - 상영일정_좌석 번호(scheduleSeatNum)을 요청 파라미터로 보냄
     - 기존에는 복합키로 데이터를 검색했지만 그럴 경우 인덱스 검색시 성능이 저하될 우려가 있고 JPA 사용시 복합키를 정의하는 class가 추가로 생기기 때문에 기본키 컬럼을 추가하여 사용하기로 결정
     - 예시)
     ```
     {api-url}/{scheduleSeatNum}
     ``` 
2.  해당 상영일정에 대한 좌석을 select for update로 가져옴
    - ```@Lock(LockModeType.PESSIMISTIC_WRITE)```사용
    - 이러면 다른 트랜잭션에서 해당 row에 대해 read를 하려고 하면 wait를 함
    - 이를 통해 여러 트랜잭션이 동시에 어떤 row를 수정하지 못하도록 할 수 있음
3. 상영일정_좌석의 상태가 AVAILABLE이 아니면 예외 발생
4. 해당 상영일정_좌석의 값 변경
   -  상태를 SELECTED로 변경
   - 선택 시간을 현재 시간으로 변경
5. 성공 여부 반환
6. 좌석 전부 선택 후 결제하기 버튼 클릭
7. 티켓 생성 API 호출 

## 구현 - 좌석 선택
1. 상영일정_좌석에 state column, userNum 추가 및 상태 변경 로직 추가
   -  state (SeatState)
      1. AVAILABLE (사용가능)
      2. SELECTED (선택됨)
      3. RESERVED (예약됨)
      4. BOOKED (결제됨)
    ```java
    @Entity(name = "SCHEDULE_SEAT")
    @AllArgsConstructor()
    @NoArgsConstructor()
    @Getter
    @Builder
    public class ScheduleSeat {
        @Id
        @GeneratedValue(strategy = GenerationType.IDENTITY)
        @Column(name = "schedule_seat_num")
        private Long scheduleSeatNum;

        @Column(name="state", nullable = false, columnDefinition = "VARCHAR(10) DEFAULT 'AVAILABLE'")
        @Enumerated(EnumType.STRING)
        @Builder.Default
        private SeatState state = SeatState.AVAILABLE;

        @Column(name="reserved_date_time")
        private LocalDateTime reservedDateTime;

        /* Foreign Key */
        @ManyToOne(fetch = FetchType.LAZY)
        @JoinColumn(name = "SCHED_NUM", nullable = false)
        private Schedule schedule;

        @ManyToOne(fetch = FetchType.LAZY)
        @JoinColumn(name = "SEAT_NUM", nullable = false)
        private Seat seat;

        @ManyToOne(fetch = FetchType.LAZY)
        @JoinColumn(name = "USER_NUM")
        private User user;

        //=== 비즈니스 로직 ===//
        public void select(){
            this.state = SeatState.SELECTED;
        }

        public void reserve(){
            this.state = SeatState.RESERVED;
            this.reservedDateTime = LocalDateTime.now();
        }

        public void cancel(){
            this.state = SeatState.AVAILABLE;
        }

        public void book(){
            this.state = SeatState.BOOKED;
        }
    }

    ```
2. 상영일정_좌석 레포지토리의 findById를 Lock 추가하여 구현
    ```java
    public interface ScheduleSeatRepository extends JpaRepository<ScheduleSeat, Long> {
        ...

        @Lock(LockModeType.PESSIMISTIC_WRITE)
        @Query("select ss from SCHEDULE_SEAT ss where ss.scheduleSeatNum = :scheduleSeatNum")
        Optional<ScheduleSeat> findByIdForUpdate(@Param("scheduleSeatNum")Long scheduleSeatNum);
    }
    ``` 
3. ScheduleSeatService의 selectScheduleSeat 함수에서 동시성 처리후 좌석 선택
    ```java
    @Service
    @Transactional
    @RequiredArgsConstructor
    public class ScheduleSeatService {
        private final ScheduleSeatRepository scheduleSeatRepo;

        ...

        /*
            동시성 처리 좌석 선택 O
        */
        public ScheduleSeat findScheduleSeatForUpdate(Long scheduleSeatNum) {
            return scheduleSeatRepo.findByIdForUpdate(scheduleSeatNum).orElseThrow(() ->
                new ResourceNotFoundException("번호가 " +scheduleSeatNum+ "인 상영일정_좌석이 없습니다."));
        }


        public synchronized Long selectScheduleSeat(Long scheduleSeatNum){
            // 해당 상영일정에 대한 좌석을 가져옴 (select for update)
            ScheduleSeat scheduleSeat = findScheduleSeatForUpdate(scheduleSeatNum);
            // 상태가 AVAILABLE이 아니면 예외 발생
            if(!scheduleSeat.getState().equals(SeatState.AVAILABLE)){
                throw new IllegalStateException("해당 좌석은 이미 예약된 상태입니다.");
            }
            // 해당 상영일정_좌석의 값 변경
            scheduleSeat.select();
            return scheduleSeat.getScheduleSeatNum();
        }
        
        /*
            테스트를 위한 기능
            동시성 처리 좌석 선택 X
        */
        public ScheduleSeat findScheduleSeat(Long scheduleSeatNum) {
            return scheduleSeatRepo.findById(scheduleSeatNum).orElseThrow(() ->
                new ResourceNotFoundException("번호가 " +scheduleSeatNum+ "인 상영일정_좌석이 없습니다."));
        }

        public synchronized Long selectScheduleSeatWithoutLocking(Long scheduleSeatNum){
            ScheduleSeat scheduleSeat = findScheduleSeat(scheduleSeatNum);
            if(!scheduleSeat.getState().equals(SeatState.AVAILABLE)){
                throw new IllegalStateException("해당 좌석은 이미 예약된 상태입니다.");
            }

            scheduleSeat.select();
            return scheduleSeat.getScheduleSeatNum();
        }

        // 상영일정에 좌석 등록
        public void insertScheduleSeat(Schedule schedule, Screen screen) {
            List<Seat> seatList = screen.getSeats();
            seatList.forEach(seat -> {
                ScheduleSeat ss = ScheduleSeat.builder().schedule(schedule).seat(seat).build();
                scheduleSeatRepo.save(ss);
            });
        }
    }
    ```
4. ScheduleSeatController에서 좌석 선택시 호출될 API 구현
    ```java
    @RestController()
    @RequestMapping("/schedule/seat")
    @RequiredArgsConstructor
    public class ScheduleSeatController {
        private final ScheduleSeatService scheduleSeatService;

        ...

        @GetMapping("/select/{scheduleSeatNum}")
        public ResponseEntity<Long> selectSeat(@PathVariable Long scheduleSeatNum){
            scheduleSeatService.selectScheduleSeat(scheduleSeatNum);
            return ResponseEntity.ok(scheduleSeatNum);
        }

        // test를 위한 api - 동시성 처리 X
        @GetMapping("/select/{scheduleSeatNum}/no")
        public ResponseEntity<Long> selectSeatWithoutLocking(@PathVariable Long scheduleSeatNum){
            scheduleSeatService.selectScheduleSeatWithoutLocking(scheduleSeatNum);
            return ResponseEntity.ok(scheduleSeatNum);
        }
    }
    ```

## 테스트 - 좌석 선택
### ScheduleSeatService 테스트
```java
@SpringBootTest
class ScheduleSeatServiceTest {
    @Autowired
    ScheduleSeatService scheduleSeatService;

    @Autowired
    ScheduleSeatRepository scheduleSeatRepository;

    @Autowired
    ScheduleRepository scheduleRepository;

    @Autowired
    MovieRepository movieRepository;

    @Autowired
    ScreenRepository screenRepository;

    @Autowired
    SeatRepository seatRepository;

    // 픽스처
    Screen screen = Screen.mock();
    Seat seat = Seat.mock(screen);
    Movie movie = Movie.mock();
    Schedule schedule = Schedule.mock(screen,movie);
    ScheduleSeat scheduleSeat;

    @BeforeEach
    public void init(){
        // 상영관 등록
        screenRepository.save(screen);

        // 상영관에 좌석 등록
        seat = seatRepository.save(seat);
        screen.getSeats().add(seat);

        // 영화 등록
        movieRepository.save(movie);

        // 상영일정 등록
        scheduleRepository.save(schedule);

        // 상영일정_좌석 등록
        scheduleSeatService.insertScheduleSeat(schedule,screen);
        scheduleSeat = scheduleSeatService.findScheduleSeat(schedule.getSchedNum(), seat.getSeatNum());
    }

    @AfterEach
    public void destory(){
        scheduleSeatRepository.deleteAll();
        scheduleRepository.deleteAll();
        movieRepository.deleteAll();
        seatRepository.deleteAll();
        screenRepository.deleteAll();
    }

    @DisplayName("selectScheduleSeat : 상영일정_좌석번호로 상영일정_좌석을 성공적으로 선택한다.")
    @Test
    public void selectScheduleSeat() {
        //given
        //when
        scheduleSeatService.selectScheduleSeat(scheduleSeat.getScheduleSeatNum());
        scheduleSeat = scheduleSeatService.findScheduleSeat(scheduleSeat.getScheduleSeatNum());

        //then
        Assertions.assertThat(scheduleSeat.getState()).isEqualTo(SeatState.SELECTED);
    }

    @DisplayName("selectScheduleSeat concurrently : 동시에 여러 사람이 상영일정_좌석번호로 상영일정_좌석을 선택할 때 성공적으로 한 명만 선택된다.")
    @Test
    public void selectScheduleSeatByNumConcurrently() throws Exception {
        //given
        int numberOfThreads = 5;
        ExecutorService executorService = Executors.newFixedThreadPool(numberOfThreads);
        CountDownLatch latch = new CountDownLatch(numberOfThreads);

        AtomicInteger atomicInt = new AtomicInteger(0);
        //when
        //then
        for (int i = 0; i < numberOfThreads; i++) {
            executorService.submit(() -> {
                try {
                    scheduleSeatService.selectScheduleSeat(scheduleSeat.getScheduleSeatNum());
                    System.out.println(Thread.currentThread().getName() + ": 성공!!!");
                    atomicInt.incrementAndGet();
                } catch (Exception e) {
                    System.out.println(e.getMessage());
                    System.out.println(Thread.currentThread().getName() + ": 실패...");
                } finally {
                    latch.countDown();
                }
            });
        }

        latch.await();
        executorService.shutdown();

        System.out.println(atomicInt.get() + "명 좌석 선택 성공!!");
        Assertions.assertThat(atomicInt.get()).isEqualTo(1);
    }

    @DisplayName("selectScheduleSeat concurrently failed: 동시에 여러 사람이 상영일정_좌석번호로 상영일정_좌석을 선택할 때 한 명만 선택되지 않는다.")
    @Test
    public void selectScheduleSeatByNumConcurrentlyFailed() throws Exception {
        //given
        int numberOfThreads = 5;
        ExecutorService executorService = Executors.newFixedThreadPool(numberOfThreads);
        CountDownLatch latch = new CountDownLatch(numberOfThreads);

        AtomicInteger atomicInt = new AtomicInteger(0);
        //when
        //then
        for (int i = 0; i < numberOfThreads; i++) {
            executorService.submit(() -> {
                try {
                    scheduleSeatService.selectScheduleSeatWithoutLocking(scheduleSeat.getScheduleSeatNum());
                    System.out.println(Thread.currentThread().getName() + ": 성공!!!");
                    atomicInt.incrementAndGet();
                } catch (Exception e) {
                    System.out.println(e.getMessage());
                    System.out.println(Thread.currentThread().getName() + ": 실패...");
                } finally {
                    latch.countDown();
                }
            });
        }

        latch.await();
        executorService.shutdown();

        System.out.println(atomicInt.get() + "명 좌석 선택 성공!!");
        Assertions.assertThat(atomicInt.get()).isNotEqualTo(1);
    }

}
```
각 테스트마다 상영일정_좌석 정보가 필요하므로 ```@BeforeEach```를 통해 필요한 픽스처(테스트에서 반복적으로 사용되는 객체)를 준비한다. 이때, 해당 테스트에서만 테스트를 위한 객체가 사용되지 않을 수도 있기 때문에 각 entity class에 mock 함수를 구현하여 테스트를 위한 객체를 쉽게 생성할 수 있도록 하였다.
```
// mock() 예시
@Entity
@AllArgsConstructor
@NoArgsConstructor
@Getter
@Setter
@Builder
public class Movie {
    ...

    // test를 위한 영화 mock object 생성
    public static Movie mock(){
        Random random = new Random();
        int randNum = random.nextInt();
        return Movie.builder()
            .title("title" + randNum)
            .isShowing(Is.N)
            .build();
    }
}

``` 

각 테스트마다 환경이 같아야 하므로 ```@AfterEach```에서 해당 테스트에서 쓰인 데이터를 deleteAll 함수를 사용하여 제거해주었다.

> **@Transactional 사용하지 않은 이유** <br>
> test할 때 scheduleSeatService의 findScheduleSeat가 호출되는데 이때 test 메소드에 @Transactional을 사용한다면 ```@BeforeEach```에서 실행된 정보가 findScheduleSeat 호출 시에 반영이 안되는 문제가 발생한다. 이는 ExecutorService를 사용했기 때문에 발생하는 문제였다. 만약 @Transactional을 사용하게 된다면 selectScheduleSeatByNumConcurrently 테스트가 A라는 transaction에서 실행이 될 것이고 ```@BeforeEach```로 인해 실행되는 init이라는 함수도 A라는 transaction에서 실행이 된다. 하지만 ExecutorService의 submit에서 실행되는 함수는 A transaction이 아니라 독립적인 transaction에서 실행이 되어버린다. 그러므로 submit 함수 안에 있는 selectScheduleSeat에서 findByIdForUpdate라는 함수가 실행될 때 A transaction에서 실행된 정보가 반영이 안되어있으므로 상영일정_좌석을 찾지 못하는 현상이 발생한다. 그러므로 init 함수의 내용이 다른 트랜잭션에서 반영되어야 하므로 rollback을 위해 @Transactional을 사용하지 않고 직접 ```@AfterEach```에서 수동으로 rollback을 해주었다. 이를 통해 init 함수 실행시 autocommit이 되어 즉각적으로 DB에 반영되므로 submit 함수를 실행했을 때 init 함수의 실행 결과를 가져올 수 있다.

위 테스트를 실행하면 전부 success가 나온다.

![test_1](https://github.com/Today-l-Learn/Today-l-Learn.github.io/assets/70849467/7d9ffa72-d0a2-4f63-a522-3f6ae0c46b32)

Lock을 사용하지 않은 경우에는 다음과 같이 모든 사용자가 좌석 선택에 성공하게 되고

![test_2](https://github.com/Today-l-Learn/Today-l-Learn.github.io/assets/70849467/0f8dc090-2646-4ad7-a14c-184138753f86)

Lock을 사용한 경우에는 다음과 같이 한 사용자만 좌석 선택에 성공하게 된다.

![test_3](https://github.com/Today-l-Learn/Today-l-Learn.github.io/assets/70849467/b0959a8d-d7cd-484c-8fd5-7bf9dcc29114)


### ScheduleSeatController 테스트
실제 api요청시에도 정상적으로 동작하는지 테스트해보겠다.
```java
@SpringBootTest
@AutoConfigureMockMvc
class ScheduleSeatControllerTest {
    @Autowired
    MockMvc mockMvc;

    @Autowired
    ScheduleSeatService scheduleSeatService;
    @Autowired
    ScheduleSeatRepository scheduleSeatRepository;

    @Autowired
    ScheduleRepository scheduleRepository;

    @Autowired
    MovieRepository movieRepository;

    @Autowired
    ScreenRepository screenRepository;

    @Autowired
    SeatRepository seatRepository;

    @Autowired
    ObjectMapper objectMapper;

    // 픽스처
    Screen screen = Screen.mock();
    Seat seat = Seat.mock(screen);
    Movie movie = Movie.mock();
    Schedule schedule = Schedule.mock(screen,movie);
    ScheduleSeat scheduleSeat;

    @BeforeEach
    public void init(){
        // 상영관 등록
        screenRepository.save(screen);

        // 상영관에 좌석 등록
        seat = seatRepository.save(seat);
        screen.getSeats().add(seat);

        // 영화 등록
        movieRepository.save(movie);

        // 상영일정 등록
        scheduleRepository.save(schedule);

        // 상영일정_좌석 등록
        scheduleSeatService.insertScheduleSeat(schedule,screen);
        scheduleSeat = scheduleSeatService.findScheduleSeat(schedule.getSchedNum(), seat.getSeatNum());
    }

    @AfterEach
    public void destory(){
        scheduleSeatRepository.deleteAll();
        scheduleRepository.deleteAll();
        movieRepository.deleteAll();
        seatRepository.deleteAll();
        screenRepository.deleteAll();
    }

    @DisplayName("selectScheduleSeat : 상영일정_좌석번호로 상영일정_좌석을 성공적으로 선택한다.")
    @Test
    public void selectScheduleSeatByNum() throws Exception {
        //given
        String url = "/schedule/seat/select/"+scheduleSeat.getScheduleSeatNum()+"/no";

        //when
        MvcResult result = mockMvc.perform(MockMvcRequestBuilders.get(url)).andReturn();

        scheduleSeat = scheduleSeatService.findScheduleSeat(scheduleSeat.getScheduleSeatNum());

        //then
        Assertions.assertThat(result.getResponse().getStatus()).isEqualTo(HttpStatus.OK.value());
        Assertions.assertThat(scheduleSeat.getState()).isEqualTo(SeatState.SELECTED);
    }


    @DisplayName("selectScheduleSeat concurrently : 동시에 여러 사람이 상영일정_좌석번호로 상영일정_좌석을 선택할 때 성공적으로 한 명만 선택된다.")
    @Test
    public void selectScheduleSeatByNumConcurrently() throws Exception {
        //given
        int numberOfThreads = 5;
        ExecutorService executorService = Executors.newFixedThreadPool(numberOfThreads);
        CountDownLatch latch = new CountDownLatch(numberOfThreads);

        String url = "/schedule/seat/select/"+scheduleSeat.getScheduleSeatNum();

        AtomicInteger atomicInt = new AtomicInteger(0);
        //when
        //then

        for (int i = 0; i < numberOfThreads; i++) {
            executorService.submit(() -> {
                try {
                    MvcResult result = mockMvc.perform(MockMvcRequestBuilders.get(url))
                        .andReturn();
                    System.out.println(Thread.currentThread().getName() + ": " + result.getResponse().getContentAsString());
                    if(result.getResponse().getStatus() == HttpStatus.OK.value()){
                        atomicInt.incrementAndGet();
                        System.out.println("[성공!!!!]");
                    }else{
                        System.out.println("[실패....]");
                    }
                } catch (Exception e) {
                    throw new RuntimeException(e);
                } finally {
                    latch.countDown();
                }
            });
        }

        latch.await();
        executorService.shutdown();

        Assertions.assertThat(atomicInt.get()).isEqualTo(1);
    }

    @DisplayName("selectScheduleSeat concurrently failed : 동시에 여러 사람이 상영일정_좌석번호로 상영일정_좌석을 선택할 때 한 명만 선택되지 않는다.")
    @Test
    public void selectScheduleSeatByNumConcurrentlyFailed() throws Exception {
        //given
        int numberOfThreads = 5;
        ExecutorService executorService = Executors.newFixedThreadPool(numberOfThreads);
        CountDownLatch latch = new CountDownLatch(numberOfThreads);

        String url = "/schedule/seat/select/"+scheduleSeat.getScheduleSeatNum()+"/no";

        AtomicInteger atomicInt = new AtomicInteger(0);
        //when
        //then

        for (int i = 0; i < numberOfThreads; i++) {
            executorService.submit(() -> {
                try {
                    MvcResult result = mockMvc.perform(MockMvcRequestBuilders.get(url)).andReturn();

                    System.out.println(Thread.currentThread().getName() + ": " + result.getResponse().getContentAsString());
                    if(result.getResponse().getStatus() == HttpStatus.OK.value()){
                        atomicInt.incrementAndGet();
                        System.out.println("[성공!!!!]");
                    }else{
                        System.out.println("[실패....]");
                    }
                } catch (Exception e) {
                    throw new RuntimeException(e);
                } finally {
                    latch.countDown();
                }
            });
        }

        latch.await();
        executorService.shutdown();

        Assertions.assertThat(atomicInt.get()).isNotEqualTo(1);
    }
}
```

service test와 비슷하지만 controller test시에는 MockMVC를 사용하여 postman과 같은 tool을 사용하지 않고 test할 수 있도록 구현하였다.

테스트 실행 결과는 다음과 같다.

![test_4](https://github.com/Today-l-Learn/Today-l-Learn.github.io/assets/70849467/c884310e-da10-402b-a16c-dc36155d67ce)

## 배운점
동시성 처리 test를 실제로 여러 컴퓨터에서 api를 요청하지 않아도 ExecutorService라는 클래스를 통해 테스트할 수 있다는 것을 알게되었고 동시성 처리시에 Lock을 걸어 처리하면 실제로 잘 동작한다는 것을 확인할 수 있었다.

multithread를 test할 때는 @Transactional을 사용할 때 주의해서 사용해야 한다는 것을 깨달았다. 만약 init 함수에서 db에 데이터를 등록하고 그 데이터를 우리가 테스트하려는 함수에서 select해야 한다면, @Transactional을 사용하지 않고 autocommit을 true로 하여 바로 반영하게 한 후 마지막에 직접 deleteAll과 같은 메소드 호출을 통해 수동으로 rollback을 해야 한다는 것을 배웠다. 
