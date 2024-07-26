# Part Four - Services and Connectors
There are some actions that should be separated from Controllers, one of which is 
communicating down stream with other APIs/services. We will use connectors for this.

## Create a connector
1. Create a new `connectors` package in app, then create a new LibraryConnector class
   * Right-click the connectors package → New → Scala Class
   
2. In order to communicate with other services we need to make HTTP requests through urls,
    we will do that through a connector. To start, we'll need another dependency in `build.sbt`

    ```scala
    libraryDependencies += ws
    ```
3. WS client uses Play specific classes in the request and response building.
   We can access this in this through dependency injection. Add the below into your new `LibraryConnector`.

    ```scala
   class LibraryConnector @Inject()(ws: WSClient) {
      def get[Response](url: String)(implicit rds: OFormat[Response], ec: ExecutionContext): Future[Response] = {
        val request = ws.url(url)
        val response = request.get()
        response.map {
          result =>
            result.json.as[Response]
        }
      }
   }
   ```
   There is documentation for WS client here https://www.playframework.com/documentation/latest/ScalaWS.
   There are a couple of things used in `.get[Response]()` that need explaining:
   * This method is used for a GET method, the url we are calling must expect this.
   * The `implicit rds: OFormat[Response]` is needed to parse the Json response model as our model.
   * `[Response]` is a type parameter for our method, this must be defined when calling the method. This allows us 
   to use this method for several models, e.g. `get[Tea]("tea.com")` and `get[Coffee]("coffee.com")`.
   * `ws.url(url)` creates our request using the url.
   * The response is made using WSClient's `.get()` method. 
   * Finally, the result's json value is parsed as our response model.

   Note: We are making many assumptions here including expecting a json body in the response, 
   that the json body can be parsed into our model and that our request made was a success. All 
   of this will need error handling, this will be an extension task.

4. Now we have our connector we are going to need to talk to it. At first, we might think to talk to our connector
   via the controller, however this could lead to a messy controller that is hard to refactor if changes later come (They almost always do!).
   Therefore, we need to make another package in app called `services`.
   * Right-click the new services package → New → Scala Class and give this a basic name like `ApplicationService`.
   * You're service should inject the connector so we can access out `.get[Reponse]()` method. Add in: 
   ```scala
   @Singleton
   class LibraryService @Inject()(connector: LibraryConnector) {

      def getGoogleBook(urlOverride: Option[String] = None, search: String, term: String)(implicit ec: ExecutionContext): Future[Book] =
         connector.get[DataModel](urlOverride.getOrElse(s"https://www.googleapis.com/books/v1/volumes?q=$search%$term"))

   }
   ```
   * Let's break down this method: 
      * We are going to be using [Google Books APIs](https://developers.google.com/books/docs/overview?hl=en) as they have a huge free database of books
        (This will make it a lot easier than entering all the data in yourself!)
      * This method takes a `urlOverride`, `search` and `term`. `urlOverride` is a way of providing the full url, 
        `term` is the special keyword that can allow searching in particular fields with `search`, e.g.
        `https://www.googleapis.com/books/v1/volumes?q=flowers+inauthor`. Learn more about this syntax in the link above.
      * We have also provided an example of a response body `Book`, go ahead and create a case class that can accept information from the Google APIs.
        Simply enter the search url with a `term` and `search` parameter into your browser, and it will show you the Json response body. You shouldn't include
        all the fields provided by Google APIs. Instead, choose the ones that match the fields of the DataModel you have already created.
      * (Hint: If you want to search for one book, each book has a unique isbn, International Standard Book Number, so use isbn as your `term`!)
   
5. Now we have our connector and service set up, we can integrate our controller and routes file.
   * Add this line to the `routes` file, note that this endpoint takes two parameters
     ```scala
        GET     /library/google/:search/:term      controllers.ApplicationController.getGoogleBook(search: String, term: String)
     ```
   * Add the corresponding method to your controller, it's common practice naming the controller method the same as/similar to the service method
    ```scala
       def getGoogleBook(search: String, term: String): Action[AnyContent] = Action.async { implicit request =>
         service.getGoogleBook(search = search, term = term).map { 
            ???
         }
       }
    ```
   * You'll notice that right now `service` doesn't exist in the controller, you will need to inject this in similar to the DataRepository
   * Fill in the missing implementation to return a `Ok` response, along with a Json body containing the book we found.

We have most things set up now, but we have no idea if what we have done is correct. Let's do some more testing in Part5!

## [Part 5](Part5.md)
