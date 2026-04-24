Smart Campus API
5COSC022W Client-Server Architectures
University of Westminster
Student Number: w2113683
Student Name: Waleed Dustgir
Due Date: 24th April 2026



HOW TO RUN

1. Open the project in NetBeans
2. Right click the project and click Clean and Build
3. Right click the project and click Run
4. The server starts at http://localhost:8080/api/v1/

Use Postman to test the endpoints listed below.


ENDPOINTS

GET    /api/v1                              API info and resource links
GET    /api/v1/rooms                        Getting all rooms
POST   /api/v1/rooms                        Creating  a new room
GET    /api/v1/rooms/roomId                 Getting one room by ID
DELETE /api/v1/rooms/roomId                 Deleting a room, blocked if sensors exist 
GET    /api/v1/sensors                      Get all sensors,filter with ?type=
POST   /api/v1/sensors                      Register a new sensor
GET    /api/v1/sensors/sensorId             Get one sensor by ID
DELETE /api/v1/sensors/sensorId             Remove a sensor
GET    /api/v1/sensors/sensorId/readings    Get reading history for a sensor
POST   /api/v1/sensors/sensorId/readings    Add a new reading to a sensor



PART 1 - SETUP AND DISCOVERY

Question 1: JAX-RS Resource Lifecycle

By default JAX-RS creates a brand new instance of a resource class every single time a request comes in. This is called request-scoped lifecycle. So if two people call the API at the same time they each get their own completely separate resource object. Once the response has been sent that object gets thrown away by the garbage collector and is gone forever.
This caused me a real problem when I first thought about how to store data. If I had put my rooms HashMap inside RoomResource it would disappear after every single request and nothing would ever persist. To fix this I created a separate DataStore class that uses the singleton pattern so it only ever gets created once. Every resource class calls DataStore.getInstance() to get the same shared copy so all the rooms, sensors and readings stay in memory between requests.
Thread safety was something I also had to think about carefully. Because multiple requests can run at the same time on different threads, two threads could try to write to the same HashMap at exactly the same moment which can cause data corruption or lost updates. I dealt with this by making getInstance() synchronized which means only one thread can enter that method at a time so you can never end up with two separate DataStore instances being created. I also check whether a sensor reading list already exists before creating a new one to prevent one thread overwriting data that another thread just added.



Question 2: HATEOAS

HATEOAS stands for Hypermedia As The Engine Of Application State. The idea is that instead of a client needing to already know every URL in the API, the responses themselves include links pointing to where the client can go next. It works like browsing a website where you just follow links rather than having to memorise every single page address in advance.
The reason this is considered advanced RESTful design is that it makes the API completely self-documenting. When someone calls GET /api/v1 the response gives them links to /api/v1/rooms and /api/v1/sensors so they immediately know what is available without reading any external documentation. It also means if a URL ever changes on the server, clients that follow the links in responses will automatically get the updated path rather than breaking. Static documentation goes out of date very quickly but links embedded directly in API responses always reflect exactly what is there at that moment.



PART 2 - ROOM MANAGEMENT

Question 3: Full Objects vs Just IDs

When a client calls GET /api/v1/rooms there are two ways you could design the response. You could send back just a list of room IDs, or you could send back the full room objects with all their details included.
Sending only IDs keeps the response very small which saves network bandwidth. But then the client has to make a completely separate HTTP request for every single room just to get any useful information out of it. If there are 200 rooms on campus that means 201 HTTP requests in total just to load a simple list which is extremely inefficient especially on slower networks or mobile connections.
Returning the full objects means the response payload is bigger but the client gets everything it needs in a single round trip. For a facilities management dashboard that needs to display room names, capacities and sensor counts all at once this is far more practical. I went with full objects in this project for that reason.



Question 4: Is DELETE Idempotent

Yes DELETE is idempotent in my implementation. I had to look this up properly to make sure I understood it correctly. Idempotent means that calling the exact same operation multiple times produces the same server state as calling it just once.
In my implementation if you call DELETE on room CG-106 and it exists the room gets removed and you get 204 No Content back. If you then send the exact same DELETE request again the room is already gone so you get 404 Not Found. The response code is different but the actual state of the server is identical both times because the room simply does not exist either way. Idempotency is specifically about what happens to the server state not about what response code comes back.
This is different from POST which is not idempotent at all. If you send the same POST request to create a room twice you either end up with two rooms or get a 409 Conflict error. Either way the server state changes depending on how many times you called it which violates idempotency.



PART 3 - SENSORS AND FILTERING

Question 5: Wrong Content Type Consequences

The POST method for sensors has @Consumes(MediaType.APPLICATION_JSON) on it. This annotation tells the JAX-RS runtime that this method will only accept requests where the Content-Type header is set to application/json.
If a client sends the request with Content-Type: text/plain or Content Type: application/xml instead, JAX-RS automatically rejects it with HTTP 415 Unsupported Media Type before my method code even gets a chance to run. This happens because JAX-RS looks for a registered MessageBodyReader that knows how to deserialise the incoming format into a Sensor object. Since no reader is registered for text/plain or xml in my project it simply refuses the request straight away and returns the 415 error automatically.
I found this really useful because it means badly formatted requests get rejected at the framework level without me needing to write any extra validation code inside my methods.



Question 6: QueryParam vs Path Parameter

