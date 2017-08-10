### Composing Functions in Java

I have been using [vavr.io](http://vavr.io) formerly Javaslang in my current project which is a REST service with considerable complexity like Schedule generation for a bunch of employees based on rules etc. I have made extensive use of Either, Try and Option monads in my code base. 

_P.S: Code written here is not exactly the same, have modified for simplification and confidentiality reasons_

I could create small or atomic functions and can chain them using these monads as shown below: 

```java
    return cleanupIncompleteAnalysis(analysisId)
            .flatMap(this::importAnalysisData)
            .flatMap(this::classify)
            .flatMap(this::generateSchedule);
```

The above method is a chain using `Either<Throwable, ?>` The right side of an Either will keep changing with the chain progression.

The code above does the following:
* For a given analysis id cleanup any incomplete data that might exist due to previous failures and fetch the analysis object
* Import fresh analytics data from an external data source
* Classify the imported data based on some algorithm
* Generate a schedule for employees based on classified data and a pre-defined set of rules

Now let us look at the signature of each of these methods:

```java 
    private Either<Throwable, Analysis> cleanupIncompleteAnalysis(Long analysisId) {
        ...
    }

    private Either<Throwable, List<Data>> importAnalysisData(Analysis analysis) {
        ...
    }

    private Either<Throwable, List<ClassifiedData>> classify(List<Data> inputData) {
        ...
    }

    private Either<Throwable, List<Session>> generateSessionsForDay0(List<ClassifiedData> classifiedData) {
        ...
    }
```

Since Either requires the left to be fixed the right part can change shape to `Analysis -> List<Data> -> List<ClassifiedData> -> List<Session>` in this case

For a brief period of time I was delighted, proud about [almost] functional code I have written, was brimming with happiness about how I made ugly Java code beautiful using monads! This is not my first stint with Either in Java, in my earlier projects I have used a handrolled version of my own `Either.java`

Then few new members entered the team and found it difficult to grasp what is happening here, the implementations of the above methods had further `Try.map().flatMap()` chains within them. It took time but they finally got a hold of it and were able to appreciate. Even now they have difficulty in following the code in some situations and ask for help. 

These interactions with new team members made me think about my choice of using functional constructs like Either, Try etc in the code base. So I thought may be I should take some time and introspect the alternatives and give a fair comparison. So what if I had not used Either and written this code using available Java constructs? Apart from allowing to chain methods Either monad also serves the purpose of breaking the chain if anything goes wrong. For e.g. in the above chain if there is an erorr with the `this::importAnalysisData` method call further calls to `classify` and `generateSessionsForDay0` would not happen. A natural equivalent of such behavior in Java is to use Exceptions. 

So let's try to change the methods with simple Java.

```java
    private Analysis cleanupIncompleteAnalysis(Long analysisId) {
        ...
    }

    private List<Data> importAnalysisData(Analysis analysis) {
        ...
    }

    private List<ClassifiedData> classify(List<Data> inputData) {
        ...
    }

    private List<Session> generateSessionsForDay0(List<ClassifiedData> classifiedData) {
        ...
    }
```     

To incroporate the "break the chain if something goes wrong" behavior we have to throw an unchecked exception from within these methods. So the implementation would change from:

**Earlier**
```java
    private Either<Throwable,List<Data>> importAnalysisData(Analysis analysis) {
       return Try.of(() -> {
           //code to fetch from source
           //...
       })
       .flatMap(x ->            
        if(error) {
               return Try.failure(new RuntimeException("Blah blah!"));
        } else {
            return Try.success(...);
        })
       .toEither();
    }
```
Some cases there will be just a simple Try.of() wrapping code that throws RuntimeExceptions similar to the one seen below as well.

**Now**

```java
    private List<Data> importAnalysisData(Analysis analysis) {
           //code to fetch from source
           //...
           if(error) {
               throw new RuntimeException("Blah blah!");
           } else {
               return data;
           }
    }
```

Assuming similar implementations for all the methods how do we chain them without `Either<?,?>` ?

```java 
    //NEW
    Analysis analysis = cleanupIncompleteAnalysis(analysisId);
    List<Data> data = importAnalysisData(analysis);
    List<ClassifiedData> classifiedData = classify(data);
    List<Session> sessions = generateSessionsForDay0(classifiedData);
    return sessions;

    //OLD
    return cleanupIncompleteAnalysis(analysisId)
            .flatMap(this::importAnalysisData)
            .flatMap(this::classify)
            .flatMap(this::generateSchedule);
```

In terms of elegance `flatMap` is the obvious winner, the code looks neat. If we are trying implement the same chain without flatMap on the NEW code

```java
    return generateSessionsForDay0(classify(importAnalysisData(cleanupIncompleteAnalysis(analysisId))));
```
:facepalm: moments :-/

Let us consider the readability, once you have grokked the concept of flatMap and monads the chain calls looks beautiful. But as newbie or someone used to typical Java programs using simple methods is more readable. And we also need not look into the individual methods for the return types since it is all written in black and white without `flatMap`

Let us consider this from the code volume perspective, the non `flatMap` way seems better with lesser lines of code but actually the method signatures have become simpler without the `Either<TYPE_1, TYPE_2<TYPE_3>>` kind of return types. Consider the method implementations. Eliminating `Try.of(...)` calls makes the code simpler to read but at the same time we have to handle the exceptions thrown somewhere else in the calling code if we are not using `Try`. With `Try` we can just treat the exceptions as just another return value similar to proper data which is a win. Treating exceptions as return values also enables us to cache these values i.e. memoize

Let us consider this from a performance perspective, `flatMap` way uses more lambdas (which are supposed to less performant than simple method calls), at the same time throwing exception might incur more cost than returning a value due to stacktrace and stuff. I don't have evidence for both the claims, perhaps some micro benchmarking can assert the same. 

Apart from `flatMap` other methods like `fold` have been immensely useful. In all the controllers the `Either` would be folded to an `ResponseEntity` for Spring MVC as show below which was easy to read and uniform. 

```java
    @RequestMapping(value = "/performanalysis", method = RequestMethod.GET)
    public @ResponseBody ResponseEntity performAnalysis(@RequestParam("analysis_id") String analysisId) {
        return Either.right(analysis)
                .flatMap(analysisService::performAnalysis)
                .fold(this::error, this::success);
    }
```
`error` and `success` are methods from superclass that can convert a throwable to HTTP 500/400 series and success values (any Object) to JSON HTTP 200 responses.  

I am not suggesting we **SHOULD/SHOULD NOT** use functional constructs in Java, this post is just an introspection of my choice to use the same and what if I hadn't used them. I am still divided on whether to use or not use these abstractions. Would like to conclude with below points.

* Using funtional constructs like Monads and flatMap makes the code more elegant and simple (subjective) 
* When using a PL just play to the power of that PL instead of arm twisting the language to do something which it is not meant to do. For e.g. if throwing exceptions are the natural way thena do it
* Using these functional abstractions would come with a performance cost as well as readability issues (subjective)
* Monads and flatMaps might be more suitable in ML dervied languages like Haskell, F# than Java

Thanks for reading till the end! 