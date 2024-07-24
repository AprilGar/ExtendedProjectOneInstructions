# Part Six - FrontEnd

Scala can use .scala.html files to displayed pages in the browser as the Frontend of your project. This section will explain how and where to create these files, add styling and submit information from the browser to your project using forms.

## Creating views files
1. Go to the views package inside app (`app/views`)
2. Inside views find the file called `main.scala.html`

    This main.scala.html file is responsible for rendering the head and body of your page. 
    Lines with an @ at the beginning signify that this line is Scala code.
    On line 7 you will see:
    ```
                 @(title: String)(content: Html)
    ```
    This line at the top of the main file has an @ at the beginning, therefore it is Scala code. Inside the first argument, (title: String), is outlining the name and data type of the parameters this file needs. Like a Scala method does, e.g. def scalaMethod(title: String)
    The second argument (content: Html) will be used to display the content of your page in the body tag.
    Inside the head and body tags you can see the names we have given to these arguments (title and content) with an @ in front of them as they are Scala code and this is where we want these arguments to be rendered.
    All the views files you create will be passed to this `main.scala.html` file which will be responsible for rendering the title and content in the head and body tags.
4. Let's create a page, when creating this page call it `example.scala.html`. Make a new file in your views folder give it a name.
    ```
                    @()
                    @main("Find Book"){
                        <h1>Google Book Api Project</h1>
                    }
    ```
    @main is calling your main file and providing it the necessary arguments, ("Find Book") is the name we have given as this pages title. Inside the {} we provide the content of this page, which we have specified in `main.scala.html` is HTML and is currently a heading. 
    Here the first line, @(), is saying this file does not accept any arguments as the () are empty. 


## Loading views files through the controller

To see these views pages in the browser we need to have a method that loads these pages in the controller.
For example:
```
            def example(): Action[AnyContent] = Action.async {implicit request =>
                Future.successful(Ok(views.html.example()))
            }      
```
When creating a scala.html page, an identical twirl template will also be created. This is what the above is referring to and can be found here: `target/scala-2.13/twirl/main/views.html`. If the scala.html file isn't found/is red when writing the method above, you need to run `sbt compile` in terminal, this will generate the corresponding twirl template. 

Then in your `conf/routes` file you would access this method like:
```
                GET     /example                           controllers.ApplicationController.example()
```

You can see this page in your browser by following this route.
1. Run your project and open up this page in your browser. You should see a page with the heading "Google Book Api Project"
2. Now try and load a book on this page. Provide your DataModel as an argument for this views file and display the DataModel parameters in the body of this page. 
    You can create a new method and route to do this or if you already have a method that is returning a DataModel you can use this.
    Remember your DataModel parameters are Scala code.
    If you give this method different parameters does it display a different book on this page?

## Adding some styling

Now we will add some styling to our scala.html page. You may have noticed this line in your `conf/routes` file:
```
                GET     /assets/*file               controllers.Assets.versioned(path="/public", file: Asset)
```
We can use this to load our css file and add some styling to our views.
Inside `app` make a folder called `assets` and inside this make a folder called `stylesheets` then a file called `main.css`(`app/assets/stylesheets/main.css`).
This will be where you can added you css code to apply style to your page.
In your `views/main.scala.html` in the `<head>` tag add this line:
```
                <link rel="stylesheet" type="text/css" media="screen" href="@routes.Assets.versioned("/stylesheets/main.css")">
```
The href is pointing to the routes file in `conf/routes` and the path `Assets.versioned(path="/public", file: Asset)` which we have given the path to the file we want `"/stylesheets/main.css"`.
1. Add some css styling to your views page that is displaying your book.

## Submit a form

You should already have a method to create a new book that then adds this book to your database. Let's use this and create a new book from the browser.
1. Create a form for your DataModel.
    You can start by creating a Form in your DataModel companion object that contains all the parameters of your DataModel case class.
    Add these imports to your DataModel file:
    ```
                import play.api.data._
                import play.api.data.Forms._

    ```
    Then in the DataModel object create a form that uses the parameters and the exact names of the case class.
    For example:
    ```
                case class Person(name: String, surname: String, age: Int)
                object Person {
                    implicit val formats: OFormat[Person] = Json.format[Person]

                    val personForm: Form[Person] = Form(
                        mapping(
                            "name" -> text,
                            "surname" -> text,
                            "age" -> number
                        )(Person.apply)(Person.unapply)
                    )         
                }           
    ```
    Use this example to make your DataModel form.

2. Create a new views file called to add a new book (DataModel).
    In your view file create a form containing all the parameters needed to create a DataModel.
    For example:
    ```
                @import models.Person
                @import helper._

                @(personForm: Form[Person])(implicit request: RequestHeader, messages: Messages)

                @main("Add Person"){
                <div>
                    @helper.form(action = routes.ApplicationController.addPersonForm()) {
                        @helper.CSRF.formField
                        @helper.inputText(personForm("name"))
                        @helper.inputText(personForm("surname"))
                        @helper.inputText(personForm("age"))
                        <input class="submitButton" type="submit" value="Submit">
                    }
                </div>
                }
    ```
    The `action = routes.ApplicationController.addPersonForm()` will be a POST route that we are now going to create in the `conf/routes` file.

3. In your `conf/routes` create two new paths one with a http GET request and one with a POST request, the GET request will be the route that will load your form page that you created in the views folder.
    ```
                    GET     /addperson/form     controllers.ApplicationController.addPerson()
                    POST     /addperson/form     controllers.ApplicationController.addPersonForm()
    ```
    We will now need to create these methods in the controller.

4. In your ApplicationController have the class extend the trait play.api.i18n.I18nSupport by writing `with play.api.i18n.I18nSupport`
5. In your controller create a method for your new GET request that will load your new views page.
6. Now add a method for your POST request and for now leave the body of the method as ???
    For example:
    ```
                    def addPersonForm() = ???
    ```
7. Now if you run your project and in your browser go to your new GET route you should see your form page. You will not be able to submit the form yet to do this we need to write the body of the method that you point to in your POST route.
8. In your controller add this method:
    ```
                def accessToken(implicit request: Request[_]) = {
                    CSRF.getToken
                }
    ```
    We need to call this method in the POST method. The CSRF (Cross Site Request Forgery) (Have a look at this website for a description and why we use it in forms: https://www.playframework.com/documentation/2.8.x/ScalaCsrf)

9. Let's write the POST route method.
    ```
               def addBookForm(): Action[AnyContent] =  Action.async {implicit request =>
                    accessToken //call the accessToken method
                    personForm.bindFromRequest().fold( //from the implicit request we want to bind this to the form in our companion object
                        formWithErrors => { 
                        //here write what you want to do if the form has errors
                        },
                        formData => {
                        //here write how you would use this data to create a new book (DataModel) 
                        }
                    )
                } 
    ```
    When you have written this run your application and test the form to see if it creates a new DataModel.
For more information on forms:
https://www.playframework.com/documentation/2.8.x/ScalaForms#Passing-MessagesProvider-to-Form-Helpers 

10. Write tests for this method.
    <details>
    <summary>Hint</summary>
    
    For these tests you need to mimic all the things a real request to this method would have, like the CSRFToken and the form data.

    <details><summary>Hint on hint</summary><blockquote>
        On your FakeRequest look at adding .withCSRFToken for the CSRFToken and adding .withFormUrlEncodedBody("nameOf YourParameter" -> "name", ...) for the form data.
    </blockquote></details>
    </details>
    

