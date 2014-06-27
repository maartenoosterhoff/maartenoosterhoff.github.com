---
layout: post
title: A generic object creator using expression trees, part 1
description: "Creating objects using expression trees"
permalink: a-generic-object-creator-using-expression-trees-part-1
comments: true
tags:
- c#
- constructor
- expression tree
- generics
- object creator
- object factory
---

This blog series contains the following posts:

[1. Create an instance of an object using the constructor with no parameters](/a-generic-object-creator-using-expression-trees-part-1)
[2. Create an instance of an object using the constructor with parameters](/a-generic-object-creator-using-expression-trees-part-2)


One of the things I've bumped into in the past years is that using reflection to create an instance of an object is a solution that works, but somehow I do not like it. In the time of .NET 2.0 it was the only option, but with .NET 3.0 another option was made available: expression trees. I like it, but unfortunately haven't used it much. Therefore I went looking for information about expression trees. The [msdn site](http://msdn.microsoft.com/en-us/library/bb397951.aspx) provided me with plenty information.

Expression trees are described on msdn as follows: *Expression trees represent code in a tree-like data structure, where each node is an expression, for example, a method call or a binary operation such as x < y.*

Today I will show how to create an instance of an object using the constructor with no parameters, using expression trees.

Creating an object without any constructor parameters turned out not to be very difficult.

{% highlight csharp %}
using System.Linq.Expressions;
 
public static class ObjectCreator {
	public static TClass CreateObject<TClass>() where TClass : new() {
		Expression<Func<TClass>> creatorExpression =
			Expression.Lambda(
				Expression.New(typeof(TClass)),
				null
			) as Expression<Func<TClass>>;
		Func<TClass> creator = creatorExpression.Compile();
		if (creator != null) {
			return creator();
		}
		throw new Exception("Cannot create instance.");
	}
}
{% endhighlight %}

What is going on here? We have a generic method which creates an expression tree, compiles the expression tree into a lambda, and executes the lambda, and returns the result of the executed lambda.

The expression tree is the difficult part of the method. It is a [lambda expression](http://msdn.microsoft.com/en-us/library/bb336566.aspx) since we want it to become a lambda after compilation, and the lambda contains a [new expression](http://msdn.microsoft.com/en-us/library/system.linq.expressions.expression.new.aspx). The new expression has multiple overloads, and [one of them](http://msdn.microsoft.com/en-us/library/bb352804.aspx) takes a parameter of type Type. This is the overload for no constructor parameters.

Since the new expression only validates the given type at run-time, I've added the `new()` constraint on the `TClass` generic parameter, so the check for a parameterless constructor is done at compile-time.

This doesn't work if for some reason, at compile-time, we cannot reference the assembly with the type we need. Say you load an assembly to load a specific type, and you want to create an instance of that type. Than you cannot use the generic version. This means we need another method which can use a parameter of type Type, and create an instance of that.

After some fiddling around I came to the following methods.

{% highlight csharp %}
using System.Linq.Expressions;

public static partial class ObjectCreator {
	public static TClass CreateObject<TClass>() where TClass : new() {
		return CreateObject<TClass>(typeof(TClass));
	}
	public static object CreateObject(Type type) {
		return CreateObject<object>(type);
	}
	public static TBase CreateObject<TBase>(Type type) {
		Expression expression = Expression.New(type);
		if (type != typeof(TBase)) {
			expression = Expression.Convert(expression, typeof(TBase));
		}
		expression = Expression.Lambda(expression, null);
		Expression<Func<TBase>> creatorExpression = expression as Expression<Func<TBase>>;

		Func<TBase> creator = creatorExpression.Compile();
		if (creator != null) {
			return creator();
		}
		throw new Exception("Cannot create instance.");
	}
}
{% endhighlight %}

Now we have three methods.

* One if you can specify the type at compile time
* One if you cannot specify the type at compile time
* One if you cannot specify the type at compile-time, but you can specify a base-class of the type

It was necessary to alter the expression a bit, since the expression created by the lambda expression is of the type that is given, and cannot be cast to type `Expression<Func<object>>` for example. To work around this, a [convert expression](http://msdn.microsoft.com/en-us/library/bb292051.aspx) is inserted if the type to instance is different from the type to return.

Now we can create instances as follows:

{% highlight csharp %}
class MyTestClass { }
class MyDerivedTestClass : MyTestClass { }
class Program {
	static void Main(string[] args) {
		// Use the generic version
		MyTestClass c = ObjectCreator.CreateObject<MyTestClass>();

		// Use the parameter version
		object o = ObjectCreator.CreateObject(typeof(MyTestClass));

		// Use the baseclass+parameter version
		MyTestClass c2 = ObjectCreator.CreateObject<MyTestClass>(typeof(MyDerivedTestClass));
	}
}
{% endhighlight %}

Notes:

* The first method of the three now calls another overload. Since a `new()` constraint is added to the generic type TClass, we could simply do `return new TClass()`.
* In the next post(s) I will show how to create an instance of an object with constructor parameters. Also I will add caching of the expression trees, since in the above code the expression trees are generated every time when the method is called, and that is not necessary.