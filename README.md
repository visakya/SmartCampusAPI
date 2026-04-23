# Smart Campus API

## Overview

The Smart Campus API is a RESTful web service developed using Java, JAX-RS and Apache Tomcat. It simulates a campus monitoring system that manages rooms, sensors and sensor readings.

## The API allows clients to: 

1.Manage rooms and their metadata
2.Register sensors and link them to rooms
3.Record and retrieve time-series sensor readings
4.Filter sensors based on type
5.Handle errors using structured JSON responses

## Technologies used

Java 17+
Jakarta RESTful Web Services(JAX-RS)
Jersey
Apache Tomcat
Maven

## How to Run the Project

1.Open the project in NetBeans
2.Right-click the project - Select Clean and Build
3.Right-click - Select Run
4.Ensure Apache Tomcat is configured as the server
5.Access API at http://localhost:8080/api/v1

## API Endpoints

### Discovery
GET /api/v1

### Rooms
GET /api/v1/rooms
GET /api/v1/rooms/{roomId}
POST /api/v1/rooms
DELETE /api/v1/rooms/{roomId}

### Sensors
GET /api/v1/sensors
GET /api/v1/sensors/{sensorId}
GET /api/v1/sensors?type=CO2
POST /api/v1/sensors

### Sensor Readings
GET /api/v1/sensors/{sensorId}/readings
POST /api/v1/sensors/{sensorId}/readings

## Sample curl commands
1.Get API Discovery
curl http://localhost:8080/api/v1

2.Get all rooms
curl http://localhost:8080/api/v1/rooms

3.Create a new room
curl -X POST http://localhost:8080/api/v1/rooms \
-H "Content-Type: application/json" \
-d '{"id":"ENG-101","name":"Engineering Room", "capacity":80}'

4.Get all sensors
curl http://localhost:8080/api/v1/sensors

5.Filter sensors by type
curl http://localhost:8080/api/v1/sensors?type=CO2

6.Create a sensor
curl -X POST http://localhost:8080/api/v1/sensors \
-H "Content-Type: application/json" \
-d '{"id": "TEST-001","type":"Temperature","status":"ACTIVE","currentValue":0.0,"roomId":"LIB-301"}'

7.Add a sensor reading
curl -X POST http://localhost:8080/api/v1/sensors/TEMP-001/readings \
-H "Content-Type:application/json" \
-d '{"value":25.5}'

# Answers for Questions

###Part 1: Service Architecture & Setup (10 Marks)

1. Project & Application Configuration (5 Marks):
##Question: 
In your report, explain the default lifecycle of a JAX-RS Resource class. Is a new instance 
instantiated for every incoming request, or does the runtime treat it as a singleton? Elaborate on 
how this architectural decision impacts the way you manage and synchronize
your in-memory data structures (maps/lists) to prevent data loss or race conditions.

## Answer
A JAX-RS resource class follows a per-request lifecycle by default. It means a new instance of the 
resource class is usually created for each incoming HTTP request rather than being treated as a singleton.
This behaviour helps keep resource classes stateless and it also avoids accidental sharing of request-specific 
data between clients.
Here that design means the resource classes themselves should not store shared application data in instance fields 
because those objects are recreated for each request. Instead, shared data such as rooms, sensors and readings should be stored in 
common in-memory data structures such as maps and lists in a separate datastore class.
However, because these shared maps and lists may be accessed by multiple requests at the same time
concurrency issues can still happen. For example, two users could try to modify the same room or 
sensor collection simultaneously. To reduce the risk of race conditions or inconsistent data, 
the shared structures should be managed carefully. In a more advanced implementation, thread-safe collections 
such as ConcurrentHashMap or explicit synchronization could be used to improve safety.

2. The ”Discovery” Endpoint (5 Marks):
## Question: 
Why is the provision of ”Hypermedia” (links and navigation within responses) considered a
hallmark of advanced RESTful design (HATEOAS)? How does this approach benefit client developers compared to static documentation? 

## Answer
Hypermedia is considered an essential characteristic of advanced RESTful design because it allows 
the server to guide clients by including links and navigation options directly inside responses.
This concept is often described as HATEOAS, meaning Hypermedia As The Engine Of Application State. 
Instead of the client hardcoding every endpoint and workflow in advance the client can discover available 
actions dynamically from the API responses.
This benefits client developers because it reduces dependence on static documentation and 
makes the API easier to use and evolve. For example, if the server includes links to the rooms and 
sensors collections in the discovery response, the client can navigate the API without memorising or 
manually constructing every URL. It also improves maintainability because changes to resource paths or 
navigation structure can often be handled by updating the server responses rather than requiring major client-side changes.

###Part 2: Room Management 

1. RoomResource Implementation:
## Question: 
When returning a list of rooms, what are the implications of returning only IDs versus returning the full room objects? Consider network bandwidth and client side processing.

## Answer
Returning room Ids only will reduce the amount of data sent over the network. So it will be more efficient in terms of bandwidth and can improve performance when there are many rooms. 
It will also make the response smaller and faster to process if the client only needs references to the rooms rather than their full details. However returning the full room objects will be
easier for the client because all relevant metadata such as the room name, capacity and assigned sensors will be available immediately in a single request. This will reduce the need for extra 
follow-up requests and will simplify client side logic. So returning only Ids will be more lightwight and bandwidth  efficient while returning full room objects will be more useful when clients
need complete information and want to reduce additional API calls.

