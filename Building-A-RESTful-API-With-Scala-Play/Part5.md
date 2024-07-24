# Part Five - Unit testing
Working on Scala & Play Framework microservices you'll find yourself needing mocks and stubs a great deal for unit testing.
This section aims to introduce you to the flexibility of the MockFactory mocking framework.

## Mocking With MockFactory

We are going to test the `LibraryService` class first. Since this requires the connector, 
we are going to use mocks to perform unit tests.

1. Create `LibraryServiceSpec` and mirror the path directory from app in the test route. This class needs some extensions:
    ```scala
    class LibraryServiceSpec extends BaseSpec with MockFactory with ScalaFutures with GuiceOneAppPerSuite
    ```

2. Next we need to create an instance of our service in the class
    ```scala
    val mockConnector = mock[LibraryConnector]
    implicit val executionContext: ExecutionContext = app.injector.instanceOf[ExecutionContext]
    val testService = new LibraryService(mockConnector)
    ```
   We've created an `ExecutionContext` class in the same way as before, but our `LibraryConnector` is different.

    ### Why?
   * We explicitly call methods on the `LibraryConnector` class in the service and these methods can return **different** responses based off what you call it with
   * We **don't** want to test our `LibraryConnector` functionality as part of this spec, we only care about the `LibraryServiceSpec`

    The solution to these problems is a concept known as mocking, where you **explicitly** tell the methods in `LibraryConnector` what to return, so that you can test how your service responds independently (usually known as unit testing). In this case, instead of making a call to the Google Books API, we can pretend to have received the `gameOfThrones` JSON from the API and test the functionality of our code. 

3. Add the following layout to the `LibraryServiceSpec` and fill in the missing implementation, so it passes
    ```scala
    val gameOfThrones: JsValue = Json.obj(
        "_id" -> "someId",
        "name" -> "A Game of Thrones",
        "description" -> "The best book!!!",
        "pageCount" -> 100
      )
    
      "getGoogleBook" should {
        val url: String = "testUrl"
    
        "return a book" in {
          (mockConnector.get[???](_: ???)(_: ???, _: ???))
            .expects(url, *, *)
            .returning(Future(gameOfThrones.as[???]))
            .once()
    
          whenReady(testService.getGoogleBook(urlOverride = Some(url), search = "", term = "").value) { result =>
            result shouldBe ???
          }
        }
      }
    ```
   Note: if this is missing,
    ```scala
    (mockConnector.get[???](_: ???)(_: ???, _: ???))
                .expects(*, *, *)
                .returning(Future(gameOfThrones.as[???]))
                .once()
    ```
   , then you can most likely expect a `java.lang.NullPointerException`. 
   This is because, although we've now created a mock[LibraryConnector], we haven't told it what to do yet.
   
   There's a couple of things to break down here:
   * `.expects` can take `*`, this shows that the connector can expect any request in place of the parameter, 
      you might sometimes see this as `any()`. 
   * `returning` explicitly states what the connector method returns and `.once` shows how many times we 
      can expect this response.
   * Since we are expecting a future from the connector, it's best to use the `whenReady` method from the 
     test helpers, this allows for the result to be waited for as the `Future` type can be seen as a placeholder
     for a value we don't have yet.
   
4. Next we will mock an error returned from the connector.
    ```scala
    "return an error" in {
          val url: String = "testUrl"
    
          (mockConnector.get[???](_: ???)(_: OFormat[???], _: ???))
            .expects(url, *, *)
            .returning(???)// How do we return an error?
            .once()
    
          whenReady(testService.getGoogleBook(urlOverride = Some(url), search = "", term = "").value) { result =>
            result shouldBe ???
          }
   }
    ```
   Right now, the connector only returns one type, `Response`, something we provide to it. 
   We need to provide another type for it to return. Let's construct an error class `APIError`

