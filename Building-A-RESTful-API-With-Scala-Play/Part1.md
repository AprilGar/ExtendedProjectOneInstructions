# Part One - Setup

## Pre-requisites
These should all be installed as part of your laptop setup.
* Scala
* Mongo and Docker
* git
* IntelliJ IDEA
* SBT

## Generating a Play/Scala application seed 
We'll first generate a template for your Play application that gives you the initial setup you need. **Note: This won't compile until the dependencies have been added later**

From the [Template](https://github.com/AprilGar/ExtendedProjectOneTemplate):
1. Select code → copy the SSH

2. Navigate in your terminal to an appropriate location on your computer

3. Enter `git clone "PASTE THE SSH LINK"`, e.g. `git clone git@github.com:MercatorDigital/play-template.git`


4. From **IntelliJ**:
   * File → Open → find your project
   * Check the project JDK is setup to Java 1.8
   * Hit OK

Take a few minutes to get familiar with what has been created for you. In particular:
* `app/controllers/HomeController`
* `index.scala.html` & `main.scala.html`
* `conf/routes`
* `test/controllers/HomeControllerSpec`
* `build.sbt`
* `.gitignore`

## Set up Git & GitHub Repository
We want to track changes to the project using git. Since we cloned this repo we are going to change the remote origin.

1. Go to your GitHub account → Your repositories → Create new repository
   * Give it the same name as your local project
   * Ensure it is public
   * Ensure 'Initialise with README' is unticked

2. Go to the command line/terminal:
   * We need to tell git where the remote GitHub repository is, so that we can push up changes. Run the following, substituting with your GitHub username and newly created project name
   * `git remote rename origin upstream` (The old origin is now renamed to upstream)
   * `git remote add origin git@github.com:<username>/<repo-name>.git` (A new origin is added, your github!)
   * e.g. `git remote add origin git@github.com:MercatorDigital/play-template.git`

3. Run `git remote -v` to check the remote is correct

4. Run `git status`, and you can now see there are various files git has picked up but is not yet tracking any changes and only recognises a whole bunch of new folders and files.

5. Run `git add .` to add all files in red to a 'staging area' in preparation for a final commit

6. To make a commit, run `git commit -m "example message"` (Think of an appropriate message for the changes you've made)

7. Push your new commit up to main `git push origin main` (Note: there are two places we 'can' push to, the old location, `upstream`, and your new one, `origin`.)

8. Go to your new project on GitHub to check everything pushed up okay.

9. Hit the button **Add a README** if a README doesn't exist.

GitHub supports making direct changes to your repo within the web UI itself. This isn't generally recommended as you're unable to run tests to ensure your changes are working okay, but we can use this feature just to create a dummy README.md file:
1. Add a commit comment describing what you're doing

2. Commit the new file

3. Back to the command line, run git pull to pull down the new README

4. You should get an error from git, informing you that you need to give it an upstream origin for the master branch

5. Using the help text git gives you, **try and fix this error**

6. Run git pull again and check you now see the README.md file

**From this point onwards, we'll now use branches to make changes, rather than committing straight to master (big no no!)**

## Checkout A New Branch
To create and checkout a new branch:

   ```
   git checkout -b <branch-name>
   ```
Keep the branch name short and sweet. If your branch on a real microservice relates to a JIRA ticket, use the ID of the ticket as the branch name.

## Installing Dependencies
As mentioned earlier, we're going to use Reactive Mongo in our project to make basic requests to a database.
We need to use a separate library for this, and we'll be using the HMRC version that is used widely across Multi-Channel Digital Tax Platform (MDTP). We'll also add a useful testing library.

We're going to first change the version of Play framework we're using so that it works with simple-reactivemongo. We'll also need to change a couple other dependency versions (If they haven't been changed already).

In your project run `sbt test` to check everything works and your app downloads the versions correctly. If not, ensure:

1. Find the version of `sbt-plugin` and change it to **2.8.18**

2. Find the `scala` version and change it to **2.13.8**

3. Find the `sbt` version and change it to **1.6.2**

To add a new library dependency to your project, open up `build.sbt`

1. Add the following resolver to the file. This tells sbt where to find any public HMRC libraries:
    ```scala
    resolvers += "HMRC-open-artefacts-maven2" at "https://open.artefacts.tax.service.gov.uk/maven2"
    ```

2. Add the new mongo library, some other important dependencies for testing, and versions to the `build.sbt` file:
    ```scala
    libraryDependencies ++= Seq(
   "uk.gov.hmrc.mongo"      %% "hmrc-mongo-play-28"   % "0.63.0",
   guice,
   "org.scalatest"          %% "scalatest"               % "3.2.15"             % Test,
   "org.scalamock"          %% "scalamock"               % "5.2.0"             % Test,
   "org.scalatestplus.play" %% "scalatestplus-play"   % "5.1.0"          % Test
   )
   ```

4. Update the `lazy val root` in `build.sbt`:
    ```scala
    lazy val root = (project in file("."))
   .settings(
   name := "some_project_name"
   )
   .enablePlugins(PlayScala)
    ```

5. Some extra configuration is needed for the reactive mongo library to know how to connect to your local Mongo database and to give it a name. Add the following to application.conf and adding in the name of your app:
    ```
    mongodb {
      uri = "mongodb://localhost:27017/replace-with-your-app-name-here"
    }
    ```
6. Run tests again and check everything is working okay

7. Add the changed files to git **on a new branch**, do a git commit briefly detailing what you have changed, then push the branch up to GitHub (hint: `git push origin <branch-name>`)

8. Navigate to GitHub and find your new branch and view the changes you have made. Once you're happy with them then, a pull request can be made to merge the branch into main.

## Running the new Application
1. Bring up the Terminal window in IntelliJ and run `sbt run`

2. Your project will compile and run by default on port `9000`

3. Navigate to `localhost:9000` in your browser and you should see a greeting message from Play

4. Where is this text coming from? Use the files in the `controllers` & `views` packages to investigate. More in-depth frontend development will be covered in a separate tutorial.

5. Try customising both the header and page title text in those files to your own message

6. Use `control + C` in the terminal to stop the server running 

## Running Tests
1. Run the command `sbt test` in the terminal. This command runs all tests that were set up by the Play application seed, i.e. in `test/controllers/HomeControllerSpec`.

2. You should get failed tests after changing the page title text above

3. Alter the tests so that they now pass

**Note: this isn't [Test-Driven Development](https://www.agilealliance.org/glossary/tdd/#q=~(infinite~false~filters~(postType~(~%27page~%27post~%27aa_book~%27aa_event_session~%27aa_experience_report~%27aa_glossary~%27aa_research_paper~%27aa_video)~tags~(~%27tdd))~searchTerm~%27~sort~false~sortDirection~%27asc~page~1))! Consider how you could have done the above tasks using a TDD approach.**

## [Part 2](Part2.md)
