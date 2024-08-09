# Circuit Choreography

[![CC BY-NC-SA 4.0][cc-by-nc-sa-shield]][cc-by-nc-sa]
![GitHub commit activity](https://img.shields.io/github/commit-activity/m/erwinkramer/circuit-choreography)

![It 973 f.21v A Gentleman dancing with two Ladies, from 'Trattato dell'Arte del Ballare' (vellum) - Italian School, (15th century)](front-image.png)

## Introduction

The Circuit Choreography pattern defines a choreographic message flow that maintains a closed circuit at all times.

The pattern integrates principles of distributed consistency, the [Compensating Transaction pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/compensating-transaction), the [Circuit Breaker pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker), the [Saga distributed transactions pattern](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/saga/saga), and the [Choreography pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/choreography) to create a robust and modern design framework.

## Rules

The following rules must be applied to implement Circuit Choreography:

1. Any service handles atomic operations independently.
2. The choreography maintains a closed circuit at all times, meaning: there is always a **forward** and **backward** route that completes. If they cannot complete, the circuit has to be restored by a **restoration** route.
3. Correlation between services in a choreography will always be implemented with a [Correlation ID](https://microsoft.github.io/code-with-engineering-playbook/observability/correlation-id/).
4. A **forward** routing is the choreography that moves to one or more subsequent services and starts at the initiating service and completes at the final service.
5. A **backward** routing is the choreography that moves back in opposite order and starts at the final service and completes at the initiating service.
6. A **restoration** routing is the choreography that replaces both forward and backward routing, in case of an open circuit (initialized by a circuit breaker), and starts at the current service and completes at that same service.
7. A restoration routing is definitive and reverses the route based on a restoration level and demands compensation from the preceding service, which in turn demands compensation from the next preceding service until the start of the restoration routing is reached.
8. A **restoration level** defines the impact of the open circuit and determines what type of restoration is needed for a service. Levels can be defined by business acumen but must be arranged in ascending order, where 1 is most critical and demands full compensation, any level with higher numerical range might have less demanding compensation, for example:
   - Level 1 does a serious reversal because of a critical issue.
   - Level 2 might do a partial reversal.
   - Level 3 might keep the transaction.
9. If a restoration routing fails moving to a next service, it activates a long running process that keeps trying to go in the same reversed order it started.

Optionally:

1. To ensure exactly once delivery, consider implementing [the Inbox and outbox pattern](https://en.wikipedia.org/wiki/Inbox_and_outbox_pattern) at each service.

## Choreography in a closed circuit

What if everything just works? There will be a route forward and a route backwards that both complete.

``` mermaid
flowchart TD

    init[Initiating Service] -- 1. and 9. --> initcompensable
    initcompensable[(Compensable or compensating logic)]
    initcompensable -- 2. --> interm

    interm[Intermediate Service] -- 3. and 7. --> intermcompensable
    intermcompensable[(Compensable or compensating logic)]
    intermcompensable -- 4. --> final
    intermcompensable -- 8. --> init

    finalcompensable[(Compensable or compensating logic)]
    final[Final Service] -- 5. --> finalcompensable
    finalcompensable -- 6. --> interm
```

*- Routing forward:*

1-2: Initiating Service transacts the way forward and calls the Intermediate Service.

3-4: Intermediate Service transacts and calls the Final Service.

5: Final Service transacts and completes the way forward.

*- Routing backwards:*

6: Final Service transacts the way backwards and calls the Intermediate Service.

7-8: Intermediate Service transacts and calls the Initiating Service.

9: Initiating Service transacts and completes the way backwards.

## Choreography restoration from an open circuit routing forwards

What if we trip the circuit at step 4? The Final Service failed on the forward route, so the Intermediate Service handles a restoration route by reversing and demanding compensation.

``` mermaid
flowchart TD

    init[Initiating Service] -- 1. and 5. --> initcompensable
    initcompensable[(Compensable or compensating logic)]
    initcompensable -- 2. and 6. --> interm

    interm[Intermediate Service] -- 3. and 7. --> intermcompensable
    intermcompensable[(Compensable or compensating logic)]
    intermcompensable -- 4a. ❌ tripped --> final
    intermcompensable -- 4b. --> init

    finalcompensable[(Compensable or compensating logic)]
    final[Final Service] -- ❌ defunct route --> finalcompensable
    finalcompensable -- ❌ defunct route --> interm
```

*- Routing forward:*

1-2: Initiating Service transacts the way forward and calls the Intermediate Service.

3-4a: Intermediate Service transacts and calls the Final Service.

4a: Final Service tripped.

*- Routing the restoration:*

4b: Intermediate Service compensates and calls the Initiating Service.

5-6: Initiating Service compensates and calls the Intermediate Service.

7: Intermediate Service transacts and completes the restoration.

### As a sequence with API Management

The following sequence diagram displays the choreography restoration from an open circuit routing forwards, as the second alternative path, in an architecture with API Management placed as a proxy to front the services to provide [Circuit breaker functionality like Azure API Management implements](https://learn.microsoft.com/en-us/azure/api-management/backends?tabs=bicep#circuit-breaker).

``` mermaid
sequenceDiagram
    participant API Management

    activate Initiating Service 
    Initiating Service->>Initiating Service: Compensable local transaction
    Initiating Service->>+API Management: Call next in route
    API Management-->>+Intermediate Service: Proxy the call
    Intermediate Service->>Intermediate Service: Compensable local transaction
    Intermediate Service->>-API Management: Call next in route

    alt completing the route forward and backwards
        API Management-->>+Final Service: Proxy the call
        Final Service->>Final Service: Compensable local transaction
        Final Service->>API Management: Call previous in route
        deactivate Final Service
        API Management-->>+Intermediate Service: Proxy the call
        Intermediate Service->>Intermediate Service: Compensable local transaction
        Intermediate Service->>API Management: Call previous in route
        deactivate Intermediate Service
        API Management-->>Initiating Service: Proxy the call
        Initiating Service->>Initiating Service: Compensable local transaction
        
    else routing the restauration
        API Management-->>+Intermediate Service: Respond with Circuit Break
        Intermediate Service->>Intermediate Service: Compensate local transaction
        Intermediate Service->>API Management: Call previous in route
        API Management-->>Initiating Service: Proxy the call
        Initiating Service->>Initiating Service: Compensate local transaction
        Initiating Service->>API Management: Call previous in route
        deactivate Initiating Service 

        API Management-->>Intermediate Service: Proxy the call
        deactivate API Management 
        
        Intermediate Service->>Intermediate Service: Completes the restoration
        deactivate Intermediate Service 
    end

```

## Choreography restoration from an open circuit routing backwards

What if we trip the circuit at step 8? The Initiating Service failed at the backwards route, so the Intermediate Service handles a restoration route by reversing and demanding compensation. It has to create a long running operation to close the circuit again.

``` mermaid
flowchart TD

    init[Initiating Service] -- 1. and 13. --> initcompensable
    initcompensable[(Compensable or compensating logic)]
    initcompensable -- 2. and 14.--> interm
    interm[Intermediate Service] -- 3. and 7.  and 11. and 15. --> intermcompensable
    intermcompensable[(Compensable or compensating logic)]
    intermcompensable -- 4. and 8b. --> final
    intermcompensable --  8a. ❌ tripped and  12. ⌛ long running--> init

    finalcompensable[(Compensable or compensating logic)]
    final[Final Service] -- 5. and 9.--> finalcompensable
    finalcompensable -- 6. and 10. --> interm
```

*- Routing forward:*

1-2: Initiating Service transacts the way forward and calls the Intermediate Service.

3-4: Intermediate Service transacts and calls the Final Service.

5: Final Service transacts and completes the way forward.

*- Routing backwards:*

6: Final Service transacts the way backwards and calls the Intermediate Service.

7-8a: Intermediate Service transacts and calls the Initiating Service.

8a: Initiating Service tripped.

*- Routing the restoration:*

8b: Intermediate Service compensates and calls the Final Service.

9-10: Final Service compensates and calls the Intermediate Service.

11-12: Intermediate Service compensates and starts a long running compensation operation on the Initiating Service.

13-14: Initiating Service restored operations, compensates and calls the Intermediate Service.

15: Intermediate Service transacts and completes the restoration.

## General implementation guidance

Ann example of implementing Circuit Choreography with RESTful services within a retail system.

Consider a retail shop where a payment is being done by a payment service (initiating service), a fraud detection service is initiated (intermediate service) to validate the payment, and a CRM service is called (final service) to store customer preference and communication history. Every service implements a forward, backward and multiple restoration routes. Additional information needed might be implemented in the request body.

The payment service implements an initiating forward route, which is a payment operation that is compensable and assumes the nearest service - in this case the fraud detection service - understands and utilizes the following operations:

- Backward route: A payment confirmation operation.
- Restoration routes:
  - Level 1: A refund operation for compensating to revert the payment. For instance the fraud detection responded with a positive detection.
  - Level 2: A disregarding operation where compensation is called for circuit break on a non-critical issue, for instance the fraud detection service passed down an issue with the CRM service.

The definition might look like this:

`PUT payments/{payment-id}/customer/{customer-id}?correlationId=[guid]&route=[forward/backwards/restoration][&restorationLevel=1/2]`

The fraud detection service assumes the nearest services - in this case the CRM and payment service - understand and utilize the following operations:

- Forward route: A fraud detection operation that is compensable.
- Backward route: An integrity confirmation operation.
- Restoration routes:
  - Level 1: A voiding operation for compensation to revert the fraud detection on critical issues in the circuit.
  - Level 2: A disregarding operation where compensation is called for circuit break on a non-critical issue, for instance the CRM service could not handle the request.

The definition might look like this:

`PUT fraudDetections/{payment-id}/customer/{customer-id}?correlationId=[guid]&route=[forward/backwards/restoration][&restorationLevel=1/2]`

The CRM service assumes the nearest services - in this case the fraud detection and payment service - understand and utilize the following operations:

- Forward route: A custom preference operation that is compensable.
- Backward route: A preference confirmation operation.
- Restoration route:
  - Level 1: A disposing operation for compensation to revert the preference.
  - Level 2: A disregarding operation where compensation is called for circuit break on a non-critical issue, for instance the payment was cancelled.

The definition might look like this:

`PUT customerPreferences/customer/{customer-id}/payment/{payment-id}?correlationId=[guid]&route=[forward/backwards/restoration][&restorationLevel=1/2]`

## License

This work is licensed under a
[Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License][cc-by-nc-sa].

[![CC BY-NC-SA 4.0][cc-by-nc-sa-image]][cc-by-nc-sa]

[cc-by-nc-sa]: http://creativecommons.org/licenses/by-nc-sa/4.0/
[cc-by-nc-sa-image]: https://licensebuttons.net/l/by-nc-sa/4.0/88x31.png
[cc-by-nc-sa-shield]: https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey.svg
