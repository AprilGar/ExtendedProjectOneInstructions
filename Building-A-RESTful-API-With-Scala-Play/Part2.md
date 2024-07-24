# Part Two - CRUD Controller Actions
In this section we'll give our API the ability to create, read, update and delete (CRUD) data in a Mongo database and create asynchronous actions in a new Controller. **[More on CRUD here](https://www.codecademy.com/articles/what-is-crud)**.

## Create A New Controller
1. Create a new Scala file in the controllers package called ApplicationController
    * Right click the controllers package → New → Scala Class

2. Extend the `ApplicationController` with `BaseController`. This will give us access to a number of functions for our actions
    * You'll notice there is now an error stating that ControllerComponents must be implemented - this is a dependency of the BaseController trait. This class contains values necessary in most frontend controllers, such as multi-language support. To remedy this, change the class signature to the following:
        ```scala
        @Singleton
        class ApplicationController @Inject()(val controllerComponents: ControllerComponents) extends BaseController
        ```
    * We won't go much into dependency injection (DI) as part of this, but using DI you inject the objects needed by a class typically through a constructor
    * Import `javax.inject.Inject` and `play.api.mvc.ControllerComponents` when prompted

3. Write a Scala method in the controller called index() that simply returns TODO
    ```scala
    def index() = TODO
    ```
    * TODO is a Play feature that is essentially a default page for controller actions that haven't been completed. It is a useful way of keeping your app functioning while building functionality incrementally.

4. Before we can see the TODO page, we need to add an app route that references the new controller and method. In the routes file, add the following:
    ```
    GET     /api     controllers.ApplicationController.index
    ```
    
5. Run your application and hit the new route. You should see the TODO text.
6. Create 4 more Scala methods and matching routes, stubbed with TODO, with the following requirements:

    | Method Name | Parameters | HTTP Method |
    |-------------|------------|-------------|
    | create      | None       | POST        |
    | read        | id: String | GET         |
    | update      | id: String | PUT         |
    | delete      | id: String | DELETE      |

    Hint: to reference a parameter in the routes file, the route should be formatted like this, so it can pass a String parameter in the URL to the controller method:
    ```
    GET     /route/:id     controllers.ControllerName.methodName(id: String)
    ```

7. Try accessing the 'read' route in the browser with a dummy id as a URL parameter

## Test the New Controller
For almost every file in the `app/` directory of our project, there should be a corresponding test file. We call these files the exact same name as the file they are testing with the suffix `Spec`, and put them in a matching folder to where they sit in the `app/` package in the `test/` package.

1. Create a new Scala file to hold tests for your new controller. Where should it sit and what should it be called?

2. Extend the new file with:
    * `BaseSpecWithApplication` (import `baseSpec.BaseSpecWithApplication`) - this contains many test helpers for writing BDD, doing assertions etc. Note this isn't by default in Play, I've added this to construct Mongo easier
    
3. At the top of the spec, within the curly brackets, add the following lines:
    ```scala
    val TestApplicationController = new ApplicationController(
        component
    )
   
    ```
    * This creates a test version of your controller that you can reuse for testing and call your new Action methods on.
    * Note `component` comes from the BaseSpecWithApplication, it is created as an instance of controller components and injected into the Controller.


For each controller action, we want a separate suite of tests. One way to structure the tests is the following:

   ```scala
   "ApplicationController .index()" should {
    
   }
    
   "ApplicationController .create()" should {
    
   }
   ```
1. Add this underneath the code you copied in step 3

2. Add placeholders for the remaining three controller methods using the same structure

### Do a commit
Now is probably a good time to make a commit. Run your (placeholder) tests first, then commit your changes and push to GitHub.

### Checking the response HTTP status
To run tests against specific methods in the controller, we will use the `TestApplicationController` and simply call the methods directly.
Add the following imports:

   ```scala
   import play.api.test.FakeRequest
   import play.api.http.Status
   import play.api.test.Helpers._
   ```
    
