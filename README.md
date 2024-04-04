# Flights Booking Service

Introduction
The Flights Booking Service is a critical microservice responsible for handling flight bookings in the Flight Ticket Booking system. It provides the necessary APIs to manage flight bookings, along with advanced features such as idempotency key for handling retries after successful payment, scheduled cron jobs for canceling old pending bookings, and a concurrent seat locking mechanism to ensure data consistency.

**High Level Design of the project**

![High Level Design of the project](./High-Level-Design.png)

**Schema of Booking Service**

![Schema](./Schema.png)

**High level flow of Booking Service**

The flow of this booking service is inspired by real life booking services like Flipkart Flights. When we initiate a booking like that on Flipkart Flights, it creates a temporary booking in the 'INITIATED' state. There's a timer of sometime (like 20 minutes or so), within which the payment must be done. If the payment service return a 200 (OK) response, then the booking service creates the required booking. Else if the session expires, then the booking transitions from 'INTIATED' state to 'TIMEDOUT' state.

When the user makes a POST call to initiate a booking, they'll be sending the payload in the format:

```
{
    flightId: 1,
    userId: 2,
    noOfSeats: 5
}
```

In order to validate whether the number of seats requested against the number of available seats, finding out the price per seat and updating the seats in case of a successful booking, we need to communicate with the Flights Search Service. Hence, we facilitate inter-service communication through REST API between the Bookings Service and Search Service.

We use row level locks in the Booking Service, so that a lock can be ensured on a particular record in the 'Flights' table. This ensures consistency of data in the table.

The entire logic to create a booking is atomic in nature, as we are leveraging the transactional capabilities provided by Sequelize. If the booking transaction fails at any point, the entire progress made so far is rolled back. The consistency of data is ensured by using the row based locks on the 'Flights' table, as discussed before.

For the scope of this project, we are not going to integrate a payment gateway. Instead, we are going to create a mock payments API. We expect the payment to be completed within 5 minutes from the time of the initiation of the booking. Else the booking will transition from INTIATED to CANCELLED state and the held seats in the 'Flight' table will also be released.

We also setup cron jobs to search every 10 minutes for bookings in the 'Bookings' table which have been timed out, and cancel those bookings.

There's one more problem that we solved in this service: which is if the user by mistake makes the API call to payment service twice, the money is going to be deducted twice. In order to solve this problem, we make the Payment API as an idempotent API. What this means is that, for every API request, we will generate a unique idempotency key. If that particular idempotency key has already been served by the payment API, the request will not be allowed. Else, we are going to serve that request, and if that request is successful, we will store this idempotency key in some storage mechanism like Database, caching mechanisms like Redis or an in-memory data structure. Four our project, we will be using an in-memory data structure.


Some of the screenshots from this service are as follows:

1. The booking rolls back in case the required number of seats are not available: 

![Booking transaction rolls back](./Booking_transaction_rollsback_when_insufficient_seats.PNG)

2. The booking commits in case the required number of seats are available:

![Booking transaction commits](./Booking-transaction-commits.PNG) 

3. The booking rolls back if the flight search service is not available:

![Booking transaction rolls back](./Transaction_rollback_if_flight_search_Service_is_unavailable.PNG)

4. The booking expires after the timeout is down:

![Booking expires after the timeout](./Booking_expires_after_timeout.PNG)

5. Running a cron job to clear old bookings:

![Running cron job to clear old bookings](./Cron_job_to_clear_old_bookings.PNG)

6. Preventing duplicate API requests by making the Payment API idempotent

![Idmepotent Payment API](./Idempotent_API.PNG)