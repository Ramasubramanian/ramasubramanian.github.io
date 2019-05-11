### Build yourself a simple Domain Specific Language with S-Expressions

#### tl;dr
A reference project implementation for using [S-Expression](https://en.wikipedia.org/wiki/S-expression) as DSL with a scala backend

[Simple DSL on Github](https://github.com/Ramasubramanian/simple-dsl)

If you are still curious and want to read a lot then proceed...

#### Motivation

As developers our job is use to write code to solve business problems. General purpose programming languages like C++, Java, C#, Scala etc are our usual goto tools for achieving these solutions. As the name suggests these languages are general purpose hence too powerful with complex syntax and semantics for general business people to understand and hence they use us coders to get the dirty job done. In some cases the business users may want to drive system/solution behavior with something less complex than a programming language but more flexible than UI. 

In such cases mostly we resort to exposing a subset of programming language with simpler syntax and semantics to give them appropriate flexibility and control without giving them the ability to shoot themselves on the foot as we developers have been doing for ages. Since this subset of a PL will be more specific to the problem domain these kind of languages are called Domain Specific languages. 

Many times we write systems for people who are intermediate users for the end consumers e.g. business workflow engines will have implementors who understand the actual customer's business processes and configure appropriate workflows while the developers of the workflow engine will expose DSLs/UI based tools to configure workflows. In such a system the implementors are the intermediate users between the actual consumer of the workflows and developers of the workflow engine. 

There are two type of DSLs:

1. **External DSLs:** Syntax is different from the host language that executes the DSL, this requires writing a parser and lexer to convert the DSL to host language code that can execute behind the scenes
2. **Internal DSLs:** Use the facilities available within the host language to create a DSL, this does not require a parser/lexer and can run within the host language runtimes as usual. e.g. Gradle using Groovy, SBT using Scala

The prime motivation of this post is to illustrate/guide the reader on how to create a simple external DSL which can be exposed to the intermediate users depending on the problem domain.

#### Word of caution!!!

External DSLs are fun to build but with time they tend to become complex and we will end up having a poorly implemented runtime of a stupidly designed programming language. This [article](https://modeling-languages.com/internal-external-dsls/) says it better than anyone. We need to be sure to think through how the DSL we are writing will evolve and what would be the demands of users in the future and how good are they with restricted functionalities. 

> Also don't use JSON/YAML/XML etc for writing DSLs they suck big time! Try one and let me know how it goes...

#### Problem domain
For simplicity sake _(yes that's how i escape from writing a mini-book, avoiding problems and prevent from exposing myself!)_ let us take the problem domain of a scientific calculator. Let us expose a DSL which can be used to calculate mathematical equations/formula that operate on numbers. 
e.g. Take the arithmetic expression below:
```scala
100 + (1000 - (200 * (1/20)))
```

#### DSL syntax
Designing a good DSL is as hard as naming. There are multiple things to consider:
- How intuitive it will be to the user? i.e. Lesser cognitive load
- How to make the user type less?
- How simple is the parser and grammar to maintain?

If your DSL is going to have more forms and variations in syntax and semantics then the parser is going to get complex eventually and we will step into the area of computer scientists from computer programmers which will be hard for many of us. Based on my experiences with multiple programming languages and DSLs over a period of time I find [S-Expression](https://en.wikipedia.org/wiki/S-expression) having a good tradeoff between intuitive, less typing and maintaining a simple parser/grammar. So we will use S-Expresssions as the DSL for this post. The above example expression might transfer into 
```clojure
(+ 100 (- 1000 (* 200 (/ 1 20))))
```
I can hear you yelling `"How the F*** is this simple?"` please wait until I show you the simplicity when it comes to writing a parser and extending this DSL to other domains. 

#### Parser generators
Writing a parser is hard for mortals like us. So there are some good people in the world who have written parser generators that write parsers for us and all we need to do is write programmatic hooks that can handle the parsed tokens. Some of the commonly used parser generators are:
- [Lex and Yacc](http://dinosaur.compilertools.net/)
- [ANTLR](https://www.antlr.org/)
- [JavaCC](https://github.com/javacc/javacc)

All are great, we will use ANTLR for this post. ANTLR is used by well established frameworks like Hibernate to parse H(S)QL. Yes! SQL is one the best DSLs out there!

#### S-ex(y)pressions Grammar and ANTLR
Let us look at the grammar of S-Expressions. Every expression follows the uniform syntax of
```clojure
(operator operand1 operand2 ... operandN)
```
the operands can again be nested expressions like
```clojure
(operator1 operand1 (operator2 operand1 operand2...) operand3 ...)
```
There is a clear advantage of simplicity in following such a notation. For example take unary vs binary operators. In normal programming languages we will have + which is an arithmetic binary operator which adds two numbers and we will have a ++ which is an unary operator which operates on a single number
```java
a + b
a++
a++ + b
```
Both expressions have different syntax hence different grammars and rules to handle. If parser encounters a + then previous and next token are operands, if it encounters a ++ symbol then previous token is the operand and so on.
Consider the same in S-Expression
```clojure
(+ a b)
(++ a)
(+ (++ a) b)
```
In this case we have two operators + and ++ with 2 and 1 arguments respectively but the syntax and rules for parsing is the same. Whether it is ++ or + just treat the next token as the operand. Of course parentheses are there, but as I said it is a good trade off between having a simpler parser and people get used to those `)))`s with time. Our parser logic can be uniform in this case. Rules are simple.
- Anything that starts with `(` is an expression to be evaluated
- First token after `(` is the operand which is a symbol
- All subsequent tokens in the expression are operands
- `)` indicates end of an expression
- Operands can be recursive expressions upto any depth

##### ANTLR v4
We will use ANTLR version 4 for this post. ANTLR has the concept of GRAMMAR files which have their own syntax :-) to specify how to parse tokens of your language. So depending on the rules we saw earlier we can write a grammar file after learning the ANTLR specification syntax or being the lazy devs we are we can download the [S-Expression ANTLR grammar file](https://github.com/antlr/grammars-v4/blob/master/sexpression/sexpression.g4).

```
grammar sexpression;

sexpr
   : item* EOF
   ;

item
   : atom
   | list
   | LPAREN item DOT item RPAREN
   ;

list
   : LPAREN item* RPAREN
   ;

atom
   : STRING
   | SYMBOL
   | NUMBER
   | DOT
   ;


STRING
   : '"' ('\\' . | ~ ('\\' | '"'))* '"'
   ;


WHITESPACE
   : (' ' | '\n' | '\t' | '\r') + -> skip
   ;


NUMBER
   : ('+' | '-')? (DIGIT) + ('.' (DIGIT) +)?
   ;


SYMBOL
   : SYMBOL_START (SYMBOL_START | DIGIT)*
   ;


LPAREN
   : '('
   ;


RPAREN
   : ')'
   ;


DOT
   : '.'
   ;


fragment SYMBOL_START
   : ('a' .. 'z') | ('A' .. 'Z') | '+' | '-' | '*' | '/' | '.'
   ;


fragment DIGIT
   : ('0' .. '9')
   ;
```

I will try my best to explain what is done here.
- DIGIT: Can contain numbers from 0 - 9
- SYMBOL_START: can begin with alphabets and arithmetic operators, these are not string literals and are considered to be keywords in the DSL
- LPAREN, RPAREN: Left and Right parentheses which is the core of our DSL
- SYMBOL: Should start with alphabets or arithmetic operators preceded by any number of alphabest or arithmetic operators or digits. Note the * in the end
- WHITESPACE: Self explanatory, note how we have asked the parser to skip these
- STRING: For string literals which are not symbols aka keywords
- ATOM: Atoms are just constant values which can be string, numbers or symbols
- list: List is the expression which can start with a LPAREN and ends with a RPAREN and can have any number item in between
- item: Is the member of a list which can exist in between the parentheses and note that an item can be a list as well in addition to being an atom, this is where the recursive nature of list within lists is specified
- sexpr: A sexpression is a collection of items which ends with the End of File character represented by EOF

##### Generating the parser using ANTLR
Install ANTLR4 using the [instructions 1](https://www.antlr.org/download.html) or [instructions 2](https://github.com/antlr/antlr4/blob/master/doc/getting-started.md) or use homebrew

```bash
brew install antlr@4
```
Download the [grammar file for S-Expressions](https://github.com/antlr/grammars-v4/blob/master/sexpression/sexpression.g4).

In the downloaded folder run below command
```bash
antlr4 sexpression.g4
```
This will generate a list of files required for the parser. Default target is Java we an generate for multiple languages.

```bash
sexpression.interp
sexpression.tokens
sexpressionBaseListener.java
sexpressionLexer.interp
sexpressionLexer.java
sexpressionLexer.tokens
sexpressionListener.java
sexpressionParser.java
```
Those interested can go through all these files I will just touch upon the ones we will have to interact with.

Let us take a look at sexpressionBaseListener.java. This is the base class which we have to extend to build our [Abstract Syntax Tree (AST)](https://en.wikipedia.org/wiki/Abstract_syntax_tree)

We can see the class has empty methods for entry an exit of each node or token type we have defined in the grammar file.

```java
/**
 * This class provides an empty implementation of {@link sexpressionListener},
 * which can be extended to create a listener which only needs to handle a subset
 * of the available methods.
 */
public class sexpressionBaseListener implements sexpressionListener {
	@Override public void enterSexpr(sexpressionParser.SexprContext ctx) { }
	@Override public void exitSexpr(sexpressionParser.SexprContext ctx) { }
	@Override public void enterItem(sexpressionParser.ItemContext ctx) { }
	@Override public void exitItem(sexpressionParser.ItemContext ctx) { }
	@Override public void enterList(sexpressionParser.ListContext ctx) { }
	@Override public void exitList(sexpressionParser.ListContext ctx) { }
	@Override public void enterAtom(sexpressionParser.AtomContext ctx) { }
	@Override public void exitAtom(sexpressionParser.AtomContext ctx) { }

	@Override public void enterEveryRule(ParserRuleContext ctx) { }
	@Override public void exitEveryRule(ParserRuleContext ctx) { }
	@Override public void visitTerminal(TerminalNode node) { }
	@Override public void visitErrorNode(ErrorNode node) { }
}
```

So we need to have a handler context of some sort may be a stack of operators and operands and then handle each entry an exit to convert the operators into appropriate AST nodes and keep pushing and popping the stack.

There is a simpler alternative for this listener based logic using the famed [visitor pattern](https://sourcemaking.com/design_patterns/visitor). Since visitor takes care of handling the context it makes lives easier for us to handle the token to AST node conversion alone. 

ANTLR generates visitor classes as well. We can force visitor based parser generation using command below:

```bash
antlr4 -visitor sexpression.g4
```
This will generate additional visitor classes as shown below:
```
sexpressionVisitor.java
sexpressionBaseVisitor.java
```
Lets take a look at sexpressionBaseVisitor.java

```java
public class sexpressionBaseVisitor<T> extends AbstractParseTreeVisitor<T> implements sexpressionVisitor<T> {
	
	@Override public T visitSexpr(sexpressionParser.SexprContext ctx) { return visitChildren(ctx); }
	
	@Override public T visitItem(sexpressionParser.ItemContext ctx) { return visitChildren(ctx); }
	
	@Override public T visitList(sexpressionParser.ListContext ctx) { return visitChildren(ctx); }
	
	@Override public T visitAtom(sexpressionParser.AtomContext ctx) { return visitChildren(ctx); }
}
```

So now we are all good and can start implementing the visiting logic for each type of node.

#### Project setup
We will build the backend for this DSL using Scala 2.12

Download and install [SBT](https://www.scala-sbt.org/download.html)

Then run the below command:
```bash
sbt new scala/scala-seed.g8
```
This will create a simple scala project after providing relevant details.

Project will have below structure
```
.
├── README.md
├── build.sbt
├── project
│   ├── Dependencies.scala
│   └── build.properties
└── src
    ├── main
    │   └── scala
    │       └── example
    │           └── Hello.scala
    └── test
        └── scala
            └── example
                └── HelloSpec.scala
```

Let us cleanup the default code and create packages for our own use
```bash
rm -Rf src/main/scala/example
rm -Rf src/test/scala/example
mkdir -p src/main/scala/in/simpledsl
mkdir -p src/main/java/in/simpledsl
```
#### Integrating ANTLR parser with project
Create a folder named antlr4 in our project root and copy the sexpression.g4 file downloaded earlier

The inside the antlr4 folder execute below command to generate parser classes 
```bash
antlr4 -package in.simpledsl.parser -visitor *.g4
```

Add ANTLR dependencies to the build file. Edit the `Dependencies.scala`  to look like below:

```scala
import sbt._

object Dependencies {
  lazy val scalaTest = "org.scalatest" %% "scalatest" % "3.0.5"
  lazy val antlr4Runtime = "org.antlr" % "antlr4-runtime" % "4.7"
  lazy val stringTemplate = "org.antlr" % "stringtemplate" % "3.2"

  val projectDeps = Seq(antlr4Runtime, stringTemplate, scalaTest % Test)
}
```

Then add the projectDeps in build.sbt as show below:

```scala
import Dependencies._

ThisBuild / scalaVersion     := "2.12.8"
ThisBuild / version          := "0.1.0-SNAPSHOT"
ThisBuild / organization     := "in.simple-dsl"
ThisBuild / organizationName := "simple-dsl"

lazy val root = (project in file("."))
  .settings(
    name := "simple-dsl",
    libraryDependencies ++= projectDeps
  )
```

Then copy all the generated files into java folder
```bash
mkdir -p ../src/main/java/in/simpledsl/parser
cp *.* ../src/main/java/in/simpledsl/parser
```

Invoke below command from project root to build and ensure all dependencies are good
```bash
 sbt clean compile
```
#### Building the AST
Now that we know what type of expressions we are going to support let us start defining the AST nodes

SExpression is going to be the root of all our node types. So let us define a trait SExpression inside the package `in.simpledsl.ast`

```scala
package in.simpledsl.ast

trait SExpression {
  def eval(): BigDecimal
}
```
Since the output of an expression evaluation is always a number in our domain of a scientific calculator we are using BigDecimal as the return type here.

Now let us define the primitive expression which is a number in our domain.

A Number expression just wraps a number i.e. a `BigDecimal` within

```scala
case class NumExpr(value: BigDecimal) extends SExpression {
  override def eval(): BigDecimal = value
}
```

Now let us define a slightly complex expression that is addition

```scala
case class PlusExpr(first: SExpression, second: SExpression) extends SExpression {
  override def eval(): BigDecimal = first.eval() + second.eval()
}
```
> Note the recursive nature of the plus expression where it accepts two arguments which are again 2 different s-expressions

Let us proceed to implement the basic arithmetic operators now.

```scala
case class MinusExpr(first: SExpression, second: SExpression) extends SExpression {
  override def eval(): BigDecimal = first.eval() - second.eval()
}

case class MultiplyExpr(first: SExpression, second: SExpression) extends SExpression {
  override def eval(): BigDecimal = first.eval() * second.eval()
}

case class DivideExpr(first: SExpression, second: SExpression) extends SExpression {
  override def eval(): BigDecimal = first.eval() / second.eval()
}
```
#### Visitor implementation and execution
Now we have all the basic expressions done let us look at how to implement a custom visitor class to convert the parsed tokens to these SExpression nodes. 

```scala
package in.simpledsl

import in.simpledsl.ast.SExpression
import in.simpledsl.parser.sexpressionBaseVisitor

class SExpressionVisitor extends sexpressionBaseVisitor[SExpression] {

}
```
You can notice that we have specified SExpression as the type parameter for the base visitor class which implies the output of the parser is going to be of type SExpression which we can eval() and get the output for.

Now let us start overriding visitor methods, we will start from `visitAtom` that it is inner most leaf node.

```scala
  override def visitAtom(ctx: sexpressionParser.AtomContext): SExpression = {
    if(ctx.NUMBER() != null) {
      NumExpr(BigDecimal(ctx.NUMBER().getText))
    } else if(ctx.SYMBOL() != null) {
      val symbol = ctx.SYMBOL().getText
      symbol match {
        case "+" => PlusExpr(null, null)
        case "-" => MinusExpr(null, null)
        case "*" => MultiplyExpr(null, null)
        case "/" => DivideExpr(null, null)
      }
    } else if(ctx.STRING() != null) {
      NumExpr(BigDecimal(ctx.NUMBER().getText))
    } else null
  }
```
If it is a number or a string we convert the same to a simple `NumExpr` node, if the operator is a symbol then we return a dummy or blank object of the correspnding complex node from here. We will use this dummy/blank object when we visit a list. 

> Note: There are ways to avoid using this dummy object but I am not choosing to dicusss them to keep this simple

Now let us implement the next critical operation `visitList` 
>the `ctx: sexpressionParser.ListContext` is an array of parsed nodes from the input string excluding the left and right nodes.

```scala
  override def visitList(ctx: sexpressionParser.ListContext): SExpression = {
    val op = visit(ctx.item(0))
    op match {
      case numExpr: NumExpr => numExpr
      case _: PlusExpr => {
        val first = visit(ctx.item(1))
        val second = visit(ctx.item(2))
        PlusExpr(first, second)
      }
      case _: MinusExpr => {
        val first = visit(ctx.item(1))
        val second = visit(ctx.item(2))
        MinusExpr(first, second)
      }
      case _: MultiplyExpr => {
        val first = visit(ctx.item(1))
        val second = visit(ctx.item(2))
        MultiplyExpr(first, second)
      }
      case _: DivideExpr => {
        val first = visit(ctx.item(1))
        val second = visit(ctx.item(2))
        DivideExpr(first, second)
      }
    }
  }
```
The line `val op = visit(ctx.item(0))` will invoke the `visitAtom` method we implemented earlier on the first argument of the list expression returning a suitable blank instance of the corresponding SExpression trait implementation. Instead of switching on the string value we are switching on the actual SExpression implementation type to recursively visit the remaining elements as per the specification.

There is one more method that needs to be overridden in the visitor to make it work

```scala
override def aggregateResult(aggregate: SExpression, nextResult: SExpression): SExpression = Option(nextResult).getOrElse(aggregate)
```
Aggregates the results of visiting multiple children of a node, the default implementation returns result of the last child visited. In our case it will be null since LPAREN is the last visited child so we are overriding 
this to return the non-null value of the two in given order.

Now let us see this in action. Let us create a Application class to run this.

```scala
package in.simpledsl

import in.simpledsl.parser.{sexpressionLexer, sexpressionParser}
import org.antlr.v4.runtime.{ANTLRInputStream, CommonTokenStream}

object Application {

  def main(args: Array[String]): Unit = {
    val expressionString =
      "(+ 10 (- 200 2))"

    val charStream = new ANTLRInputStream(expressionString)
    val lexer = new sexpressionLexer(charStream)
    val tokens = new CommonTokenStream(lexer)
    val parser = new sexpressionParser(tokens)

    val visitor = new SExpressionVisitor()

    val expression = visitor.visit(parser.sexpr())
    println(s"Expression:: $expression")
    val result = expression.eval()
    println(s"Result:: $result")
  }
}
```
The above code prints:
```
Expression:: PlusExpr(NumExpr(10),MinusExpr(NumExpr(200),NumExpr(2)))
Result:: 208
```
Which is inline with our expectations, the expression can support any level of nesting now. 

Now let us take a case of volume of a sphere with radius 10 the formula is `4/3 * pi * (10 ^ 3)`
We can very well write this expression using the primitives we have as below:

```clojure
(* (/ 4 3) (* 3.141 (* (* 10 10) 10)))
```
Running our code Application class with expression gives below output:
```
Expression:: MultiplyExpr(DivideExpr(NumExpr(4),NumExpr(3)),MultiplyExpr(NumExpr(3.141),MultiplyExpr(MultiplyExpr(NumExpr(10),NumExpr(10)),NumExpr(10))))
Result:: 4187.999999999999999999999999999999
```
Which is well and good but a normal user who has to use this formula frequently would find it hard. So if we decide to support sphere volume as a first class construct e.g.
```clojure
(sphere-volume 10)
```
We can implement this using below changes:

Let us start with an expression
```scala
case class SphereVolumeExpr(radius: SExpression) extends SExpression {
  override def eval(): BigDecimal = (BigDecimal(4) / BigDecimal(3)) * BigDecimal(3.141) * radius.eval().pow(3)
}
```
Next we need to plug this expression implementation in the visitor

```scala
  override def visitAtom(ctx: sexpressionParser.AtomContext): SExpression = {
    if(ctx.NUMBER() != null) {
      NumExpr(BigDecimal(ctx.NUMBER().getText))
    } else if(ctx.SYMBOL() != null) {
      val symbol = ctx.SYMBOL().getText
      symbol match {
        case "+" => PlusExpr(null, null)
        case "-" => MinusExpr(null, null)
        case "*" => MultiplyExpr(null, null)
        case "/" => DivideExpr(null, null)
        case "sphere-volume" => SphereVolumeExpr(null)
        case x => throw new UnsupportedOperationException(s"Operator $x is not supported")
      }
    } else if(ctx.STRING() != null) {
      NumExpr(BigDecimal(ctx.NUMBER().getText))
    } else null
  }

override def visitList(ctx: sexpressionParser.ListContext): SExpression = {
    val op = visit(ctx.item(0))
    op match {
      case numExpr: NumExpr => numExpr
      case _: PlusExpr => {
        val first = visit(ctx.item(1))
        val second = visit(ctx.item(2))
        PlusExpr(first, second)
      }
      case _: MinusExpr => {
        val first = visit(ctx.item(1))
        val second = visit(ctx.item(2))
        MinusExpr(first, second)
      }
      case _: MultiplyExpr => {
        val first = visit(ctx.item(1))
        val second = visit(ctx.item(2))
        MultiplyExpr(first, second)
      }
      case _: DivideExpr => {
        val first = visit(ctx.item(1))
        val second = visit(ctx.item(2))
        DivideExpr(first, second)
      }
      case _: SphereVolumeExpr => {
        val radius = visit(ctx.item(1))
        SphereVolumeExpr(radius)
      }
    }
  }  
```

Now testing our Application class with `(sphere-volume 10)`
prints below

```
Expression:: SphereVolumeExpr(NumExpr(10))
Result:: 4187.999999999999999999999999999999
```
Testing with `(sphere-volume (* (+ 3 2) 2))` also gives same results. So we have implemented a S-Expression parser and executor now! Note that `sphere-volume` is a unary operator and `+/-/*` etc are binary operators but we did not handle them differently and did not care because in S-Expression world everything is an operator/function with any number of arguments as appropriate. This unification IMHO is the advantage we get by choosing S-Expressions.

#### Conclusion
S-Expressions are a powerful and simple alternative compared to JSON/YAML/XML like languages used for configuration. The possibilities of using this are endless. We can build DSLs that can support do, let operators that can use variable assignments like below:
```clojure
(do
  (let "radius" (+ 5 (* 3 2)))
  (sphere-volume (ref "radius")))
```
**Big data pipelines**

Suppose if we are driving a big data pipeline running on hadoop or spark or something else which needs to read data from parquet files, select then transform few fields and write to postgres we can use something like below:

```clojure
(do
  (let "employee-transform"
    (transform
     (field "employeeId" (select "EmployeeID"))
     (field "employeeName"
            (concat (select "FirstName")
                    (select "LastName")))
     (field "dob" (parse-date "DateOfBirth"))))
  (run
   (source (parquet-file "/input/{CLIENT_ID}/datafiles" "employees.parquet"))
   (ref "employee-transform")
   (target (postgres "{JDBC_URL}" "employees_table"))))
```

**Workflow engines:**

Suppose we are designing a workflow engine with a workflow having series of tasks 
- Upload proposal
- Approve or reject
- Send reject email
- If approved send proposal to Salesforce CRM

```clojure
(do
  (task "upload-proposal" (upload-document "proposal.pdf"))
  (task "validate-proposal" (validate-document "{proposalId}"))
  (task "post-validate-action"
        (match "{validationResult}"
               (when "FAILED" (send-email "Your proposal has been rejected" "{customerEmail}"))
               (otherwise (run-task "upload-to-salesforce"))))
  (task "upload-to-salesforce" (salesforce-upload "{proposalId}"))
  (run
   (workflow "wf001"
             (steps "upload-proposal"
                    "validate-proposal"
                    "post-validate-action"))))
```
Each and every operation like `validation-document`/`salesforce-upload` can be complex implementations but hidden from business or intermediate users. Variables enclosed in `{}` can be resolved from a global context and so on. 

We can write DSLs that support 100s of operators and constructs using this example, refactoring and organising or improving this is left to the reader's imagination. 


> P.S If you are scared of maintaining a parser or feel this way of approaching DSL problem is an abomination then chill and simply use [Clojure](https://clojure.org/). You get S-Expressions on steroids, an internal DSL and more

```clojure
(keep-calm (and (use 'clojure)))
```

Code used in this post is available as project in [Github](https://github.com/Ramasubramanian/simple-dsl)

Thank you for making it till the end, for any corrections drop an email to r a m [d o t] 6 3 8 3 [a t] g m a i l [d o t] c o m