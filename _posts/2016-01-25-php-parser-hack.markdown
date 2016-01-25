---
layout: post
title:  "The PHP Parser Hack"
date:   2016-01-25 21:13:02
categories: jekyll update
---

PHP's weird grammar enables it to do things that should be impossible at first
sight. In this article, we will take a look at a relatively arcane parser hack
that has existed since PHP 4.


# What is it?

The parser hack enables for arbitrary expressions to be used anywhere a
variable can be used. Let's take a look at it:
{% highlight php %}
<?php

${!${''} = 'str'};
${!${''} = (function(){return 'str';})()};
${!${''} = someFuncCall()};
{% endhighlight %}

It can use this in a few of places, such as:
{% highlight php %}
<?php

// with `new`
new ${!${''} = (new class() {
	function className() : string
	{
		return 'StdClass';
	}
})->className()}(); // new StdClass()

// normal property/method access
(new A)->${!${''} = (function(){return 'b';})()}(); // A->b()

// static property/method access
A::${!${''} = 'b'}(); // A::b()

// with `global`
global ${${!${''} = (function(){return 'str';})()}}; // global $str
{% endhighlight %}

Basically, it can be used to bypass restrictions where a variable is expected
but particular expressions cannot be used (like with `PDOStatement::bindParam`,
interpolation, etc).

The syntax looks rather strange, but the concept behind it is actually very
simply. Let's take a look at how it works.


# How does it work?

Let's take a quick look at the `simple_variable` production rule:
{% highlight c %}
simple_variable:
		T_VARIABLE		{ $$ = $1; }
	|	'$' '{' expr '}'	{ $$ = $3; }
	|	'$' simple_variable	{ $$ = zend_ast_create(ZEND_AST_VAR, $2); }
;
{% endhighlight %}

We can see that variable names can be made up of any arbitrary expression by
putting the expression inbetween curly braces and prepending a dollar sign to
it. The following is the `expr` production rule and (a trimmed version of) the
`expr_without_variable` production rule:
{% highlight c %}
expr:
		variable			{ $$ = $1; }
	|	expr_without_variable		{ $$ = $1; }
;

expr_without_variable:
		...
	|	variable '=' expr
			{ $$ = zend_ast_create(ZEND_AST_ASSIGN, $1, $3); }
	|	...
;
{% endhighlight %}

From the above, we can see that PHP's grammar allows for assignments as
variable names, since assignments themselves are expressions.

So how does this make the hack possible? Well, PHP will evaluate an expression
from inside-out, and so we can break down the following expression:
{% highlight php %}
<?php

${!${''} = 'str'};
```
To:
```PHP
${''} = 'str' // the nested assignment
!'str' // negate the returned 'str' value
${''} // the negated expression above becomes the variable name here
{% endhighlight %}

The variable in the nested assignment is evaluated first, and so this sets the
`${''}` variable to `'str'`. Next, the returned value from the assignment is
negated to produce `false` (which is equivilent to an empty string). This means
that the outer variable effectively access the freshly assigned `${''}`
variable, and simply returns its value.

With the above in mind, we could therefore rewrite the hack to:
{% highlight php %}
<?php

function funcCall() : string {return 'str';}

${!${false} = $a->b()};
${!!!${''} = 'str'};
${!!${true} = func()};
${${funcCall()} = 'str'};
${!${!${''}=++$str}}
// etc
{% endhighlight %}


# Conclusion

It is a rather perplexing, yet very simple trick for using arbitrary
expressions in places where variables are required. Given how confusing it
looks at first sight though, I would eschew it in code :)
