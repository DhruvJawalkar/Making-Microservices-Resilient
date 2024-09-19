# Circuit Breaker pattern implementation

## Overview
![Circuit breaker pattern states](circuit-breaker-pattern-states.jpg)

## Base project structure
- Modules in project are Accounts (Accounts microservice), Config Server (Spring Cloud Configuration server), Eureka Server (Netflix Eureka server for Service Discovery of microservice in project) and Gateway Server (Spring Cloud API Gateway server for the edge server/ entrypoint into our microservices).   

## Bringing up the services
- Run configserver (Runs on localhost:8071)
- Run eurekaserver (Runs on localhost:8070)
- Run accounts microservice (Runs on localhost:8080)
- Run gatewayserver (Runs on localhost:8072, edge server and entrypoint into our services)

## Sample endpoint to test the pattern
- http://localhost:8080/api/contact-info ("/api/contact-info" endpoint in accounts microservice)

## Implementation
- We'll be adding the circuit breaker at the gatewayserver
- Add 'spring-cloud-starter-circuitbreaker-reactor-resilience4j' dependency in pom.xml
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-reactor-resilience4j</artifactId>
</dependency>
```
- Configure the route to accounts microservice and add the circuit breaker
```java
@Bean
public RouteLocator eazyBankRouteConfig(RouteLocatorBuilder routeLocatorBuilder) {
    return routeLocatorBuilder.routes()
            .route(p -> p
                    .path("/eazybank/accounts/**")
                    .filters( f -> f.rewritePath("/eazybank/accounts/(?<segment>.*)","/${segment}")
                            .addResponseHeader("X-Response-Time", LocalDateTime.now().toString())
                            .circuitBreaker(config -> config.setName("accountsCircuitBreaker")))
                    .uri("lb://ACCOUNTS")).build();
}
```
- Specify the circuit breaker config properties in the application.yml file of the gatewayserver
```yaml
resilience4j.circuitbreaker:
  configs:
    default:
      slidingWindowSize: 4
      permittedNumberOfCallsInHalfOpenState: 2
      failureRateThreshold: 50
      waitDurationInOpenState: 10000
```
- Invoke the endpoint through the gatewayserver 
```text
endpoint url: http://localhost:8072/eazybank/accounts/api/contact-info
```
- View circuit breaker events through actuator url on the gatewayserver
```text
accountsCircuitBreaker events url: 
http://localhost:8072/actuator/circuitbreakerevents/accountsCircuitBreaker
```
```json
{
  "circuitBreakerEvents": [
    {
      "circuitBreakerName": "accountsCircuitBreaker",
      "type": "SUCCESS",
      "creationTime": "2024-09-18T11:33:08.139189700-07:00[America/Los_Angeles]",
      "errorMessage": null,
      "durationInMs": 409,
      "stateTransition": null
    }
  ]
}
```
- Simulate accounts microservice unavailability through adding a breakpoint in the endpoint handler method. Gateway server responds with a Timeout Exception. 
```json
{
    "timestamp": "2024-09-18T18:40:38.550+00:00",
    "path": "/eazybank/accounts/api/contact-info",
    "status": 504,
    "error": "Gateway Timeout",
    "message": "Did not observe any item or terminal signal within 1000ms in 'circuitBreaker' (and no fallback has been configured)",
    "requestId": "e60f04e6-12",
    "trace": "java.util.concurrent.TimeoutException: Did not observe any item or terminal signal within 1000ms in 'circuitBreaker' (and no fallback has been configured)\r\n\tat reactor.core.publisher.FluxTimeout$TimeoutMainSubscriber.handleTimeout(FluxTimeout.java:296)\r\n\tat reactor.core.publisher.FluxTimeout$TimeoutMainSubscriber.doTimeout(FluxTimeout.java:281)\r\n\tat reactor.core.publisher.FluxTimeout$TimeoutTimeoutSubscriber.onNext(FluxTimeout.java:420)\r\n\tat reactor.core.publisher.FluxOnErrorReturn$ReturnSubscriber.onNext(FluxOnErrorReturn.java:162)\r\n\tat reactor.core.publisher.MonoDelay$MonoDelayRunnable.propagateDelay(MonoDelay.java:270)\r\n\tat reactor.core.publisher.MonoDelay$MonoDelayRunnable.run(MonoDelay.java:285)\r\n\tat reactor.core.scheduler.SchedulerTask.call(SchedulerTask.java:68)\r\n\tat reactor.core.scheduler.SchedulerTask.call(SchedulerTask.java:28)\r\n\tat java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)\r\n\tat java.base/java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:304)\r\n\tat java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1136)\r\n\tat java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:635)\r\n\tat java.base/java.lang.Thread.run(Thread.java:840)\r\n"
}
```
- Simulate 2 failure events to trigger circuit breaker to move into OPEN state
```json
{
  "circuitBreakerEvents": [
    ...
    {
      "circuitBreakerName": "accountsCircuitBreaker",
      "type": "ERROR",
      "creationTime": "2024-09-18T11:42:46.514662800-07:00[America/Los_Angeles]",
      "errorMessage": "java.util.concurrent.TimeoutException: Did not observe any item or terminal signal within 1000ms in 'circuitBreaker' (and no fallback has been configured)",
      "durationInMs": 1000,
      "stateTransition": null
    },
    {
      "circuitBreakerName": "accountsCircuitBreaker",
      "type": "FAILURE_RATE_EXCEEDED",
      "creationTime": "2024-09-18T11:42:46.515513200-07:00[America/Los_Angeles]",
      "errorMessage": null,
      "durationInMs": null,
      "stateTransition": null
    },
    {
      "circuitBreakerName": "accountsCircuitBreaker",
      "type": "STATE_TRANSITION",
      "creationTime": "2024-09-18T11:42:46.521503500-07:00[America/Los_Angeles]",
      "errorMessage": null,
      "durationInMs": null,
      "stateTransition": "CLOSED_TO_OPEN"
    }
  ]
}
```
- Wait for 'waitDurationInOpenState' config time to redo the request to move the circuit into HALF_OPEN state and check for failure/ success to move into OPEN or CLOSED states
```json
{
  "circuitBreakerEvents": [
    ...
    {
      "circuitBreakerName": "accountsCircuitBreaker",
      "type": "STATE_TRANSITION",
      "creationTime": "2024-09-18T11:42:46.521503500-07:00[America/Los_Angeles]",
      "errorMessage": null,
      "durationInMs": null,
      "stateTransition": "CLOSED_TO_OPEN"
    },
    {
      "circuitBreakerName": "accountsCircuitBreaker",
      "type": "STATE_TRANSITION",
      "creationTime": "2024-09-18T11:45:48.757567500-07:00[America/Los_Angeles]",
      "errorMessage": null,
      "durationInMs": null,
      "stateTransition": "OPEN_TO_HALF_OPEN"
    },
    {
      "circuitBreakerName": "accountsCircuitBreaker",
      "type": "SUCCESS",
      "creationTime": "2024-09-18T11:45:48.773950900-07:00[America/Los_Angeles]",
      "errorMessage": null,
      "durationInMs": 16,
      "stateTransition": null
    }
  ]
}
```


