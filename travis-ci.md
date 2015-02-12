####Travis CI 

####What is Continous Integration?

> “Continuous Integration doesn’t get rid of bugs, but it does make them dramatically easier to find and remove.”
-Martin Fowler

###Setup

We need to configure an additional file in our project so that our project can be integrated with Travis CI, first though, we need to do the neccessary online setup. 

####Step 1

We'll need to setup a Githup repo( do that now by creating an empty repo on Github).

On our computer we'll do the following:

```bash
/setup a new empty travis repo
$ rails new travis-test -d postgresql —quiet

$ cd travis-test

//scaffold a User so some tests are created
$ rails g scaffold User name email

//create a repo on Github then on command line
$ echo "# travis-test" >> README.md
$ git init
$ git add README.md
$ git commit -m "first commit”
// DO NOT use the git@github address below...use the one for YOUR repo on Github
//If you're not using SSH..you'll need to supply the https: address below
$ git remote add origin git@github.com:grant-roy/travis-test.git
$ git push -u origin master
```

#### Step 2

Sign in to [Travis CI](https://travis-ci.org/) with your GitHub id. Make sure to flip the switch to 'ON' for the repo you want to integrate with Travis, in this case 'travis-test'.

#### Step 3

Let's change into the travis-test directory, and then create a .travis.yml file. Take note of the preciding '.' in the file name, this marks it as a hidden file that will not be displayed by default in finder. Make sure the .travis.yml is added to the project root directory, at the same level as the Gemfile. 

```shell

$ cd ~/Documents/rails-workspace/travis-test

$ touch .travis.yml

```
We next need to fill in the basic language setup for travis to build ruby projects. To do this correctly we will determine the ruby version on our computer. The following command can be run from inside the project directory

```shell

$ ruby -v 
// output will look something like this:
// ruby 2.0.0p481 (2014-05-08 revision 45883) [universal.x86_64-darwin13]
```

Then in our .travis.yml file we can add this as the rvm version under the language specification. 

**NOTE:** If your ruby version is -2.1.2p95 or higher just use the above version as Travis won't work yet with these versions. 

.travis.yml
```yml

language: ruby
rvm:
  - 2.0.0-p481

```

Next we will install the travis linter gem. Why a linter??....don't get me started please:) LINT YOUR CODE!!!!! The linter will pick up any problems in the .travis.yml file, including spelling and syntax errors of field names. 

```shell

$ gem install travis-lint

//run the linter
$ travis-lint
```

If you remove the 'm' from the 'rvm' key in the file the linter will pick up an error 

![linter error](http://s3.amazonaws.com/grant-wdi/to-do-list/linter-error.png )

####Step 4 

To actually trigger Travis to run a build, all that we need to do is push a change to our origin master branch on Github. Travis already knows about this remote repo and is monitoring it for any changes. Once a change is detected Travis will attempt to build our project, and if successful, run any tests we have specified. 

If we were to push this change to Github, our Travis build would fail because of a database connection error, so we'll need to add some additional information to our .travis.yml, setting up a test database for Travis to use while running our tests.

First we will need to modify the test section of our database.yml file

database.yml
```yml
#test:
#  <<: *default
#  database: travis-test_test
test:
  adapter: postgresql
  database: travis_ci_test
  username: postgres


```

Next we need to adjust our .travis.yml file.

In .travis.yml we will need to run a psql script before our test suite runs that will create a test database. The full listing is below.  

.travis.yml
```yml

language: ruby
rvm:
  - 2.0.0-p481

//make sure the name here matchers your project name..i.e if you did rails new travis-test..use 
//travis_test like below
before_script:
  - psql -c 'create database travis_test_test;' -U postgres

```



Now commit these changes and push master to the remote, this will trigger a build that should pass

```bash

$ git add -A

$ git commit -m "Add postgres compatibility to .travis.yml"

```

![build passing](http://s3.amazonaws.com/grant-wdi/to-do-list/build-passing.png )


Also take note that all of the tests are running and passing. THIS IS HUGE!

![tests passing](http://s3.amazonaws.com/grant-wdi/to-do-list/tests-passing.png )


###Deployment

One of the key principles of Continuous Integration is _Automated Deployment_. The idea is that you merge&&push your changes to the remote master, those changes are built and tested, and if all tests pass the build is automatically deployed to the server. 

####Step 1 

We will configure this project to deploy to Heroku since that is what we have been doing thus far. So first we will install the actual Travis gem. This will install the command line tools that will help us with the neccessary configuration

```bash

//this may take a minute to run as the travis gem has several dependencies
$ gem install travis -v 1.7.5 --no-rdoc --no-ri

//make sure this outputs: '1.7.5'
$ travis -v

```
####Step 2

Add a deploy section to our .travis.yml file

.travis.yml
```yml
deploy:
  provider: heroku
  api_key: "API KEY"

```

####Step 3 

We now need to setup our API key properly. You absolutely do not want to have your API key just hanging in the wind on your public Github repo. Suprisingly this was a real problem with AWS for awhile and many other hosting providers, developers were unwittingly pushing repos to Github that contained secret access tokens. Don't ever be one of those developers. 

We'll need to take care to encrypt our access token. 

```bash
$ travis encrypt $(heroku auth:token) --add deploy.api_key

```

This command runs 'heroku auth:token', which tells the heroku command line tools to grab the auth token for the account, then travis encrypts that token and adds it to the correct section of .travis.yml file. Open that file after running the command and take a look at the long string it has written there. 

####Step 4

We'll specify a couple of additional fields in our .travis.yml before we are ready to test our automated deployment. We'll add an app name and also add some heroku commands to run so our app is kept up to date. 

Log in to Heroku and create an app, this time you get to be the creative one and choose your apps name. This is the app we'll hook up with travis. 


You can list out the name's of your apps with the following commands:

```bash
//list the apps so we can get our app name
$ heroku apps
```

Now, let's add in the remaining fields.

Full listing of .travis.yml should look like this:
```yml
language: ruby
rvm:
- 2.0.0-p481
before_script:
- psql -c 'create database travis_ci_test;' -U postgres
deploy:
  provider: heroku
  api_key:
    secure: KxoeHbmu3K4czIjYv/MdfDloLgU2MzT9VrviiNfEc08if9U+brOHFC4pKAB9e63omxpojB3dNHGwMeKPccOSHpAxPvQJ0UK4+mcgi1eLjRCtHsu3K88tLoesAFDW/cXKxpLkIEOtPS72o3t+lNdBA60cqOAP87fb0y5ULSBLtqQ=
  app: travis-test-app
  run:
    - "rake db:migrate"
    - restart

```

You'll one again need to add and commit all these changes to git and push the repo. 

We added an **app** field and **run** field under **deploy**. Take care, all of the fields under deploy must be indented to indicate they are part of the deploy group, this is how yaml works. 

It's important to run **rake db:migrate** because as your models change you want these changes to automatically be propagated to your production environment, adding in a rake db:migrate to the deployment process takes care of that. 

Let's take a momement to appreciate the magnitude of what we have just done. We set up a **proccess** whereby all we have to do is check code in to master, the code is built and tested, and **if** and **only** if all our tests pass, our code is autodeployed onto our production server.  It's really difficult to overstate the importance of this process...so just imagine that I have stated the above a jillion times. 





