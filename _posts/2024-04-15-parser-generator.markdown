---
layout: post
title:  "Parser Generators"
date:   2024-04-15 06:21:52 -0400
categories: Compilers
---
### Prologue
The first stage of the compilation process is the pasring and undestanding  the sytax of the languages. Parsers job is look at code and make sure it follows the approved syntax of the grammer and also manipulate code structure to add or extract information

### What are Parser Generators
Programs that recognize languages are called parsers or syntax analyzers. A grammar is just a set of rules, each one expressing the structure of a phrase. The ANTLR tool and tools like yaac translates grammars to parsers which are ready made code to parse the defined laguage in the grammar. Grammars themselves follow the syntax of a language optimized for specifying other languages

### An example of a grammar
The grammar below can be used parse an array of integers.

{% highlight ruby %}
grammar ArrayInit;
init : '{' value (',' value)* '}' ;
value : init
      | INT
      ;
INT : [0-9]+ ;
WS : [ \t\n]+ -> skip ;

{% endhighlight %}

### How to organize code

We can orgaize the code in following way for Antlr to generate parsers from the g4 file.

{% highlight bash %}
└── exampleAntlr
    ├── LICENSE
    ├── README.md
    ├── exampleAntlr.iml
    ├── pom.xml
    ├── src
    │   └── main
    │       ├── antlr4
    │       │   └── com
    │       │       └── srini
    │       │           └── antrl4
    │       │               ├── ArrayInit.g4
    │       ├── java
    │       │   └── com
    │       │       └── srini
    │       │           └── antrl4
    │       │               └── TestAntlr.java

{% endhighlight %}

### Run the generated parser 
A maven install can generate the lexer parser for the g4 grammar. 
The below code can take an array and parse it and form the tree 

{% highlight java %}
ANTLRInputStream input = new ANTLRInputStream("{1,2,3,4}");
ArrayInitLexer lexer = new ArrayInitLexer(input); 
CommonTokenStream tokens = new CommonTokenStream(lexer);
ArrayInitParser parser = new ArrayInitParser(tokens);
ParseTree tree = parser.init();
System.out.println(tree.toStringTree(parser));
{% endhighlight %}

{% highlight bash %}

(init { (value 1) , (value 2) , (value 3) , (value 4) })

{% endhighlight %}