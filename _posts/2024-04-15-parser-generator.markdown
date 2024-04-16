---
layout: post
title:  "Parser Generators an Introduction"
date:   2024-04-15 06:21:52 -0400
categories: Compilers
---
### Prologue
The first stage of the compilation process is the parsing and understanding of the syntax of a language. The Parsers job is to look at code and make sure it follows the approved syntax of the grammar and also manipulate code structure to add or extract information

### What are Parser Generators
Programs that recognize languages are called parsers or syntax analyzers. A grammar is just a set of rules, each one expressing the structure of a phrase. The ANTLR tool and tools like yaac translate grammar to parsers which are ready-made codes to parse the defined language in the grammar. Grammars themselves follow the syntax of a language optimized for specifying other languages

### An example of a grammar
The grammar below can be used to parse an array of integers.

{% highlight ruby %}
grammar ArrayInit;
init : '{' value (',' value)* '}' ; # define a array
value : init # define the value as an INT Type
      | INT
      ;
INT : [0-9]+ ; # define what is an INT
WS : [ \t\n]+ -> skip ; # Skip white spaces

{% endhighlight %}

### How to organize code

We can organize the code in the following way for Antlr to generate parsers from the g4 file. Keep the `g4` files in a `antlr4` directory and make sure the package structure matches between the `java` code and `antlr4` 

{% highlight bash %}
└── exampleAntlr
    ├── LICENSE
    ├── README.md
    ├── exampleAntlr.iml
    ├── pom.xml
    ├── src
    │   └── main
    │       ├── antlr4
    │       │   └── com
    │       │       └── srini
    │       │           └── antrl4
    │       │               ├── ArrayInit.g4
    │       ├── java
    │       │   └── com
    │       │       └── srini
    │       │           └── antrl4
    │       │               └── TestAntlr.java

{% endhighlight %}

### Run the generated parser 
A Maven install can generate the lexer parser for the g4 grammar. 
The below code can take an array like `{1,2,3,4}` and parse it and form the tree of its components. 

{% highlight java %}
ANTLRInputStream input = new ANTLRInputStream("{1,2,3,4}");
ArrayInitLexer lexer = new ArrayInitLexer(input); 
CommonTokenStream tokens = new CommonTokenStream(lexer);
ArrayInitParser parser = new ArrayInitParser(tokens);
ParseTree tree = parser.init();
System.out.println(tree.toStringTree(parser));
{% endhighlight %}

### The output of the parser 
{% highlight bash %}

(init { (value 1) , (value 2) , (value 3) , (value 4) })

{% endhighlight %}

So the parser has taken the array and broken it down to its components of array start token the integer values and the array end token.Now when we look at grammar syntax we can relate to it better.

 - `init {` refers to the start of an array represented in the grammar line no 1`init : '{' `
 - `value` is the Integer values of the array 

### A bad input

If we have input like `{1,2,3sadas,4}`  which has a bad integer in the array then the parser would generate an error 

{% highlight bash %}
line 1:6 token recognition error at: 's'
line 1:7 token recognition error at: 'a'
line 1:8 token recognition error at: 'd'
line 1:9 token recognition error at: 'a'
line 1:10 token recognition error at: 's'
{% endhighlight %}