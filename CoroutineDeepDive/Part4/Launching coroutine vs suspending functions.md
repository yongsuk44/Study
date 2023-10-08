# Launching coroutine vs suspending functions

다수의 작업을 동시에 수행하는 상황에서 일반 함수와 일시 중지 함수, 이 2가지 방법을 고려할 수 있습니다.

```kotlin
class NotificationsSender(
    private val client: NotificationsClient,
    private val notificationScope: CoroutineScope
) {
    fun sendNotifications(notifications: List<Notification>) {
        notifications.forEach { notification ->
            notificationScope.launch {
                client.send(notification)
            }
        }
    }
}

class NotificationsSender(
    private val client: NotificationsClient
) {
    suspend fun sendNotifications(notifications: List<Notification>) = supervisorScope {
        notifications.forEach { notification ->
            launch { 
                client.send(notification)
            } 
        }
    }
}
```

일반 함수와 일시 중지 함수는 작업을 동시에 처리하는 방식에 몇 가지 본질적인 차이가 있습니다.

### 일반 함수

일반 함수는 코루틴을 시작할 때 스코프 객체가 필요하며, 코루틴의 완료를 기다리지 않습니다.   
이로 인해 실행 시간은 매우 빠를 수 있습니다.

또한 외부 스코프에서 시작된 코루틴의 예외는 해당 스코프에서 처리되며, 대부분 출력되고 무시됩니다.  
마지막으로 이러한 코루틴은 스코프 객체로부터 컨텍스트를 상속받아, 코루틴을 취소하려면 스코프를 취소해야 합니다.

### 일시 중지 함수

일시 중지 함수는 모든 코루틴이 완료될 때까지 종료되지 않는다는 점에서 일반 함수와 다르며 이는 동기화에 있어 중요한 역할을 합니다.  
또한 부모 코루틴을 취소하지 않고 예외를 던지거나 무시할 수 있어 일시 중지 함수는 독립적으로 동작이 가능하고, 예외 처리에 있어서도 더욱 유연합니다.
