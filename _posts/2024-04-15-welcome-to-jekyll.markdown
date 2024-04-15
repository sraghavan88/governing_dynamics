---
layout: post
title:  "Parser Generators"
date:   2024-04-15 06:21:52 -0400
categories: Compilers
---
The first stage of the compilation process is the pasring and undestanding  the sytax of the languages

Programs that recognize languages are called parsers or syntax analyzers. A grammar is just a set of rules, each one expressing the structure of a phrase. The ANTLR tool and tools like yaac translates grammars to parsers which are ready made code to parse the defined laguage in the grammer. Grammars themselves follow the syntax of a language optimized for specifying other languages

{% highlight ruby %}
grammar ArrayInit;
init : '{' value (',' value)* '}' ;
value : init
      | INT
      ;
INT : [0-9]+ ;
WS : [ \t\n]+ -> skip ;

{% endhighlight %}
