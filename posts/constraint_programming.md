### Constraint Programming

Yes! We as developers have always been programming with constraints like deadlines, changing user requirements, internet downloading build systems, FP vs OOP dilemma, to use or not to use the new shiny web framework of the week, half assed tech bros, slow compiling languages, meetings etc. But this post is not about developer constraints. 

**This is about programs which run based on specific constraints and try to find out solutions 
for us based on the constraints i.e. rules. It is a different [programming paradigm](https://en.wikipedia.org/wiki/Constraint_programming).**

Programming is always about finding solutions at various degrees of scale. A software platform finds solution to an existing business problem as a whole. Components of the software system finds solutions to parts of the business problem. Individual class/code running within those components find solutions to their respective problems at a granular level. If your problem composes then the solution also composes. 

Some (or most) problems might be as good as finding applicable values for variables based on a set of rules. Constraint programming enables us to specify these rules as models and allow the system to find appropriate values for the variables. 

_Wannabe mathematician impostor alert below!!!_

Suppose there are two variables `V1, V2, ..., Vn` which can have possible values from domain `D1, D2,..., Dn` respectively and the relationships between those variables are specified as a set of rules aka Constraints `C1, C2, ..., Cm` where `m > 0 and m = n or m <> n` then a program `P` which finds the appropriate values of `V1, V2, ..., Vn` from respective domains which satisfy the constraints is called a constraint programming system. At the end os successful execution we will have `V1=d1, V2=d2, ..., Vn=dn` where `d1, d2,..., dn` are specific values from domains `D1, D2,..., Dn` respectively.

Now the variables can be of type simple integers to arrays/sets or matrices. Each domain will obviously be of the same data type as the variable. 

#### Current state and library choices
Our industry is driven by libraries created by computer science literates turned developers and normal developers (i don't want to talk about CS literacy here :-P) who understand (rarely ;-)) and use those libraries to create wonderful products. Similar to any CS concept there are a good number of open source constraint programming libraries available in the market. We initially evaluate multiple choices and landed up on [Google Optimisation tools](https://developers.google.com/optimization/), of course standing in the shoulder of a giant is safe isn't it?

Though the library has multiple awesome functionality and necessary wrappers in java, python and .net we tried using the java wrappers as the customer was a java shop. Though the documentation was vague and we have to scan through reference docs we were able to write some programs and spike it out. Even though the wrappers were in java the core has been written in C++ and any exception stack traces ended up us reading C++ code to figure out the problem. This was confusing with minimal help from documentation (we did this 1.5 years ago, may be the docs are good now). 

So after facing some road blocks with Google's implementation I set out to find alternative library choices and ended up with [Choco Solver](http://www.choco-solver.org/). This is sweet (pun intended), pure java library for Constraint Programming. The DSL they have used was easy, stack traces were traceable to Java code and was realtively easier for us to use. The pleasure of adding it just like Google Guava or another utility in our build gradle file was good. The creators are also pretty responsive in support forums for queries. 

#### A Simple example or how to draw an owl or Hello World!

Let us consider our simple school math equation problem.

```
Given a + b = 8 and a - b = 4 find the value of a and b
```

Our typical approach to solve this problem using a paper and pen would be:
```
1. a = 4 + b
2. a + b = 8
3. (4 + b) + b = 8
4. 4 + 2b = 8
5. 2b = 8 - 4
6. 2b = 4 
7. b = 4 / 2
8. b = 2
9. a + b = 8
10. a + 2 = 8
11. a = 8 - 2 
12. a = 6
13. a = 6, b = 2
```

##### Choco Solver basics
In Choco solver (v4) the whole problem is modelled as a `Model` object which contains all the variables, constraints along with applicable value domains for variables. So first we have to create a model.

```java
Model model = new Model("Simple Problem");
```

Now after having created a model we need to start defining the variables for which we need the values.

```java
        IntVar a = model.intVar("a", 1, 8);
        IntVar b = model.intVar("b", 1, 8);
```
a is an integer variable with name "a" (of course!) and applicable values ranging from 1 to 8 which is the applicable domain of this variable. 

Since our constraint is `a + b = 8 and a - b = 4` a or b cannot be greater than 8 logically we restrict the domain from 1 to 8 for simplicity purposes. 

Next step will be to specify the constraints. Choco solver has a nice DSLesque way to specify simple arithmetic constraints.

```java
        //a + b = 8
        model.arithm(a, "+", b, "=", 8).post();
        //a - b = 4
        model.arithm(a, "-", b, "=", 4).post();
```

> Note: Always remeber to call .post() after declaring each constraint else choco solver will forget it

Now each model will have a corresponding `Solver` implementation.  Solver has a `solve()` method which will return us `true/false` depending on whether a solution is found or not.  If the solution is found then the variables i.e. `IntVar` we declared earlier will have the values corresponding to the current solution.

To extract all possible solutions we can use code below:

```java
package com.ajira.chocosolver;

import org.chocosolver.solver.Model;
import org.chocosolver.solver.variables.IntVar;

public class SimpleConstraint {
    public static void main(String[] args) {
        Model model = new Model("Simple Constraint");

        IntVar a = model.intVar("a", 1, 8);
        IntVar b = model.intVar("b", 1, 8);

        //a + b = 8
        model.arithm(a, "+", b, "=", 8).post();
        //a - b = 4
        model.arithm(a, "-", b, "=", 4).post();

        int i = 1;

        while (model.getSolver().solve()) {
            System.out.println("Solution " + (i++) + " found : " + a.getValue() + ", " + b.getValue());
        }

    }
}
```

Notice the `a.getValue()` calls to extract the value of a variable for the current solution. This is a simple problem with a very small search space i.e. 2 variables with 8 applicable values each. Real life problems are more complex and can have millions of combinations to search for. That is were an optimisation problem kicks in and specific solvers take care of the finding the best solutions. Wish i knew this in school for those math equation assignments ;-)

Output of the above program would be:
```
Solution 1 found : 6, 2
```
Which is the same as our paper and pencil solution. We took some few seconds or a minute but the system responds immediately!

```java
// TODO - explain scaleView using 3a + 2b = 16 etc
```

Now that we have understood how to draw an Owl via simple steps (image on the left), let us draw an actual owl (image on the right)

![How to draw an owl](https://afinde-production.s3.amazonaws.com/uploads/8706a488-d0f4-41ba-bf76-7151762fd5d1.jpg)

Image credits: https://eurokeks.com/memes/how-to-draw-an-owl

### Drawing an actual Owl or an actual problem
So in our customer place we had a problem of scheduling a set of tasks for a set of employees. No I am not going to discuss the schedule generation part, even before scheduling we have to group these tasks into N minute sessions of fixed time so that it can be evenly distributed among employees. Since each task can take variable times with a high degree of variations like 1 minute task to 45 minute task choosing the best combination of tasks for grouping into sessions so that number of sessions are kept minimal and hence the breaks between each sessions is minimal was a challenge. There can be 1000s of tasks per day distributed across 100s of employees per day. Using naive approach or writing a customer algorithm does not work since the search space is huge. 

So in essence it is grouping n tasks `T1, T2,..., Tn` into m sessions `S1, S2, ..., Sm` where `m <= n` so that m is minimal. There is no restriction on the number of tasks a session can hold. The expected output for example is 

```
S1 = {T23, T28, T89}
S2 = {T1, T7}
S3 = {T90}
.
.
.
Sm = {Tn-1, Tn}
```
You can notice that each session here can be considered as bin inside which each task is an object of a specific volume i.e. time taken. So this is apparently a [bin packing optimisation problem](https://en.wikipedia.org/wiki/Bin_packing_problem) which is common in Constraing Programming domain. Thankfully our CS literates have exposed APIs in Choco Solver to find solutions for such bin packing problems. 

We will follow an approach similar to the simple equation problem, first let us define the required variables

```java
// TODO
// tasks generated and stored in DB from some other system
// Task.java
// Task to variable conversion
```

Next steps is to specify the constraints

```java
//TODO
```

Then specify and optimisation goal for the solver since this problem can have multiple number of possible solutions. We always need the best! ;-)

```java
//TODO
```

now just run the solver to get necessary output.

```java
//TODO
```
If you have come until this I guess you would have some questions in the mind, a mini WTFAQ given below:

_What if there is no possible solution at all?_

_What if the solver is running for eternity?_

_How to debug this?_

Even if you are not using Google's OR library the [documentation](https://developers.google.com/optimization/introduction/java) has some good information on the type of problems solved by Constraint Programming

