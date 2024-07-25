# Part Three - Testing
This section will show you how to layout your `ApplicationControllerSpec`, 
we will be calling the database directly instead of mocking it. This process is a form of integration testing 
as we will be testing the program from start to result. More specific unit testing will be done later using Mocks,
but for now lets see if our controller and repository are working.

## Building the application

Go back to your `ApplicationControllerSpec`. Make it extend `BaseSpecWithApplication`. 

Command and click on `BaseSpecWithApplication`, this will navigate you to this traits information. You will see: 

```scala
trait BaseSpec extends AnyWordSpec with Matchers

trait BaseSpecWithApplication extends BaseSpec with GuiceOneServerPerSuite with ScalaFutures with BeforeAndAfterEach with BeforeAndAfterAll with Eventually {

  implicit val mat: Materializer = app.materializer
  implicit val executionContext: ExecutionContext = app.injector.instanceOf[ExecutionContext]

  lazy val component: MessagesControllerComponents = injector.instanceOf[MessagesControllerComponents]
  lazy val repository: DataRepository = injector.instanceOf[DataRepository]

  implicit val messagesApi = app.injector.instanceOf[MessagesApi]
  lazy val injector: Injector = app.injector

  override def fakeApplication(): Application =
    new GuiceApplicationBuilder()
      .configure(Map(
        "mongodb.uri"                                    -> "mongodb://localhost:27017/play-template"
      ))
      .build()

  lazy val fakeRequest: FakeRequest[AnyContentAsEmpty.type] =
    FakeRequest("", "").withCSRFToken.asInstanceOf[FakeRequest[AnyContentAsEmpty.type]]
  implicit val messages: Messages = messagesApi.preferred(fakeRequest)

  def buildPost(url: String): FakeRequest[AnyContentAsEmpty.type] =
    FakeRequest(POST, url).withCSRFToken.asInstanceOf[FakeRequest[AnyContentAsEmpty.type]]

  def buildGet(url: String): FakeRequest[AnyContentAsEmpty.type] =
    FakeRequest(GET, url).withCSRFToken.asInstanceOf[FakeRequest[AnyContentAsEmpty.type]]
}
```
This is something that is **not** provided by Play, and it just an example to help get you started. Let's break it down:
* All the traits in the extensions come from the `libraryDependencies` in our `build.sbt`
* The implicit vals aren't important right now, but know they are needed in other areas (Hint: To see where they are used try commenting one out)
* `component` and `repository` are created instances of the controller components and data repository, note that new instances will be made for each suite ran
* `fakeRequest` and `fakeApplication` use Guice to help construct an instance of the application being called
* Similarly, `buildPost` and `buildGet` are methods we can use in the tests to pass fake requests to the controller

Back to your code. Inside `ApplicationControllerSpec`, create an instance of the controller above all tests:
```scala
val TestApplicationController = new ApplicationController(
    repository,
    component
  )
```
Now we should be able to call all the methods in the controller

### Testing the .create function
* Add the following to your test, just **below** where you create the `TestApplicationController`.

    ```scala
    private val dataModel: DataModel = DataModel(
        "abcd",
        "test name",
        "test description",
        100
    )
    ``` 
* We've created a simple test `DataModel` to use later. For example, add and complete the following test 
    ```scala
    "ApplicationController .create" should {
    
      "create a book in the database" in {
    
        val request: FakeRequest[JsValue] = buildPost("/api").withBody[JsValue](Json.toJson(dataModel))
        val createdResult: Future[Result] = TestApplicationController.create()(request)
    
        status(createdResult) shouldBe Status.???
      }
    }
    ```
* Running this test with an empty database works fine, but once the result is created the database is never emptied.
  Likewise, run this test with an existing `dataModel` example, and we'll get a conflict where the value already exists.
* This is why mocking is useful, we could test the Controller exclusive to the database's current state, but we'll come to that later.
* For now add these lines to the bottom of the test class and call them at the top and bottom of each test. They will delete the contents of the repository before and after each test.
    ```scala
    "test name" should {
      "do something" in {
         beforeEach 
         ... 
         afterEach
      }
    }
  ...
      override def beforeEach(): Unit = await(repository.deleteAll())
      override def afterEach(): Unit = await(repository.deleteAll())
    ```
  
