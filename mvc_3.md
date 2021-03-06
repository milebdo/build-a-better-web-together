# Quick MVC Tutorial #3

Nothing stops you from using your favorite **folder structure**. Iris is a low level web framework, it has got MVC first-class support but it doesn't limit your folder structure, this is your choice.

Structuring depends on your own needs. We can't tell you how to design your own application for sure but you're free to take a closer look to one of use-case examples below;

[![folder structure example](https://github.com/kataras/iris/raw/master/_examples/mvc/overview/folder_structure.png)](https://github.com/kataras/iris/raw/master/_examples/mvc/overview/folder_structure.png)

Shhh, let's spread the code itself.

#### Data Model Layer

```go
// file: datamodels/movie.go

package datamodels

// Movie is our sample data structure.
// Keep note that the tags for public-use (for our web app)
// should be kept in other file like "web/viewmodels/movie.go"
// which could wrap by embedding the datamodels.Movie or
// declare new fields instead butwe will use this datamodel
// as the only one Movie model in our application,
// for the shake of simplicty.
type Movie struct {
    ID     int64  `json:"id"`
    Name   string `json:"name"`
    Year   int    `json:"year"`
    Genre  string `json:"genre"`
    Poster string `json:"poster"`
}
```

#### Data Source / Data Store Layer

```go
// file: datasource/movies.go

package datasource

import "github.com/kataras/iris/_examples/mvc/overview/datamodels"

// Movies is our imaginary data source.
var Movies = map[int64]datamodels.Movie{
    1: {
        ID:     1,
        Name:   "Casablanca",
        Year:   1942,
        Genre:  "Romance",
        Poster: "https://iris-go.com/images/examples/mvc-movies/1.jpg",
    },
    2: {
        ID:     2,
        Name:   "Gone with the Wind",
        Year:   1939,
        Genre:  "Romance",
        Poster: "https://iris-go.com/images/examples/mvc-movies/2.jpg",
    },
    3: {
        ID:     3,
        Name:   "Citizen Kane",
        Year:   1941,
        Genre:  "Mystery",
        Poster: "https://iris-go.com/images/examples/mvc-movies/3.jpg",
    },
    4: {
        ID:     4,
        Name:   "The Wizard of Oz",
        Year:   1939,
        Genre:  "Fantasy",
        Poster: "https://iris-go.com/images/examples/mvc-movies/4.jpg",
    },
    5: {
        ID:     5,
        Name:   "North by Northwest",
        Year:   1959,
        Genre:  "Thriller",
        Poster: "https://iris-go.com/images/examples/mvc-movies/5.jpg",
    },
}
```

#### Repositories

The layer which has direct access to the "datasource" and can manipulate data directly.

```go
// file: repositories/movie_repository.go

package repositories

import (
    "errors"
    "sync"

    "github.com/kataras/iris/_examples/mvc/overview/datamodels"
)

// Query represents the visitor and action queries.
type Query func(datamodels.Movie) bool

// MovieRepository handles the basic operations of a movie entity/model.
// It's an interface in order to be testable, i.e a memory movie repository or
// a connected to an sql database.
type MovieRepository interface {
    Exec(query Query, action Query, limit int, mode int) (ok bool)

    Select(query Query) (movie datamodels.Movie, found bool)
    SelectMany(query Query, limit int) (results []datamodels.Movie)

    InsertOrUpdate(movie datamodels.Movie) (updatedMovie datamodels.Movie, err error)
    Delete(query Query, limit int) (deleted bool)
}

// NewMovieRepository returns a new movie memory-based repository,
// the one and only repository type in our example.
func NewMovieRepository(source map[int64]datamodels.Movie) MovieRepository {
    return &movieMemoryRepository{source: source}
}

// movieMemoryRepository is a "MovieRepository"
// which manages the movies using the memory data source (map).
type movieMemoryRepository struct {
    source map[int64]datamodels.Movie
    mu     sync.RWMutex
}

const (
    // ReadOnlyMode will RLock(read) the data .
    ReadOnlyMode = iota
    // ReadWriteMode will Lock(read/write) the data.
    ReadWriteMode
)

func (r *movieMemoryRepository) Exec(query Query, action Query, actionLimit int, mode int) (ok bool) {
    loops := 0

    if mode == ReadOnlyMode {
        r.mu.RLock()
        defer r.mu.RUnlock()
    } else {
        r.mu.Lock()
        defer r.mu.Unlock()
    }

    for _, movie := range r.source {
        ok = query(movie)
        if ok {
            if action(movie) {
                if actionLimit >= loops {
                    break // break
                }
            }
        }
    }

    return
}

// Select receives a query function
// which is fired for every single movie model inside
// our imaginary data source.
// When that function returns true then it stops the iteration.
//
// It returns the query's return last known "found" value
// and the last known movie model
// to help callers to reduce the LOC.
//
// It's actually a simple but very clever prototype function
// I'm using everywhere since I firstly think of it,
// hope you'll find it very useful as well.
func (r *movieMemoryRepository) Select(query Query) (movie datamodels.Movie, found bool) {
    found = r.Exec(query, func(m datamodels.Movie) bool {
        movie = m
        return true
    }, 1, ReadOnlyMode)

    // set an empty datamodels.Movie if not found at all.
    if !found {
        movie = datamodels.Movie{}
    }

    return
}

// SelectMany same as Select but returns one or more datamodels.Movie as a slice.
// If limit <=0 then it returns everything.
func (r *movieMemoryRepository) SelectMany(query Query, limit int) (results []datamodels.Movie) {
    r.Exec(query, func(m datamodels.Movie) bool {
        results = append(results, m)
        return true
    }, limit, ReadOnlyMode)

    return
}

// InsertOrUpdate adds or updates a movie to the (memory) storage.
//
// Returns the new movie and an error if any.
func (r *movieMemoryRepository) InsertOrUpdate(movie datamodels.Movie) (datamodels.Movie, error) {
    id := movie.ID

    if id == 0 { // Create new action
        var lastID int64
        // find the biggest ID in order to not have duplications
        // in productions apps you can use a third-party
        // library to generate a UUID as string.
        r.mu.RLock()
        for _, item := range r.source {
            if item.ID > lastID {
                lastID = item.ID
            }
        }
        r.mu.RUnlock()

        id = lastID + 1
        movie.ID = id

        // map-specific thing
        r.mu.Lock()
        r.source[id] = movie
        r.mu.Unlock()

        return movie, nil
    }

    // Update action based on the movie.ID,
    // here we will allow updating the poster and genre if not empty.
    // Alternatively we could do pure replace instead:
    // r.source[id] = movie
    // and comment the code below;
    current, exists := r.Select(func(m datamodels.Movie) bool {
        return m.ID == id
    })

    if !exists { // ID is not a real one, return an error.
        return datamodels.Movie{}, errors.New("failed to update a nonexistent movie")
    }

    // or comment these and r.source[id] = m for pure replace
    if movie.Poster != "" {
        current.Poster = movie.Poster
    }

    if movie.Genre != "" {
        current.Genre = movie.Genre
    }

    // map-specific thing
    r.mu.Lock()
    r.source[id] = current
    r.mu.Unlock()

    return movie, nil
}

func (r *movieMemoryRepository) Delete(query Query, limit int) bool {
    return r.Exec(query, func(m datamodels.Movie) bool {
        delete(r.source, m.ID)
        return true
    }, limit, ReadWriteMode)
}
```

#### Services

The layer which has access to call functions from the "repositories" and "models" (or even "datamodels" if simple application). It should contain the most of the domain logic.

```go
// file: services/movie_service.go

package services

import (
    "github.com/kataras/iris/_examples/mvc/overview/datamodels"
    "github.com/kataras/iris/_examples/mvc/overview/repositories"
)

// MovieService handles some of the CRUID operations of the movie datamodel.
// It depends on a movie repository for its actions.
// It's here to decouple the data source from the higher level compoments.
// As a result a different repository type can be used with the same logic without any aditional changes.
// It's an interface and it's used as interface everywhere
// because we may need to change or try an experimental different domain logic at the future.
type MovieService interface {
    GetAll() []datamodels.Movie
    GetByID(id int64) (datamodels.Movie, bool)
    DeleteByID(id int64) bool
    UpdatePosterAndGenreByID(id int64, poster string, genre string) (datamodels.Movie, error)
}

// NewMovieService returns the default movie service.
func NewMovieService(repo repositories.MovieRepository) MovieService {
    return &movieService{
        repo: repo,
    }
}

type movieService struct {
    repo repositories.MovieRepository
}

// GetAll returns all movies.
func (s *movieService) GetAll() []datamodels.Movie {
    return s.repo.SelectMany(func(_ datamodels.Movie) bool {
        return true
    }, -1)
}

// GetByID returns a movie based on its id.
func (s *movieService) GetByID(id int64) (datamodels.Movie, bool) {
    return s.repo.Select(func(m datamodels.Movie) bool {
        return m.ID == id
    })
}

// UpdatePosterAndGenreByID updates a movie's poster and genre.
func (s *movieService) UpdatePosterAndGenreByID(id int64, poster string, genre string) (datamodels.Movie, error) {
    // update the movie and return it.
    return s.repo.InsertOrUpdate(datamodels.Movie{
        ID:     id,
        Poster: poster,
        Genre:  genre,
    })
}

// DeleteByID deletes a movie by its id.
//
// Returns true if deleted otherwise false.
func (s *movieService) DeleteByID(id int64) bool {
    return s.repo.Delete(func(m datamodels.Movie) bool {
        return m.ID == id
    }, 1)
}
```

#### View Models

There should be the view models, the structure that the client will be able to see.

Example:

```go
import (
    "github.com/kataras/iris/_examples/mvc/overview/datamodels"

    "github.com/kataras/iris/context"
)

type Movie struct {
    datamodels.Movie
}

func (m Movie) IsValid() bool {
    /* do some checks and return true if it's valid... */
    return m.ID > 0
}
```

Iris is able to convert any custom data Structure into an HTTP Response Dispatcher,
so theoritically, something like the following is permitted if it's really necessary;

```go
// Dispatch completes the `kataras/iris/mvc#Result` interface.
// Sends a `Movie` as a controlled http response.
// If its ID is zero or less then it returns a 404 not found error
// else it returns its json representation,
// (just like the controller's functions do for custom types by default).
//
// Don't overdo it, the application's logic should not be here.
// It's just one more step of validation before the response,
// simple checks can be added here.
//
// It's just a showcase,
// imagine the potentials this feature gives when designing a bigger application.
//
// This is called where the return value from a controller's method functions
// is type of `Movie`.
// For example the `controllers/movie_controller.go#GetBy`.
func (m Movie) Dispatch(ctx context.Context) {
    if !m.IsValid() {
        ctx.NotFound()
        return
    }
    ctx.JSON(m, context.JSON{Indent: " "})
}
```

However, we will use the "datamodels" as the only one models package because
Movie structure doesn't contain any sensitive data, clients are able to see all of its fields
and we don't need any extra functionality or validation inside it.

#### Controllers

Handles web requests, bridge between the services and the client.

```go
// file: web/controllers/movie_controller.go

package controllers

import (
    "errors"

    "github.com/kataras/iris/_examples/mvc/overview/datamodels"
    "github.com/kataras/iris/_examples/mvc/overview/services"

    "github.com/kataras/iris"
    "github.com/kataras/iris/mvc"
)

// MovieController is our /movies controller.
type MovieController struct {
    // mvc.C is just a lightweight lightweight alternative
    // to the "mvc.Controller" controller type,
    // use it when you don't need mvc.Controller's fields
    // (you don't need those fields when you return values from the method functions).
    mvc.C

    // Our MovieService, it's an interface which
    // is binded from the main application.
    Service services.MovieService
}

// Get returns list of the movies.
// Demo:
// curl -i http://localhost:8080/movies
//
// The correct way if you have sensitive data:
// func (c *MovieController) Get() (results []viewmodels.Movie) {
//  data := c.Service.GetAll()
//
//  for _, movie := range data {
// 	  results = append(results, viewmodels.Movie{movie})
//  }
//  return
// }
// otherwise just return the datamodels.
func (c *MovieController) Get() (results []datamodels.Movie) {
    return c.Service.GetAll()
}

// GetBy returns a movie.
// Demo:
// curl -i http://localhost:8080/movies/1
func (c *MovieController) GetBy(id int64) (movie datamodels.Movie, found bool) {
    return c.Service.GetByID(id) // it will throw 404 if not found.
}

// PutBy updates a movie.
// Demo:
// curl -i -X PUT -F "genre=Thriller" -F "poster=@/Users/kataras/Downloads/out.gif" http://localhost:8080/movies/1
func (c *MovieController) PutBy(id int64) (datamodels.Movie, error) {
    // get the request data for poster and genre
    file, info, err := c.Ctx.FormFile("poster")
    if err != nil {
        return datamodels.Movie{}, errors.New("failed due form file 'poster' missing")
    }
    // we don't need the file so close it now.
    file.Close()

    // imagine that is the url of the uploaded file...
    poster := info.Filename
    genre := c.Ctx.FormValue("genre")

    return c.Service.UpdatePosterAndGenreByID(id, poster, genre)
}

// DeleteBy deletes a movie.
// Demo:
// curl -i -X DELETE -u admin:password http://localhost:8080/movies/1
func (c *MovieController) DeleteBy(id int64) interface{} {
    wasDel := c.Service.DeleteByID(id)
    if wasDel {
        // return the deleted movie's ID
        return iris.Map{"deleted": id}
    }
    // right here we can see that a method function can return any of those two types(map or int),
    // we don't have to specify the return type to a specific type.
    return iris.StatusBadRequest
}
```

```go
// file: web/controllers/hello_controller.go

package controllers

import (
    "errors"

    "github.com/kataras/iris/mvc"
)

// HelloController is our sample controller
// it handles GET: /hello and GET: /hello/{name}
type HelloController struct {
    mvc.C
}

var helloView = mvc.View{
    Name: "hello/index.html",
    Data: map[string]interface{}{
        "Title":     "Hello Page",
        "MyMessage": "Welcome to my awesome website",
    },
}

// Get will return a predefined view with bind data.
//
// `mvc.Result` is just an interface with a `Dispatch` function.
// `mvc.Response` and `mvc.View` are the built'n result type dispatchers
// you can even create custom response dispatchers by
// implementing the `github.com/kataras/iris/mvc#Result` interface.
func (c *HelloController) Get() mvc.Result {
    return helloView
}

// you can define a standard error in order to be re-usable anywhere in your app.
var errBadName = errors.New("bad name")

// you can just return it as error or even better
// wrap this error with an mvc.Response to make it an mvc.Result compatible type.
var badName = mvc.Response{Err: errBadName, Code: 400}

// GetBy returns a "Hello {name}" response.
// Demos:
// curl -i http://localhost:8080/hello/iris
// curl -i http://localhost:8080/hello/anything
func (c *HelloController) GetBy(name string) mvc.Result {
    if name != "iris" {
        return badName
        // or
        // GetBy(name string) (mvc.Result, error) {
        //  return nil, errBadName
        // }
    }

    // return mvc.Response{Text: "Hello " + name} OR:
    return mvc.View{
        Name: "hello/name.html",
        Data: name,
    }
}
```

```go
// file: web/middleware/basicauth.go

package middleware

import "github.com/kataras/iris/middleware/basicauth"

// BasicAuth middleware sample.
var BasicAuth = basicauth.New(basicauth.Config{
    Users: map[string]string{
        "admin": "password",
    },
})
```

```html
<!-- file: web/views/hello/index.html -->
<html>

<head>
    <title>{{.Title}} - My App</title>
</head>

<body>
    <p>{{.MyMessage}}</p>
</body>

</html>
```

```html
<!-- file: web/views/hello/name.html -->
<html>

<head>
    <title>{{.}}' Portfolio - My App</title>
</head>

<body>
    <h1>Hello {{.}}</h1>
</body>

</html>
```

> Navigate to the [kataras/iris/_examples/view](https://github.com/kataras/iris/tree/master/_examples/#view) for more examples
like shared layouts, tmpl funcs, reverse routing and more!

#### Main

This file creates any necessary component and links them together.

```go
// file: main.go

package main

import (
    "github.com/kataras/iris/_examples/mvc/overview/datasource"
    "github.com/kataras/iris/_examples/mvc/overview/repositories"
    "github.com/kataras/iris/_examples/mvc/overview/services"
    "github.com/kataras/iris/_examples/mvc/overview/web/controllers"
    "github.com/kataras/iris/_examples/mvc/overview/web/middleware"

    "github.com/kataras/iris"
)

func main() {
    app := iris.New()

    // Load the template files.
    app.RegisterView(iris.HTML("./web/views", ".html"))

    // Register our controllers.
    app.Controller("/hello", new(controllers.HelloController))

    // Create our movie repository with some (memory) data from the datasource.
    repo := repositories.NewMovieRepository(datasource.Movies)
    // Create our movie service, we will bind it to the movie controller.
    movieService := services.NewMovieService(repo)

    app.Controller("/movies", new(controllers.MovieController),
        // Bind the "movieService" to the MovieController's Service (interface) field.
        movieService,
        // Add the basic authentication(admin:password) middleware
        // for the /movies based requests.
        middleware.BasicAuth)

    // Start the web server at localhost:8080
    // http://localhost:8080/hello
    // http://localhost:8080/hello/iris
    // http://localhost:8080/movies
    // http://localhost:8080/movies/1
    app.Run(
        iris.Addr("localhost:8080"),
        iris.WithoutVersionChecker,
        iris.WithoutServerError(iris.ErrServerClosed),
        iris.WithOptimizations, // enables faster json serialization and more
    )
}
```

----

More folder structure guidelines can be found at the [https://github.com/kataras/iris/tree/master/_examples/#structuring](https://github.com/kataras/iris/tree/master/_examples/#structuring) section.

Follow the examples below

- [Session Controller](https://github.com/kataras/iris/blob/master/_examples/mvc/session-controller/main.go)
- [A simple but featured Controller with model and views](https://github.com/kataras/iris/tree/master/_examples/mvc/controller-with-model-and-view).
- [Login showcase](https://github.com/kataras/iris/tree/master/mvc/login) **NEW**