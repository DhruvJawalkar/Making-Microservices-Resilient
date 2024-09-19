# Inter service resiliency with Spring Cloud OpenFeign implementation

## Overview
![Circuit breaker pattern states](circuit-breaker-pattern-states.jpg)

## Base project structure
- Modules in project are Accounts (Accounts microservice), Cards (Cards microservice), Loans (Loans microservice), Config Server (Spring Cloud Configuration server), Eureka Server (Netflix Eureka server for Service Discovery of microservice in project) and Gateway Server (Spring Cloud API Gateway server for the edge server/ entrypoint into our microservices).   

## Bringing up the services
- Run configserver (Runs on localhost:8071)
- Run eurekaserver (Runs on localhost:8070)
- Run accounts microservice (Runs on localhost:8080)
- Run cards microservice (Runs on localhost:9000)
- Run loans microservice (Runs on localhost:8090)
- Run gatewayserver (Runs on localhost:8072, edge server and entrypoint into our services)

## Sample endpoint to test inter service resiliency
- We'll be calling the http://localhost:8072/eazybank/accounts/api/fetchCustomerDetails?mobileNumber=4354437687 endpoint
- The gatewayserver redirects the call to the accounts microservice and accounts microservice in turn composes a response from local customer data and for fetching their card info makes a call to the Cards microservice
- We aim to make the inter service call from Accounts microservice to Cards microservice resilient to failures and implement a fallback method

## Implementation
- We'll be adding the circuit breaker and openfeign client at the Accounts microservice
- Add 'spring-cloud-starter-circuitbreaker-resilience4j' and 'spring-cloud-starter-openfeign' dependency in pom.xml
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```
- Enable Feign support on the project and specify the circuit breaker config properties in the application.yml file 
```yaml
spring:
  cloud:
    openfeign:
      circuitbreaker:
        enabled: true

resilience4j.circuitbreaker:
  configs:
    default:
      slidingWindowSize: 4
      permittedNumberOfCallsInHalfOpenState: 2
      failureRateThreshold: 50
      waitDurationInOpenState: 10000
```
- Enable feign clients on the application
```java
//AccountsApplication.java file
@EnableFeignClients
```
- Specify FeignClient interface and implementation class for it to be the client to communicate with Cards microservice.
```java
//CardsFeignClient.java
@FeignClient(name="cards", fallback = CardsFallback.class)
public interface CardsFeignClient {
    @GetMapping(value = "/api/fetch",consumes = "application/json")
    public ResponseEntity<CardsDto> fetchCardDetails(@RequestHeader("eazybank-correlation-id")
                                                     String correlationId, @RequestParam String mobileNumber);
}

//CardsFallback.java
@Component
public class CardsFallback implements CardsFeignClient{
    @Override
    public ResponseEntity<CardsDto> fetchCardDetails(String correlationId, String mobileNumber) {
        return null;
    }
}
```
- Update reference in the service layer on the fetchCustomerDetails method.
```java
//CustomersServiceImpl.java
ResponseEntity<CardsDto> cardsDtoResponseEntity = cardsFeignClient.fetchCardDetails(correlationId, mobileNumber);
if(null != cardsDtoResponseEntity) {
    customerDetailsDto.setCardsDto(cardsDtoResponseEntity.getBody());
}
```
- This implementation achieves the behavior of setting the card details as null on cards microservice failure else fills it completely with the details from its response. 
```json
JSON response on successful composition
{
  "name": "Madan Reddy",
  "email": "tutor@eazybytes",
  "mobileNumber": "4354437687",
  "accountsDto": {
    "accountNumber": 1831744132,
    "accountType": "Savings",
    "branchAddress": "123 Main Street, New York"
  },
  "loansDto": {
    "mobileNumber": "4354437687",
    "loanNumber": "100230056899",
    "loanType": "Home Loan",
    "totalLoan": 100000,
    "amountPaid": 0,
    "outstandingAmount": 100000
  },
  "cardsDto": {
    "mobileNumber": "4354437687",
    "cardNumber": "100878775911",
    "cardType": "Credit Card",
    "totalLimit": 100000,
    "amountUsed": 0,
    "availableAmount": 100000
  }
}