I used @QueryParam to filter sensors so the URL looks like /api/v1/sensors?type=CO2 rather than something like /api/v1/sensors/type/CO2 where the filter is baked into the path itself.
Path parameters are specifically designed to identify one unique resource. For example /api/v1/sensors/TEMP-001 points to one exact specific sensor. Putting a filter criterion like type into the path makes it look like you are identifying a specific named resource which is semantically wrong because you are just narrowing down a collection.
Query parameters are also optional by nature which is exactly what you want for filtering. You can call /api/v1/sensors with nothing and get the full list back, or add ?type=CO2 to narrow it down. You can even combine multiple filters like ?type=CO2&status=ACTIVE which would become extremely complicated to implement with path parameters. Query parameters also follow established REST conventions for search and filter operations which makes the API immediately familiar to any developer who has worked with REST APIs before.

----------------------------------------------------------------

PART 4 - SUB RESOURCES

Question 7: Sub Resource Locator Pattern

In SensorResource I have a method with @Path on sensorId/readings but no HTTP method annotation like @GET or @POST on it. This is called a sub-resource locator. Instead of handling the request itself it creates and returns a new instance of SensorReadingResource and then JAX-RS figures out which method on that class to actually invoke based on the HTTP method of the incoming request.
The main benefit is that it keeps each class focused on one single responsibility. If I had put all the readings logic directly inside SensorResource that class would have become extremely long and very difficult to maintain. By delegating to SensorReadingResource each class has one clear job. SensorResource handles sensor CRUD operations and SensorReadingResource handles reading history. If the readings logic ever needs to change I only need to look in one place.
In a larger API with many nested resources this pattern becomes even more valuable. Without it you would end up with one massive controller class containing methods for every possible nested path which would be nearly impossible to read, test or modify without breaking something else. This is sometimes called the God Controller anti-pattern and the sub-resource locator pattern is the solution to it.

Question 8: currentValue Side Effect

SensorReadingResource has a GET method that returns all historical readings for a sensor and a POST method that adds a new reading to that list.
After a successful POST I also call sensor.setCurrentValue(reading.getValue()) to update the parent Sensor object with the value from the new reading. This side effect is important for keeping the data consistent across the whole API. Without it a client could call GET /api/v1/sensors/TEMP-001 and see a stale currentValue that does not match the most recent reading in the history. By updating the parent sensor immediately after every new reading is posted both the sensor object and the readings history always agree on what the latest measurement was.


PART 5 - ERROR HANDLING

Question 9: Why 422 and Not 404

When a sensor is posted with a roomId that does not exist in the system I return 422 Unprocessable Entity rather than 404 Not Found.
404 means the URL endpoint itself was not found. But in this case the URL is completely valid, POST /api/v1/sensors exists and works correctly. The problem is not the URL it is the content of the request body. The roomId field inside the JSON is referencing a room that does not exist in the system.
422 Unprocessable Entity is specifically designed for exactly this situation where the request is syntactically valid JSON and the URL is correct but the semantic content of the request body cannot be processed. Using 422 gives the client a much clearer and more accurate signal that they need to fix the data in their request body rather than their URL. If I returned 404 the client might waste time thinking they are hitting the wrong endpoint when the endpoint is completely fine.



Question 10: Stack Traces Security Risk

Sending a raw Java stack trace back to a client is a serious security vulnerability. A stack trace exposes which exact version of Java is running on the server, which libraries and frameworks are being used along with their exact version numbers, internal class names and method names from inside the application, and sometimes even file paths on the server.
An attacker could take all of this information and cross-reference it against public CVE databases to find known unpatched vulnerabilities in those specific library versions and then craft a targeted attack using that knowledge. This is called information disclosure and it is one of the OWASP Top 10 security risks.
The GlobalExceptionMapper catches any exception that none of the more specific mappers handled and returns a plain generic 500 message with no technical detail whatsoever. The full error including the complete stack trace still gets logged on the server side where only developers with server access can see it. The client never receives any information that could help an attacker understand the internal workings of the system.


Question 11: Logging Filter Advantages

Rather than adding individual log statements inside every single resource method I implemented a JAX-RS filter using ContainerRequestFilter and ContainerResponseFilter. This automatically intercepts every HTTP request and response without me having to modify any resource class at all.
The main advantage is that it follows the separation of concerns principle. Logging is a cross-cutting concern that should not be mixed in with business logic. Resource classes stay focused purely on handling requests and the filter handles observability completely independently. If I add a brand new endpoint tomorrow the logging just works for it automatically without me having to remember to add anything to the new class.
The ContainerResponseFilter also runs after everything else has finished including any exception mappers which means it always captures the actual final HTTP status code sent to the client. If I had logged inside the resource method itself I would not always know what the final status code was going to be especially when exceptions get thrown and caught by mappers further down the chain.


PROJECT STRUCTURE

application
    DataStore.java
    Main.java
    SmartCampusApplication.java

model
    Room.java
    Sensor.java
    SensorReading.java

resource
    DiscoveryResource.java
    RoomResource.java
    SensorResource.java
    SensorReadingResource.java

exception
    ErrorResponse.java
    GlobalExceptionMapper.java
    LinkedResourceNotFoundException.java
    LinkedResourceNotFoundMapper.java
    RoomNotEmptyException.java
    RoomNotEmptyMapper.java
    SensorUnavailableException.java
    SensorUnavailableMapper.java

filter
    ApiLoggingFilter.java

