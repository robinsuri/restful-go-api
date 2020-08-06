# Gorilla Mux Router

In the previous section we saw how we can create a simele REST endpoint with net/http. We saw there were some limitations. But In most cases when we don't need complex path matching, net/http works just fine. 

Lets see how gorilla mux addresses the issues we saw with net/http.

## Run the example

```bash
git checkout origin/gorilla-mux-03
```

If you are not already in the folder
```bash
cd api-with-gorilla-mux
```

```bash
go run main.go
```

```bash
curl localhost:7999
```

## Why gorilla mux

In benchmarks gorilla mux is probably one of the slower routers out there. 

The following if froma benchmark result with one parameter.

| lib/framework | Operations K | ns/op | B/op | allocs/op |
|:-------------:|:------------:|:-----:|:----:|:---------:|
|     Beego     |      442     |  2791 |  352 |     3     |
|      Chi      |     1000     |  1006 |  432 |     3     |
|      Echo     |     14662    |  81.9 |   0  |     0     |
|      Gin      |     16683    |  72.3 |   0  |     0     |
|  Gorilla Mux  |      434     |  2943 | 1280 |     10    |
|   HttpRouter  |     23988    |   50  |   0  |     0     |

HttpRouter is around 60x faster. So why are we starting with gorilla mux?

For one, your router will almost never be your bottleneck, in an application where you have file i/o or database operations your performance / speed will depend way more on those things than your router. Also gorilla mux is compliant with net/http. So if you have a project already written with net/http you will have a easier time converting to mux compared to some of the other libraries / frameworks that has a different implementation of Handler.

## Routing based on Http Method

With net/http we were sending all Http Verb request to the route and handling each type in the same function. With mux we can specify the http method for each our route.

```go
  r.HandleFunc("/user", s.getUser).Methods(http.MethodGet)
  r.HandleFunc("/user", s.updateUser).Methods(http.MethodPut)
```

We will also need to create corresoponding methods for each routes. and extract 

With these our api should behave exactly the same.

```bash
curl localhost:7999/user 
```

```bash
curl -X PUT -d '{"username":"mofi","email":"mofi@gmail.com","age":27}' localhost:7999/user
```

## Path Params

With net/http we saw a simple example of how we can do path params. We gorilla mux this becomes significantly easier (this is also where the libraries and framework differ in implementations). 

For this part we start with a more complex example. Lets imagine a application that is keeping track of courses, instructors and attendees to these courses.

We have HandlerFunc for returing all users, instructors and courses. As well as getting individual user, course or instructor. And thats the route we are interested in here.

```go
  r.HandleFunc("/users/{id}", getUserByID).Methods(http.MethodGet)
```

In this route we have a path param `{id}` which we will access in the `getUserByID` function. 

```go
  pathParams := mux.Vars(r)
```

Thats all we have to do to get access to all the path parameters in `map[string]string`. Although depending on the use case that might not be enough. For example in our use case we expect the param to be an int. Its simple to convert the data to the right format. 

```go
if val, ok := pathParams["id"]; ok {
  id, err = strconv.Atoi(val)
  if err != nil {
    w.WriteHeader(http.StatusBadRequest)
    w.Write([]byte(`{"error": "need a valid id"}`))
    return
  }
}
```

Sending the error message and setting the status code helps the consumer of this api to take appropriate actions.

Once we hace access to the id we can search our list for the appropirate resource. In our case its looping through an array. In most cases it would be querying some sort of database.

## Query Parameters

We can get access the the query map by calling `r.URL.Query()`. This returns a object of type `url.Value` which has the underlying type of `map[string][]string`. It is a map of array of string. Query parameters can be repeated thats why its an array.

We can also use `r.FormValue(key)` to get a query. 
>This method can get both query param and form value in request body.

For our example say we want to filter our all lists with some criterias like users with interests, instructors with expertise and courses with topics.

Lets look at `getAllUsers` function.

```go
  query := r.URL.Query()
```

With this we get access the the `map[string][]string`. Some applications mandate only one query per key. For that we can use `query.Get(key)` to get the first instance of the key in the query map. For our example we are allowing user to do multiple query of the same key. So we access the array.

```go
  interests, ok := query["interest"]
```

In go on map data access we get a second value which says if the value was there or not. If there is no query with the key we just return the whole result. In this example we are treating the queries as an `and` relation. If we query for `NodeJS` and `AI` all the users with interests in both of the topic will be returned. It is also possible to treat this like a or relation albeit not that common. 