5. Inside the models package, create this class
    ```scala
    import play.api.http.Status
    
    sealed abstract class APIError(
                                   val httpResponseStatus: Int,
                                   val reason: String
                                 )
    
    object APIError {
    
      final case class BadAPIResponse(upstreamStatus: Int, upstreamMessage: String)
        extends APIError(
          Status.INTERNAL_SERVER_ERROR,
          s"Bad response from upstream; got status: $upstreamStatus, and got reason $upstreamMessage"
        )
      
    }
    ```
   In this instance, we are using an abstract class over a trait as an abstract class can pass constructor 
   parameters. For more information on the difference between the two, try [here](https://www.geeksforgeeks.org/difference-between-traits-and-abstract-classes-in-scala/#:~:text=In%20Scala%2C%20an%20abstract%20class,and%20cannot%20support%20multiple%20inheritances.&text=Like%20a%20class%2C%20Traits%20can,and%20fields%20as%20its%20members.)

6. Now we have APIError we can use this for error cases in our other files. e.g. in the DataRepository
   ```scala
    def index(): Future[Either[Int, Seq[DataModel]]]  =
      collection.find().toFuture().map{
      case books: Seq[DataModel] => Right(books)
      case _ => Left(404)
      }
    ```
   Can become:   
   ```scala
    def index(): Future[Either[APIError.BadAPIResponse, Seq[DataModel]]] =
    collection.find().toFuture().map {
      case books: Seq[DataModel] => Right(books)
      case _ => Left(APIError.BadAPIResponse(404, "Books cannot be found"))
    }
    ```
7. Now we can adjust the connector to return "Either" a `Response` or an `APIError`. Since Futures are involved
   here, we are going to use a helpful library called `Cats`. Add this line to your `build.sbt`
    ```scala
    libraryDependencies += ("org.typelevel"                %% "cats-core"                 % "2.3.0")
    ```
8. Next, rewrite the return type and body of your connector method to
    ```scala
    def get[Response](url: String)(implicit rds: OFormat[Response], ec: ExecutionContext): EitherT[Future, APIError, Response] = {
        val request = ws.url(url)
        val response = request.get()
        EitherT {
          response
            .map {
              result =>
                Right(result.json.as[Response])
            }
            .recover { case _: WSResponse =>
              Left(APIError.BadAPIResponse(500, "Could not connect"))
            }
        }
      }
    ```
   `EitherT` will look really confusing right now, so let's unpack it.
   * The response of our `request.get()` will give a `WSResponse`, we then try to parse the Json body as type `Response`
   * So at first this was a `Future[Response]`.
   * When that doesn't work, we catch the error with the `.recover` method, this returns a type `Future[APIError]`
   * `EitherT` allows us to return either of the two, `Future[APIError]` or `Future[Response]`.
   * Note that you cannot have `APIError[Response]` or `Response[APIError`, the syntax of `EitherT` only allows the first type,
     i.e. the `Future`, to be applied over the remaining two types.
   * (It is also standard to have the error as the `Left` and not the `Right` type, since the error is never right!)
9. Our service method and controller method will be giving back errors right about now, let's change that
    * Go to your service and simply change your method to return type `EitherT[Future, APIError, Book]`. 
    * And go to your controller method and complete the following
     ```scala
     def getGoogleBook(search: String, term: String): Action[AnyContent] = Action.async { implicit request =>
         service.getGoogleBook(search = search, term = term).value.map {
           case Right(book) => ??? //Hint: This should be the same as before
           case Left(error) => ???
         }
       }
     ```
    (Notice how we didn't have to change much of the other classes at all since we kept everything separate!)

10. Okay, back to the testing, go ahead and make the "return a book" and "return an error" tests pass in `LibraryServiceSpec`, 
    this should only require small changes).

### Tasks

1. Add error handling for Mongo and all the controller methods, you won't need `Cats`.

2. Find a book in Mongo via different fields, e.g. book name.

3. Update an existing book field in Mongo by only providing the `_id`, field name and change, i.e. not a whole new book.

4. Normally the controller "layer" does not interact directly with the data repository. Instead, the controller makes a call to a "service" layer (which also performs data validation and other checks) which then calls the data repository. Build this service layer between the `ApplicationController` and `DataRepository`. Do this by creating a new scala class called `RepositoryService` in your services collection. 
    * Your `ApplicationController` methods should call those in the service layer, and the service layer should call the `DataRepository`. It won't do much at the moment, but it's needed for the next task. 
    * You'll need to inject the service into `ApplicationController` and add the following to `BaseSpec` alongside the other 'lazy val's:
`lazy val repoService: RepositoryService = injector.instanceOf[RepositoryService]`

5. Add error handling to `DataRepository`, similar to how we did it to the connector. Use Eithers to return either valid data (Right) or an error (Left) to the service layer, which acts on that.
   Add tests for this to `ApplicationControllerSpec`

6. Make sure that your `RepositoryService` functions as intended by using mocking. Do this with a new file called `RepositoryServiceSpec`, making sure to use Eithers where appropriate. 

7. Try mocking Mongo for the controller. This will involve a few new steps:
   * Create a `trait` in the repository class
   * Above this trait copy in this line: @ImplementedBy(classOf[DataRepository])
   * Have the repository extend the trait
   * Add all the `defs` you want to mock to the `trait` as abstract methods (find out what purpose abstract methods/classes serve!)
   * Then in your test suite, have your `mockRepository` val mock the trait instead of the class
   * Note: You will have to inject the trait in all places where you injected the repository class, e.g. the service layer
   * This avoids having us to mock the `PlayMongoRepository` extension, our only concern is mocking the methods we made

8. Use the html files in the `views` package to display at least one book from the Google Books API. Do this by
   * using the connector to retrieve the book by searching for the `isbn` in the url (you may have to change the structure of `dataModels`),
   * store the book in Mongo to show you have it
   * then display the book in your browser by returning the JSON of your book model
   * call made at browser url → controller → service → connector → Google Books → connector → service → views → service → controller

## Conclusion

Excellent, you've now created a RESTful API with CRUD functions, a connector to retrieve data from other APIs, unit tested them with MockFactory and learnt some useful CURL commands.