2. RoomDeletion & Safety Logic: 
## Question: 
Is the DELETE operation idempotent in your implementation? Provide a detailed justification by describing what happens if a client mistakenly sends the exact same DELETE request for a room
multiple times.

## Answer
Yes, the DELETE operation is intended to be idempotent.Idempotent means that sending the same request multiple times should have the same overall effect on the server state as sending it once.
In this implementation the first successful DELETE request removes the room from the in memory datastore. Sending the DELETE request to the server again will return a not found response since the entry 
has already been deleted. While the response status may vary depending on whether the request was issued initially or repeatedly the end result is that the room does not exist.
Therefore the operation is considered idempotent since there is no further change to the resource after the first success.

### Part 3: Sensor OPerations & Linking 

1. Sensor Resource & Integrity:
## Question: 
We explicitly use the @Consumes (MediaType.APPLICATION_JSON) annotation on the POST method. Explain the technical consequences if a client attempts to send data in a different format, such as text/plain or application/xml.
How does JAX-RS handle this mismatch?

## Answer
The @Consumes(MediaType.APPLICATION_JSON)annotation indicates to JAX-RS that the method only accepts request bodies in JSON format. If a client sends data in a different format such as text/plain or application/aml the JAX-RS runtime 
will not find a suitable message body reader for that content type. As a result the request will be rejected typically with an HTTP 415 Unsupported Media Type response. This behaviour is useful because it sets up a solid interface between the client and the RESTful service.
It helps prevent confusion when parsing client requests and processing them.

2. Filtered Retrieval & Search
## Question: 
You implemntated this filtering using QueryParam. Contrast this with an alternative design where the type is part of the URL path (e.g., /api/v1/sensors/type/C02). Why is the query parameter approach generally considered 
superior for filtering and searching collections?

## Answer
Using a query parameter such as /api/v1/sensors?type=CO2 is generally better for filtering because it clearly shows that the client is searching within the same resource collection rather than requesting a completely different resource path. The main collection is still sensors 
and the query parameter simply refines the result set. This approach is more flexible and more consistent with RESTful design especially when adding multiple filters like status, roomId or value ranges. For instance query parameters make it easy to support requests like 
/api/v1/sensors?type=CO2&status=ACTIVE. However placing the filter value inside the path like /api/v1/sensors/type/CO2 becomes impractical in this case.

###Part 4: Deep Nesting with Sub-Resources

1. The Sub-Resource Locator Pattern:
##Question: 
Discuss the architectural benefits of the Sub-Resource Locator pattern. How does delegating logic to separate classes help manage complexity in large APIs compared to defining every nested path (e.g., sensors/{id}/readings/{rid}) 
in one massive controller class?

## Answer
The Sub-Resource Locator pattern improves API design by delegating nested resource handling to a dedicated class instead of placing every nested path inside one large controller. It keeps the code more modular, readable and easier to maintain.
Here the SensorResource class is responsible for the main sensor collection and SensorReadingResource handles the reading history for a specific sensor. Having two separate classes makes the design cleaner as each class has a focused responsibility. 
It also makes future changes easier because additional reading related operations can be added without making the main sensor resource over complicated. Compared to putting all nested paths such as sensors/{id}/readings/{rid} into one large resource class 
the sub-resource approach reduces clutter and improves scalability for larger APIs.


###Part 5: Advanced Erro Handling, Exception Mapping & Logging

2. Dependency Validation:
## Question: 
Why is HTTP 422 often considered more semantically accurate than a standard 404 when the issue is a missing reference inside a valid JSON payload?

## Answer
HTTP 422 Unprocessable Entity is often more semantically accurate than 404 in this situation because the requested endpoint itself exists but the content of the request is invalid. Here the client successfully sends a JSON payload to the sensor creation endpoint 
but the roomId inside that valid payload refers to a room that does not exist. A 404 Not Found usually means the URI being requested does not exist. However 422 shows that the server understood the request and its format but could not process it because one of the 
values inside the payload was logically invalid. So 422 communicates that the problem lies in the submitted data better rather than in the endpoint path. 

4. The Global Safety Net (500):
## Question : 
From a cybersecurity standpoint, explain the risks assoicated with exposing internal java stack traces to external API consumers. What specific information could an attacker gather from such a trace?

## Answer 
Exposing internal java stack traces to external users is a cybersecurity risk because it reveals implementation details of the server. An attacker can learn class names, package structures, library versions, file paths, framework configuration and internal method calls.
These can help them identify weaknesses, guess technologies that have been used and craft more targeted attacks. Because of this a global exception mapper is very important. Instead of returning raw stack traces the API should return a generic 500 Internal Server Error response
with a safer message to the client while keeping the actual exception details only in the server logs for developers.

5. API Request & Response Logging Filters:
## Question: 
Why is it advantageous to use JAX-RS filters for cross-cutting concerns like logging, rather than manually inserting Logger.info() statements inside every single resource method? 

## Answer
The reason why using JAX-RS filters for cross-cutting concerns such as logging is better than manually placing Logger.info() statements inside every resource method is because filters apply to all requests consistently and responses in one central place. This reduces duplicated code 
and makes the application easier to maintain. This also improves separation of concerns. Resource classes can focus on business logic while the filter handles infrastructure concerns such as observability and monitoring. If logging behaviour needs to change later it can be updated in 
one class rather than across every endpoint method. 