And add the following code:

   ```scala
   "ApplicationController .index" should {
    
     val result = TestApplicationController.index()(FakeRequest())
   
     "return TODO" in {
        status(result) shouldBe Status.NOT_IMPLEMENTED
     }
   }
   ```
Breaking down the above:
* Line 3
    * Creates a new value `result` and assigns it the outcome of calling the function `index()` on the controller
    * The `FakeRequest()` is needed to mimic an incoming HTTP request, the same as hitting the route in the browser.

* Line 6
    * Uses a helper method called `status()` to pull out the HTTP response status of calling the function
    * `shouldBe` is just one of the many ways of doing assertions in unit tests
    * Because Play's TODO method actually returns a 501 / NOT_IMPLEMENTED response, we can assert that the code we wrote in our controller means that a HTTP 501 is returned
    * `status(result) shouldBe Status.NOT_IMPLEMENTED` is the same as writing `status(result) shouldBe 501`

Run the tests! Do they all pass?

1. Change `Status.NOT_IMPLEMENTED` to `Status.OK`

2. Run the tests and watch them fail

3. See if you can change the `ApplicationController.index()` to return a 200 OK response and fulfill the test. Hint: look in `play.api.mvc.Results`. Be sure to update the test description.

4. Run the tests again and check they pass

5. Congrats, you've just done **test-driven development**

## Data Access
Before we can build our controller actions, we'll add a data access layer and design a structure for our data to surface some data to our controller.

