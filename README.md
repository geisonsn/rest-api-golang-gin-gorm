
# Building a REST API with Golang using Gin and Gorm - [Rahman Fadhil](https://blog.logrocket.com/author/rahmanfadhil/) at LogRocket Blog 
_**Editor’s note**: This tutorial was last updated on 10 November 2022 to make the code examples compatible with Go v1.19 and address questions posed in the comments section._

Go is a popular language for good reason. It offers similar performance to other “low-level” programming languages such as Java and C++, but it’s also incredibly simple, which makes the development experience delightful.

What if we could combine a fast programming language with a speedy web framework to build a high-performance RESTful API that can handle a crazy amount of traffic?

After doing a lot of research to find a fast and reliable framework for this beast, I came across a fantastic open-source project called [Gin](https://github.com/gin-gonic/gin).

Jump ahead:

*   [What is Gin?](#what-is-gin)
*   [Building a Rest API in Go using Gin and Gorm](#building-rest-api-go-gin-gorm)
*   [Setting up the server](#setting-up-server)
*   [Setting up the database](#setting-up-database)
*   [Setting up the RESTful routes](#setting-up-restful-routes)

What is Gin?
------------

The Gin framework is lightweight, well-documented, and, of course, extremely fast.

Unlike other Go web frameworks, Gin uses a custom version of [HttpRouter](https://github.com/julienschmidt/httprouter), which means it can navigate through your API routes faster than most frameworks out there. The creators also claim it can run 40 times faster than [Martini](https://github.com/go-martini/martini), a relatively similar framework to Gin. You can see a more detailed comparison in [this benchmark](https://github.com/gin-gonic/gin/blob/master/BENCHMARKS.md).

However, Gin is a microframework that doesn’t come with a ton of fancy features out of the box. It only gives you the essential tools to build an API, such as routing, form validation, etc.

For tasks such as authenticating users, uploading files, and sending emails, you need to either install another third-party library or implement them yourself.

This might not be the best fit for a small team of developers that needs to ship a lot of features in a very short time. Another web framework, such as [Laravel](https://laravel.com/) and [Ruby on Rails](https://rubyonrails.org/), might be more appropriate for such a team. Such frameworks are opinionated, easier to learn, and provide a lot of features out of the box, which enables you to develop a fully functioning web application in an instant.

This stack may be overkill if you’re part of a small team. But if you have the appetite to make a long-term investment, you can take advantage of Gin’s extraordinary performance and flexibility.

Building a REST API in Go using Gin and Gorm
--------------------------------------------

In this tutorial, we’ll demonstrate how to build a bookstore REST API that provides book data and performs CRUD operations.

Before we get begin, I’ll assume that you:

*   Have Go installed on your machine
*   Understand the basics of Go language
*   Have a general understanding of RESTful API

Let’s start by initializing a new [Go module](https://blog.golang.org/using-go-modules) to manage our project’s dependencies. Make sure you run this command inside your [Go environment folder](https://github.com/golang/go/wiki/GOPATH):

```
$ go mod init

```


Now let’s install [Gin](https://github.com/gin-gonic/gin) and [Gorm](https://github.com/jinzhu/gorm):

```
go get github.com/gin-gonic/gin gorm.io/gorm

```


After the installation is complete, your folder should contain two files: `go.mod` and `go.sum`. Both of these files contain information about the packages you installed, which is helpful when working with other developers. If somebody wants to contribute to the project, all they need to do is run the `go mod download` command on their terminal to install all the required dependencies on their machine.

For reference, the entire source code of this project on [GitHub](https://gist.github.com/Goodnessuc/314604e369497774a85f14c0ce6c485f). Feel free to poke around and work with the code.

```
$ git clone https://github.com/rahmanfadhil/gin-bookstore.git

```


Setting up the server
---------------------

Let’s start by creating a Hello World server inside the `main.go` file:

```
package main

import (
  "net/http"
  "github.com/gin-gonic/gin"
)

func main() {
  r := gin.Default()

  r.GET("/", func(c *gin.Context) {
    c.JSON(http.StatusOK, gin.H{"data": "hello world"})    
  })

  r.Run()
}

```


We first need to declare the `main` function that will be triggered whenever we run our application. Inside this function, we’ll initialize a new Gin router within the `r` variable. We’re using the `Default` router because Gin provides some useful middlewares we can use to debug our server.

Next, we’ll define a `GET` route to the `/` endpoint. If you’ve worked with other frameworks, such as [Express.js](https://expressjs.com/), [Flask](https://www.palletsprojects.com/p/flask), or [Sinatra](http://sinatrarb.com/), you should be familiar with this pattern.

To define a route, we need to specify two things: the endpoint and the handler. The endpoint is the path the client wants to fetch. For instance, if the user wants to grab all books in our bookstore, they’d fetch the `/books` endpoint. The handler, on the other hand, determines how we provide the data to the client. This is where we put our business logic, such as grabbing the data from the database, validating the user input, and so on.

We can send several types of response to the client, but RESTful APIs typically give the response in JSON format. To do that in Gin, we can use the `JSON` method provided from the request context. This method requires an [HTTP status code](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status) and a JSON response as the parameters.  
Lastly, we can run our server by simply invoking the `Run` method of our Gin instance.

To test it out, we’ll start our server by running the command below:

```
$ go run main.go

```


Setting up the database
-----------------------

The next thing we need to do is to build our database models.

Model is a class (or structs in Go) that allows us to communicate with a specific table in our database. In Gorm, we can create our models by defining a Go struct. This model will contain the properties that represent fields in our database table. Since we’re trying to build a bookstore API, let’s create a `Book` model:

```
// models/book.go

package models

type Book struct {
  ID     uint   `json:"id" gorm:"primary_key"`
  Title  string `json:"title"`
  Author string `json:"author"`
}

```


Our `Book` model is pretty straightforward. Each book should have a title and the author name that has a string data type, as well as an ID, which is a unique number to differentiate each book in our database.

We also specify the [tags](https://golang.org/ref/spec#Struct_types) on each field using backtick annotation. This allows us to map each field into a different name when we send them as a response since JSON and Go have different naming conventions.  
To organize our code a little bit, we can put this code inside a separate module called models.

Next, we need to create a utility function called `ConnectDatabase` that allows us to create a connection to the database and migrate our model’s schema. We can put this inside the `setup.go` file in our `models` module:

```
// models/setup.go

package models

import (
  "gorm.io/gorm"
  _ "gorm.io/driver/sqlite"
)

var DB *gorm.DB

func ConnectDatabase() {

        database, err := gorm.Open(sqlite.Open("test.db"), &gorm.Config{})

        if err != nil {
                panic("Failed to connect to database!")
        }

        err = database.AutoMigrate(&Book{})
        if err != nil {
                return
        }

        DB = database
}

```


Inside this function, we create a new connection with the `gorm.Open` method. Here, we specify which kind of database we plan to use and how to access it. Currently, Gorm only supports four types of SQL databases. For learning purposes, we’ll use [SQLite](https://www.sqlite.org/index.html) and store our data inside the `test.db` file. To connect our server to the database, we need to import the database’s driver, which is located inside the `gorm.io/driver/sqlite` module.

We also need to check whether the connection is created successfully. If it doesn’t, it will print out the error to the console and terminate the server.

Next, we migrate the database schema using `AutoMigrate`. Make sure to call this method on each model you have created.

Lastly, we populate the the `DB` variable with our database instance. We will use this variable in our controller to get access to our database.

In `main.go`, we need to call the following function before we run our app:

```
package main

import (
  "github.com/gin-gonic/gin"

  "github.com/rahmanfadhil/gin-bookstore/models" // new
)

func main() {
  r := gin.Default()

  models.ConnectDatabase() // new

  r.Run()
}

```


Setting up the RESTful routes
-----------------------------

We’re almost there!

The last thing we need to do is to implement our controllers. In the previous section, we learned how to create a route handler (i.e., controller) inside our `main.go` file. However, this approach makes our code much harder to maintain. Instead of doing that, we can put our controllers inside a separate module called `controllers`.

### The `Read` handler function

First, let’s implement the `FindBooks` controller:

```
// controllers/books.go

package controllers

import (
"net/http"

"github.com/gin-gonic/gin"
"github.com/rahmanfadhil/gin-bookstore/models"
)

// GET /books
// Get all books
func FindBooks(c *gin.Context) {
var books []models.Book
models.DB.Find(&books)

c.JSON(http.StatusOK, gin.H{"data": books})
}
```


Here, we have a `FindBooks` function that will return all books from our database. To get access to our model and `DB` instance, we need to import our `models` module at the top.

Next, we can register our function as a route handler in `main.go`:

```
package main

import (
  "github.com/gin-gonic/gin"

  "github.com/rahmanfadhil/gin-bookstore/models"
  "github.com/rahmanfadhil/gin-bookstore/controllers" // new
)

func main() {
  r := gin.Default()

  models.ConnectDatabase()

  r.GET("/books", controllers.FindBooks) // new

  r.Run()
}

```


Now, let’s run our server and hit the `/books` endpoint:

```
{
  "data": []
}

```


If you see an empty array as the result, it means your applications are working. We get this because we haven’t created a book yet. To do so, let’s create a create book controller.

### The `Create` handler function

To create a book, we need to have a schema that can validate the user’s input to prevent us from getting invalid data:

```
type CreateBookInput struct {
  Title  string `json:"title" binding:"required"`
  Author string `json:"author" binding:"required"`
}

```


The schema is very similar to our model. We don’t need the `ID` property since it will be generated automatically by the database.

Now we can use that schema in our controller:

```
// POST /books
// Create new book
func CreateBook(c *gin.Context) {
  // Validate input
  var input CreateBookInput
  if err := c.ShouldBindJSON(&input); err != nil {
    c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
    return
  }

  // Create book
  book := models.Book{Title: input.Title, Author: input.Author}
  models.DB.Create(&book)

  c.JSON(http.StatusOK, gin.H{"data": book})
}

```


We first validate the request body by using the `ShouldBindJSON` method and pass the schema. If the data is invalid, it will return a `400` error to the client and tell them which fields are invalid. Otherwise, it will create a new book, save it to the database, and return the book.

Now, we can add the `CreateBook` controller in `main.go`:

```
func main() {
  // ...

  r.GET("/books", controllers.FindBooks)
  r.POST("/books", controllers.CreateBook) // new

  r.Run()
}

```


So, if we try to send a POST request to `/books` endpoint with this request body:

```
{
  "title": "Start with Why",
  "author": "Simon Sinek"
}

```


The response should look like this:

```
{
  "data": {
    "id": 1,
    "title": "Start with Why",
    "author": "Simon Sinek"
  }
}

```


### The `Create` handler function for single data

We’ve successfully created our first book. Let’s add a controller that can fetch a single book:

```
// GET /books/:id
// Find a book
func FindBook(c *gin.Context) {  // Get model if exist
  var book models.Book

  if err := models.DB.Where("id = ?", c.Param("id")).First(&book).Error; err != nil {
    c.JSON(http.StatusBadRequest, gin.H{"error": "Record not found!"})
    return
  }

  c.JSON(http.StatusOK, gin.H{"data": book})
}

```


Our `FindBook` controller is pretty similar to the `FindBooks` controller. However, we only get the first book that matches the ID that we got from the request parameter. We also need to check whether the book exists by simply wrapping it inside an `if` statement.

Next, register it into your `main.go`:

```
func main() {
  // ...

  r.GET("/books", controllers.FindBooks)
  r.POST("/books", controllers.CreateBook)
  r.GET("/books/:id", controllers.FindBook) // new

  r.Run()
}

```


To get the `id` parameter, we need to specify it from the route path, as shown above.

Let’s run the server and fetch `/books/1` to get the book we just created:

```
{
  "data": {
    "id": 1,
    "title": "Start with Why",
    "author": "Simon Sinek"
  }
}

```


### The `Update` handler function

So far, so good. Now let’s add the `UpdateBook` controller to update an existing book. But before we do that, we need to define the schema for validating the user input first:

```
type struct UpdateBookInput {
  Title  string `json:"title"`
  Author string `json:"author"`  
}

```


The `UpdateBookInput` schema is pretty much the same as our `CreateBookInput`, except that we don’t need to make those fields required since the user doesn’t have to fill all the properties of the book.

To add the controller:

```
// PATCH /books/:id
// Update a book
func UpdateBook(c *gin.Context) {
  // Get model if exist
  var book models.Book
  if err := models.DB.Where("id = ?", c.Param("id")).First(&book).Error; err != nil {
    c.JSON(http.StatusBadRequest, gin.H{"error": "Record not found!"})
    return
  }

  // Validate input
  var input UpdateBookInput
  if err := c.ShouldBindJSON(&input); err != nil {
    c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
    return
  }

  models.DB.Model(&book).Updates(input)

  c.JSON(http.StatusOK, gin.H{"data": book})
}

```


First, we can copy the code from the `FindBook` controller to grab a single book and make sure it exists. After we find the book, we need to validate the user input with the `UpdateBookInput` schema. Finally, we update the book model using the `Updates` method and return the updated book data to the client.

Register it into your `main.go`:

```
func main() {
  // ...

  r.GET("/books", controllers.FindBooks)
  r.POST("/books", controllers.CreateBook)
  r.GET("/books/:id", controllers.FindBook)
  r.PATCH("/books/:id", controllers.UpdateBook) // new

  r.Run()
}

```


Let’s test it out! Fire a `PATCH` request to the `/books/:id` endpoint to update the book title:

```
{
  "title": "The Infinite Game"
}

```


The result should be as follows:

```
{
  "data": {
    "id": 1,
    "title": "The Infinite Game",
    "author": "Simon Sinek"
  }
}

```


### The `Delete` handler function

The last step is to implement to `DeleteBook` feature:

```
// DELETE /books/:id
// Delete a book
func DeleteBook(c *gin.Context) {
  // Get model if exist
  var book models.Book
  if err := models.DB.Where("id = ?", c.Param("id")).First(&book).Error; err != nil {
    c.JSON(http.StatusBadRequest, gin.H{"error": "Record not found!"})
    return
  }

  models.DB.Delete(&book)

  c.JSON(http.StatusOK, gin.H{"data": true})
}

```


Just like the update controller, we get the book model from the request parameters if it exists and delete it with the `Delete` method from our database instance, which we get from our middleware. Then, return `true` as the result since there is no reason to return a deleted book data back to the client:

```
func main() {
        // ...
        r := gin.Default()
        models.ConnectDatabase()

        r.GET("/books", controllers.FindBooks)
        r.POST("/books", controllers.CreateBook)
        r.GET("/books/:id", controllers.FindBook)
        r.PATCH("/books/:id", controllers.UpdateBook)
        r.DELETE("/books/:id", controllers.DeleteBook) // new

        err := r.Run()
        if err != nil {
                return
        }
}

```


Let’s test it out by sending a `DELETE` request to the `/books/1` endpoint:

```
{
  "data": true
}

```


If we fetch all books in `/books`, we’ll see an empty array again:

```
{
  "data": []
}

```


Conclusion
----------

Go offers two major qualities that all developers desire and all programming languages aim to achieve: simplicity and performance. While this technology may not be the best option for every developer team, it’s still a very solid solution and a skill worth learning.

By building this project from scratch, I hope you gained a basic understanding of how to develop a RESTful API with Gin and Gorm, how they work together, and how to implement the CRUD features. There is still plenty of room for improvement, such as authenticating users with JWT, implementing unit testing, containerizing your app with Docker, and a lot of other cool stuff you can experiment with if you want to dig deeper.
