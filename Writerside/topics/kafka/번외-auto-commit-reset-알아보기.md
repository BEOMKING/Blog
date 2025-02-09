# 번외. auto.commit.reset 알아보기

`auto.commit.reset` 옵션에 대해 좀 잘못 이해하고 있었던 게 있었다.

예를 들어, 오프셋 9까지 레코드를 커밋한 상황에서 컨슈머에 문제가 생겨서 컨슈머가 죽었고 (레코드 보관 주기가 만료되지 않아 레코드는 사라지지 않았다고 가정) 컨슈머를 재시작하기까지 오프셋 14까지 레코드가 추가되었다고 가정하자.

내가 잘못 이해한 내용은 다음과 같다.

위 상황에서 만약 `earliest`로 되어있다면 맨 처음 오프셋인 1부터 읽어오게 된다. 10 ~ 14까지의 레코드를 정상적으로 처리할 수 있지만 1 ~ 9까지의 레코드를 재처리하게 될 것이다.

`latest`로 설정되어 있다면 가장 마지막 오프셋인 14부터 읽어오게 된다. 중복이 발생하지는 않지만 10 ~ 13까지의 레코드를 처리하지 못하게 된다.

라고 이해를 했었는데 실제로 동작을 테스트해보니까 둘 다 10 ~ 14의 레코드를 가져오고 있었다.

## 그 이유는?

조금 생각해 보면 당연한 결과이다. 오프셋 정보는 카프카 브로커에 저장되어 있고 컨슈머가 재시작하면 브로커에 저장된 오프셋 정보를 가져오게 된다.

따라서, 컨슈머가 다시 실행한다고 해도 저장된 브로커에 저장된 오프셋은 9이기 때문에 10부터 읽어오게 되는 것이다.

공식 문서를 보면 다음과 같이 나와있다.

> What to do when there is no initial offset in Kafka or if the current offset does not exist any more on the server (e.g. because that data has been deleted):
>
> earliest: automatically reset the offset to the earliest offset  
> latest: automatically reset the offset to the latest offset  
> none: throw exception to the consumer if no previous offset is found for the consumer's group  
> anything else: throw exception to the consumer.
>
> Note that altering partition numbers while setting this config to latest may cause message delivery loss since producers could start to send messages to newly added partitions (i.e. no initial offsets exist yet) before consumers reset their offsets.

나는 이 문서의 `when there is no initial offset in Kafka or if the current offset does not exist any more on the server (e.g. because that data has been deleted):` 부분이

컨슈머의 재시작 시점도 포함하는 것으로 이해했지만 카프카 서버에 current offset이 존재하기 때문에 위 부분에 해당하지 않는다.

새로운 컨슈머 그룹이 생성되는 시점 같은 상황을 의미하는 것 같다.

이에 대한 테스트를 해보았다.

## 테스트
<img src="../../images/kafka/initial%20offset.png" alt="img" style="zoom:60%;" />

모든 테스트는 위 그룹에 5번의 메시지를 전송하고 컨슈머를 종료시키고 5번의 메시지를 추가로 전송하고 재시작하는 방식으로 진행했다.

리스너는 다음과 같이 구현했다.

```java
@Slf4j
@Component
public class ExampleConsumer {
    @KafkaListener(
            topics = "${kafka.consumers.example.topic}",
            containerFactory = "exampleKafkaListenerContainerFactory"
    )
    public void consume(final String message, final ConsumerRecordMetadata metadata) {
        log.info("Offset : {}\n Message : {}", metadata.offset(), message);
    }
}
```

### latest & Same Consumer Group

실행 후 5개의 메시지를 전송한다.

<img src="../../images/kafka/initial%20message.png" alt="img" style="zoom:60%;" />

여기서 종료 시키고 5번의 메시지를 추가로 전송하고 다시 컨슈머를 실행했다.

<img src="../../images/kafka/latest%20same%20group.png" alt="img" style="zoom:60%;" />

오프셋 5번부터의 메시지를 모두 가져온 것을 확인할 수 있다.

### latest & Another Consumer Group

group-1 실행 후 5개의 메시지를 전송한다.

<img src="../../images/kafka/latest%20another%20group%20init.png" alt="img" style="zoom:60%;" />

추가적으로 5개의 메시지를 더 보내고 그룹을 group-2 로 바꾸어 재시작한다.

<img src="../../images/kafka/latest%20another%20group%20after.png" alt="img" style="zoom:60%;" />

아무런 메시지도 가져오지 않아서 임의로 하나를 추가로 전송했다.

재시작 시점의 오프셋은 9이기 때문에 생각했던 대로 10부터 가져오는 것을 확인할 수 있다.

그리고 맨 위 로그를 보면 `Found no committed offset for partition` 커밋된 오프셋이 없다고 나온다.

### earliest & Same Consumer Group

설정을 earliest로 변경하고 컨슈머를 재시작했다.

```yaml
    example:
      bootstrap-servers: localhost:9092
      topic: example
      group-id: example-1
      auto-offset-reset: earliest
```

group-1 실행 후 5개의 메시지를 전송한다.

종료 시키고 5번의 메시지를 추가로 전송하고 다시 컨슈머를 실행했다.

<img src="../../images/kafka/earliest%20same%20group%20after.png" alt="img" style="zoom:60%;" />

커밋된 오프셋이 있기 때문에 5번부터 가져오는 것을 확인할 수 있다.

### earliest & Another Consumer Group

group-1 실행 후 5개의 메시지를 전송한다.

추가적으로 5개의 메시지를 더 보내고 그룹을 group-2 로 바꾸어 재시작한다.

<img src="../../images/kafka/earliest%20another%20group%20after.png" alt="img" style="zoom:60%;" />

커밋된 오프셋이 없고 시작 시점에 바로 오프셋 0부터 가져오는 것을 확인할 수 있다.

## 결론

`auto.offset.reset` 옵션이 어떤 차이가 있는지 직접 테스트를 통해 알아보았다.

오프셋 정보는 카프카 브로커에 저장되어있다는 것을 다시 복기하게 되었고 새로운 그룹이 구독하는 상황 같이 오프셋 정보가 없을 때 `auto.offset.reset` 옵션이 의미가 있다는 것을 알 수 있었다.