### Testing the .read function
* Testing the `.read` function in the controller should be a similar process, 
  except we will need to use `.create` in order to find something in our repository. 
  Add and complete the missing implementation from the test below
    ```scala
    "ApplicationController .read" should {
    
      "find a book in the database by id" in {
    
        val request: FakeRequest[JsValue] = buildGet("/api/${dataModel._id}").withBody[JsValue](Json.toJson(dataModel))
        val createdResult: Future[Result] = TestApplicationController.create()(request)
    
        //Hint: You could use status(createdResult) shouldBe Status.CREATED to check this has worked again
    
        val readResult: Future[Result] = TestApplicationController.read("abcd")(FakeRequest())
    
        status(readResult) shouldBe ???
        contentAsJson(readResult).as[???] shouldBe ???
      }
    }
    ```

#### Tasks
1. We have created a happy path case for two of the controller methods,
   go ahead and make tests for the other controller methods following the same approach.

2. Make a BadRequest for each controller method and test for these. It might seem backwards, 
   but your tests should pass for errors if you are expecting them.

## Manually Testing your API 

What if you want to see your API in action adding data to Mongo?

You can use [CURL](https://curl.haxx.se/) to make HTTP requests for each of the new endpoints you've created.

1. Start your service with `sbt run`

2. In the command line, run the following command:

```
curl "localhost:9000/api" -i
```

You should see something like the following:
```  
HTTP/1.1 200 OK
Referrer-Policy: origin-when-cross-origin, strict-origin-when-cross-origin
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
X-Content-Type-Options: nosniff
Content-Security-Policy: default-src 'self' https://stackpath.bootstrapcdn.com/bootstrap/4.5.0/css/bootstrap.min.css
X-Permitted-Cross-Domain-Policies: master-only
Date: Mon, 23 May 2022 14:08:53 GMT
Content-Type: application/json
Content-Length: 2

[]
```
* The first line shows the response was a HTTP 200 OK
* The next few lines show response headers and meta data
* The last line `[]` is the response body. Because there's nothing in Mongo yet, this is empty

### Populating the database

To populate the database with some data, run the following CURL command:
```
curl -H "Content-Type: application/json" -d '{ "_id" : "1", "name" : "testName", "description" : "testDescription", "pageCount" : 1 }' "localhost:9000/api/create" -i
```
* `-H "Content-Type: application/json"` sets a header so we can pass JSON into the body
* `-d '{ "_id" : "1", "name" : "testName", "description" : "testDescription", "pageCount" : 1 }'` defines the JSON body

You should get the following response:
``` 
HTTP/1.1 201 Created
Referrer-Policy: origin-when-cross-origin, strict-origin-when-cross-origin
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
X-Content-Type-Options: nosniff
Content-Security-Policy: default-src 'self' https://stackpath.bootstrapcdn.com/bootstrap/4.5.0/css/bootstrap.min.css
X-Permitted-Cross-Domain-Policies: master-only
Date: Mon, 23 May 2022 14:12:38 GMT
Content-Length: 0
```

Try running that same command again. You should get the following:
```
HTTP/1.1 500 Internal Server Error
Date: Mon, 23 May 2022 14:23:05 GMT
Content-Type: text/html; charset=UTF-8
Content-Length: 16040
...
```

This is because you've tried to add a document to mongo with the same `_id`.
A DatabaseException was thrown. This needs to be caught with error handling.

### Viewing your database
There are a few ways you can view your newly created data in Mongo. One way is to download an app that provides a useful UI.

1. Download and install [Studio 3T](https://studio3t.com/download/) if you didn't do this as part of your laptop set up.

2. Once installed, open the app and you should be able to connect with the default settings.

3. You should see a database with the same name as your project (set in `application.conf`). Click this.

4. There should be a collection called `data` (set in `DataRepository`). Click this.

5. You should see an entry for the data you created!

### Manually testing the rest of the API

1. Add another document to the database with a **different** `_id`

2. Run the CURL command from earlier to test the `GET /api` route to check it returns your data items

3. Try hitting the `GET /api/:id` route to retrieve **only** the second document you added

4. Hit the `PUT /api/:id` route with a CURL command. To specify the PUT HTTP method, use `-X PUT` in your CURL command

5. Hit the `DELETE /api/:id` route with a CURL command to delete a specific item. How can you specify the DELETE HTTP method?

Now that we have a basic repository, we need to populate it. 
Instead of entering all this information yourself, we will use an existing database!

## [Part 4](Part4.md)