JSON response when Cards microservice is down (consequence of OpenFeign client calling the fallback method when circuit is in OPEN state)
{
  "name": "Madan Reddy",
  "email": "tutor@eazybytes",
  "mobileNumber": "4354437687",
  "accountsDto": {
    "accountNumber": 1831744132,
    "accountType": "Savings",
    "branchAddress": "123 Main Street, New York"
  },
  "loansDto": {
    "mobileNumber": "4354437687",
    "loanNumber": "100230056899",
    "loanType": "Home Loan",
    "totalLoan": 100000,
    "amountPaid": 0,
    "outstandingAmount": 100000
  },
  "cardsDto": null
}
```

- View circuit breaker events associated with this client 
```text
circuit breaker name: CardsFeignClientfetchCardDetailsStringString
events url : http://localhost:8080/actuator/circuitbreakerevents/CardsFeignClientfetchCardDetailsStringString
```
```json
{
  "circuitBreakerEvents": [
    {
      "circuitBreakerName": "CardsFeignClientfetchCardDetailsStringString",
      "type": "SUCCESS",
      "creationTime": "2024-09-19T13:21:13.435892700-07:00[America/Los_Angeles]",
      "errorMessage": null,
      "durationInMs": 49,
      "stateTransition": null
    }
  ]
}
```
- Bring Cards microservice down to simulate failure. 'failureRateThreshold' percentage of calls fail and as the circuit moves into OPEN state, OpenFeign Client returns response from the fallback method.
As service comes back up and 'waitDurationInOpenState' passes the circuit moves to HALF_OPEN state to check if calls succeed and return response accordingly. 
```json
{
  "circuitBreakerEvents": [
    {
      "circuitBreakerName": "CardsFeignClientfetchCardDetailsStringString",
      "type": "SUCCESS",
      "creationTime": "2024-09-19T13:21:13.435892700-07:00[America/Los_Angeles]",
      "errorMessage": null,
      "durationInMs": 49,
      "stateTransition": null
    },
    {
      "circuitBreakerName": "CardsFeignClientfetchCardDetailsStringString",
      "type": "ERROR",
      "creationTime": "2024-09-19T13:24:09.758075600-07:00[America/Los_Angeles]",
      "errorMessage": "java.util.concurrent.TimeoutException: TimeLimiter 'CardsFeignClientfetchCardDetailsStringString' recorded a timeout exception.",
      "durationInMs": 1012,
      "stateTransition": null
    },
    {
      "circuitBreakerName": "CardsFeignClientfetchCardDetailsStringString",
      "type": "ERROR",
      "creationTime": "2024-09-19T13:24:44.435052300-07:00[America/Los_Angeles]",
      "errorMessage": "feign.FeignException$ServiceUnavailable: [503] during [GET] to [http://cards/api/fetch?mobileNumber=4354437687] [CardsFeignClient#fetchCardDetails(String,String)]: [Load balancer does not contain an instance for the service cards]",
      "durationInMs": 2,
      "stateTransition": null
    },
    {
      "circuitBreakerName": "CardsFeignClientfetchCardDetailsStringString",
      "type": "FAILURE_RATE_EXCEEDED",
      "creationTime": "2024-09-19T13:24:44.435052300-07:00[America/Los_Angeles]",
      "errorMessage": null,
      "durationInMs": null,
      "stateTransition": null
    },
    {
      "circuitBreakerName": "CardsFeignClientfetchCardDetailsStringString",
      "type": "STATE_TRANSITION",
      "creationTime": "2024-09-19T13:24:44.435052300-07:00[America/Los_Angeles]",
      "errorMessage": null,
      "durationInMs": null,
      "stateTransition": "CLOSED_TO_OPEN"
    },
    {
      "circuitBreakerName": "CardsFeignClientfetchCardDetailsStringString",
      "type": "STATE_TRANSITION",
      "creationTime": "2024-09-19T13:27:49.205135600-07:00[America/Los_Angeles]",
      "errorMessage": null,
      "durationInMs": null,
      "stateTransition": "OPEN_TO_HALF_OPEN"
    },
    {
      "circuitBreakerName": "CardsFeignClientfetchCardDetailsStringString",
      "type": "SUCCESS",
      "creationTime": "2024-09-19T13:27:49.249700200-07:00[America/Los_Angeles]",
      "errorMessage": null,
      "durationInMs": 44,
      "stateTransition": null
    }
  ]
}
```