### Data Models
To design how our data should look, we can use what are known as 'models' in the Scala world. Models are similar to classes and create data structures based off the parameters you pass in. These parameters are immutable (can't be changed).
A simple model:
   ```scala
   case class DataModel(_id: String,
                        name: String,
                        description: String,
                        pageCount: Int)
   ```
To create a new instance of that model, you can do either of the following:
   ```scala
   val bookOne = DataModel("id1", "Book name", "Author name", 10)
   ```
Or
   ```scala
   val bookOne = DataModel(_id = "id1", name = "Book name", description = "Author name", pageCount = 10)
   ```

1. Create a new folder in your project called `models` inside the `app` directory

2. In that directory, create a new Scala file named `DataModel.scala`

3. Create a model called DataModel using the example, with the following required fields:

    | Name        | Data Type |
    |-------------|-----------|
    | _id         | String    |
    | name        | String    |
    | description | String    |
    | pageCount   | Int       |
    
4. Underneath your model, add the following code:    
   ```scala
   object DataModel {
     implicit val formats: OFormat[DataModel] = Json.format[DataModel]
   }
   ```
This allows for easily transforming the model to and from JSON.

### Data Access Layer
Next we will define the contract for our data access layer. Traits are similar to Interfaces in Java – more info [here](http://docs.scala-lang.org/tutorials/tour/traits.html).

1. Create a new folder called `repositories` inside the `app` directory. 

2. In that directory, create a new Scala file named `DataRepository.scala` with the following content:
    ```scala
    package repositories

    import models.DataModel
    import org.mongodb.scala.bson.conversions.Bson
    import org.mongodb.scala.model.Filters.empty
    import org.mongodb.scala.model._
    import org.mongodb.scala.result
    import uk.gov.hmrc.mongo.MongoComponent
    import uk.gov.hmrc.mongo.play.json.PlayMongoRepository
    
    import javax.inject.{Inject, Singleton}
    import scala.concurrent.{ExecutionContext, Future}
    
    @Singleton
    class DataRepository @Inject()(
      mongoComponent: MongoComponent
    )(implicit ec: ExecutionContext) extends PlayMongoRepository[DataModel](
      collectionName = "dataModels",
      mongoComponent = mongoComponent,
      domainFormat = DataModel.formats,
      indexes = Seq(IndexModel(
        Indexes.ascending("_id")
      )),
      replaceIndexes = false
    ) {
    
      def index(): Future[Either[Int, Seq[DataModel]]]  =
    collection.find().toFuture().map{
      case books: Seq[DataModel] => Right(books)
      case _ => Left(404)
    }
   
      def create(book: DataModel): Future[DataModel] = 
        collection
          .insertOne(book)
          .toFuture()
          .map(_ => book)
    
      private def byID(id: String): Bson =
        Filters.and(
          Filters.equal("_id", id)
        )
    
      def read(id: String): Future[DataModel] =
        collection.find(byID(id)).headOption flatMap {
          case Some(data) =>
            Future(data)
         } 
    
      def update(id: String, book: DataModel): Future[result.UpdateResult] = 
        collection.replaceOne(
          filter = byID(id),
          replacement = book,
          options = new ReplaceOptions().upsert(true) //What happens when we set this to false?
        ).toFuture()
    
      def delete(id: String): Future[result.DeleteResult] =
        collection.deleteOne(
        filter = byID(id)
       ).toFuture()
    
      def deleteAll(): Future[Unit] = collection.deleteMany(empty()).toFuture().map(_ => ()) //Hint: needed for tests
    
    }

    ```
There's a lot going on so let's try to break it down:

```scala
@Singleton
class DataRepository @Inject()(
                                mongoComponent: MongoComponent
                              )(implicit ec: ExecutionContext) extends PlayMongoRepository[DataModel]
```
* This section creates a new DataRepository class and injects dependencies into it required for every Mongo Repository. `extends PlayMongoRepository[DataModel]` tells the library what the structure of our data looks like by using our newly created `DataModel`.
This means that every document inserted into the database has the same structure, with `_id`, `name`, `description` and `pageCount` properties.
For this database structure, each document will be identified by the `_id`.

```scala
collectionName = "dataModels",
mongoComponent = mongoComponent,
domainFormat = DataModel.formats,
indexes = Seq(IndexModel(
  Indexes.ascending("_id")
)),
replaceIndexes = false
```

* These lines pass in required parameters to the PlayMongoRepository abstract class.
    * `"dataModels"` is the name of the collection (you can set this to whatever you like).
    * `DataModel.formats` uses the `implicit val formats` we created earlier. It tells the driver how to read and write between a `DataModel` and `JSON` (the format that data is stored in Mongo)
    * `indexes` is shows the structure of the data stored in Mongo, notice we can ensure the bookId to be unique

```scala
  def create(book: DataModel): Future[DataModel] = 
    collection
      .insertOne(book)
      .toFuture()
      .map(_ => book)

  private def byID(id: String): Bson =
    Filters.and(
      Filters.equal("bookId", id)
    )

  def read(id: String): Future[DataModel] =
    collection.find(byID(id)).headOption flatMap {
      case Some(data) =>
        Future(data)
    } 

  def update(id: String, book: DataModel): Future[result.UpdateResult] = 
    collection.replaceOne(
      filter = byID(id),
      replacement = book,
      options = new ReplaceOptions().upsert(true) //What happens when we set this to false?
    ).toFuture()

  def delete(id: String): Future[result.DeleteResult] =
    collection.deleteOne(
      filter = byID(id)
    ).toFuture()

  def deleteAll(): Future[Unit] = collection.deleteMany(empty()).toFuture().map(_ => ()) //Hint: needed for tests

```
* Each of these methods correspond to a CRUD function. 
  * `create()` adds a DataModel object to the database
  * `read()` retrieves a DataModel object from the database. It uses an `id` parameter to find the data its looking for
  * `update()` takes in a DataModel, finds a matching document with the same `id` and updates the document. It then returns the updated DataModel
  * `delete()` deletes a document in the database that matches the `id` passed in
  * `delteAll()` is similar to delete, this removes all data from Mongo with the same collection name
* All of the return types of these functions are [asynchronous futures](https://www.playframework.com/documentation/2.6.x/ScalaAsync).

## The Controller
At this point, we can return to the controller to round out the implementation details there. Before we can start adding implementation details, we need to configure some dependency injections.
We will inject the DataRepository into our Controller, then use that to create our repository. 

**IMPORTANT: From here on if you are testing the controller with an instance of the repository, ensure mongo is running in docker on port 27017**

1. Update the signature of `ApplicationController` so that `DataRepository` is injected as a dependency, as per previous examples. You will also need to update the tests to inject an instance of your repository, there should be a `repository` val in `BaseSpecWithApplication`

2. Also inject `implicit val ec: ExecutionContext` as a dependency. ExecutionContext is needed in asynchronous code as it lets Scala decide where in the thread pool to execute the related function

### The Index Action
There is an unimplemented `index()` method. This is meant to display a list of all DataModels in the database, selected without parameters. The implementation for this is very simple.

Update `index()` to the following:
   ```scala
     def index(): Action[AnyContent] = Action.async { implicit request =>
      dataRepository.index().map{ 
        case Right(item: Seq[DataModel]) => Ok {Json.toJson(item)}
        case Left(error) => Status(error)(Json.toJson("Unable to find any books"))
      }
    }
   ```
* `.find()` is a built-in method in the library we're using, and will return all items in the `data` repository.
* The result returned by this method is a `Future` - essentially a placeholder for the result of performing the lookup operation in the database. We use `.map(items => Json.toJson(items)).map(result => Ok(result))` to write what we want to do with the result.
* In this case, we take the resulting object (of type `Seq[DataModel]`), transform it into JSON, and return it in the body of an `Ok` / 200 response

### Read and Delete Actions
See if you can complete two more methods, using the appropriate methods created in `DataRepository` with the following prompts:
* `read()` should return a HTTP OK response, with a body containing the item(s) it found
* `delete()` should return a HTTP ACCEPTED response, with no body

### The Create Action
The remaining `create()` and `update()` methods introduce a small amount of complexity in that they require the request body to be parsed using Scala’s built in pattern matching functionality. 

For the `create()` method, we’ll apply a JSON [Body Parser](https://www.playframework.com/documentation/2.6.x/ScalaBodyParsers) to a request. Then, we'll use a `validate()` method in the Play JSON library to check that the request body contains all of the fields with the correct types needed to create a `DataModel`.
We use a `match` statement with `JsSuccess` and `JsError` to deal with the two possible outcomes: a valid or invalid request.
```scala
def create(): Action[JsValue] = Action.async(parse.json) { implicit request =>
    request.body.validate[DataModel] match {
        case JsSuccess(dataModel, _) =>
            dataRepository.create(dataModel).map(_ => Created)
        case JsError(_) => Future(BadRequest)
    }
}
```
**Why is `BadRequest` wrapped in a `Future`?**
- The result of `dataRepository.create()` is a `Future[Result]`, so even though we're not doing any lookup here, the type must be the same    

**What does `_` mean in the `case` statements?**
- This is a character used when you don't care what the value of that field is. For `JsError(_)`, if we wanted to we could pull out the reason(s) the validation failed and log them somewhere.
- Have a look in the documentation for the `JsSuccess` and `JsError` case classes to see what other properties they have. Hint: start [here](https://www.playframework.com/documentation/2.6.x/api/scala/play/api/libs/json/index.html)

**How would check something is created in the database?**
- Think about what is returned form `.create()` and how you can match on this similar to validating the request body

### The Update Action
See if you can apply what you've seen so far to the `update()` function and complete the code.
* It should take in an `id` URL parameter, similar to `read()` and `delete()`
* It should also take in a JSON body
* You need to validate the body in the same way as in the `create()` method
* If the validation is successful, it should return HTTP ACCEPTED, with the **new** updated DataModel (in the form of JSON) in the body of the response. 
  You can either simply return the body from the request or .read() for the id of the new body.

## [Part 3](Part3.md)