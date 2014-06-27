---
layout: post
title: A generic object creator using expression trees, part 2
description: "Creating objects using expression trees"
permalink: a-generic-object-creator-using-expression-trees-part-2
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

[1. Create an instance of an object using the constructor without parameters](/a-generic-object-creator-using-expression-trees-part-1)
[2. Create an instance of an object using the constructor with parameters](/a-generic-object-creator-using-expression-trees-part-2)

In the [previous post](/a-generic-object-creator-using-expression-trees-part-1) in this series I showed how to create an instance of an object, using expression trees, using the constructor with no parameters. In this post I will show how to do it using a constructor with parameters.

This was a little more difficult. My biggest problem was getting the expression parameters right. Getting the expression itself right was not that difficult, it is almost the same as the previous one.

We now have to use [another overload](http://msdn.microsoft.com/en-us/library/bb351483.aspx) of the [new expression](http://msdn.microsoft.com/en-us/library/system.linq.expressions.expression.new.aspx). It uses a [ConstructorInfo](http://msdn.microsoft.com/en-us/library/system.reflection.constructorinfo.aspx) object to specify which constructor overload you want to use, and you have to supply the correct parameters.

First step is to retrieve a ConstructorInfo object. This can be retrieved from the type, using the [GetContructor](http://msdn.microsoft.com/en-us/library/system.type.getconstructor.aspx) method.

{% highlight csharp %}
using System.Reflection;
 
public static partial class ObjectCreator {
	public static TClass CreateObject<TClass, TParam1>(Func<TParam1> param1Provider) {
		var constructorInfo = typeof(TClass).GetConstructor(new Type[] { typeof(TParam1) });
		// TODO: Finish this
	}
}
{% endhighlight %}

Once we have the ConstructorInfo object, we can use that to create a new expression.

{% highlight csharp %}
using System.Reflection;
using System.Linq.Expressions;
 
public static partial class ObjectCreator {
	public static TClass CreateObject<TClass, TParam1>(Func<TParam1> param1Provider) {
		var constructorInfo = typeof(TClass).GetConstructor(new Type[] { typeof(TParam1) });
		var parameter = Expression.Parameter(typeof(TParam1));
		var newExpression = Expression.New(constructorInfo, parameter);
		// TODO: Finish this
	}
}
{% endhighlight %}

And now we can create a lambda expression using the new expression.

{% highlight csharp %}
using System.Reflection;
using System.Linq.Expressions;
 
public static partial class ObjectCreator {
	public static TClass CreateObject<TClass, TParam1>(Func<TParam1> param1Provider) {
		var constructorInfo = typeof(TClass).GetConstructor(new Type[] { typeof(TParam1) });
		var parameter = Expression.Parameter(typeof(TParam1));
		var newExpression = Expression.New(constructorInfo, parameter);
		var creatorExpression = Expression.Lambda(newExpression, parameter) as Expression<Func<TParam1, TClass>>;
		// TODO: Finish this
	}
}
{% endhighlight %}

Now we only have to compile the expression to get the lambda itself, and execute it.

{% highlight csharp %}
using System.Reflection;
using System.Linq.Expressions;
 
public static partial class ObjectCreator {
	public static TClass CreateObject<TClass, TParam1>(Func<TParam1> param1Provider) {
		var constructorInfo = typeof(TClass).GetConstructor(new Type[] { typeof(TParam1) });
		var parameter = Expression.Parameter(typeof(TParam1));
		var newExpression = Expression.New(constructorInfo, parameter);
		var creatorExpression = Expression.Lambda(newExpression, parameter) as Expression<Func<TParam1, TClass>>;
		var creator = creatorExpression.Compile();
		if (creator != null) {
			return creator(param1Provider());
		}
		throw new Exception("Cannot create instance.");
	}
}
{% endhighlight %}

This only gives us a method to use a constructor with a single parameter, but the methods for multiple parameter are essentially copies of the method above. With some refactoring I come to the following code.

{% highlight csharp %}
using System.Reflection;
using System.Linq;
using System.Linq.Expressions;
 
public static partial class ObjectCreator {
	public static TClass CreateObject<TClass, TParam1>(Func<TParam1> param1Provider) {
		var parameters = CreateArray<Type, ParameterExpression>(t => Expression.Parameter(t), typeof(TParam1));
		var types = CreateArray<Type, Type>(t => t, typeof(TParam1));
		var creator = CreateLambda<TClass, Func<TParam1, TClass>>(parameters, types);
		if (creator != null) {
			return creator(param1Provider());
		}
		throw new Exception("Cannot create instance.");
	}

	public static TClass CreateObject<TClass, TParam1, TParam2>(Func<TParam1> param1Provider, Func<TParam2> param2Provider) {
		var parameters = CreateArray<Type, ParameterExpression>(t => Expression.Parameter(t), typeof(TParam1), typeof(TParam2));
		var types = CreateArray<Type, Type>(t => t, typeof(TParam1), typeof(TParam2));
		var creator = CreateLambda<TClass, Func<TParam1, TParam2, TClass>>(parameters, types);
		if (creator != null) {
			return creator(param1Provider(), param2Provider());
		}
		throw new Exception("Cannot create instance.");
	}

	public static TClass CreateObject<TClass, TParam1, TParam2, TParam3>(Func<TParam1> param1Provider, Func<TParam2> param2Provider, Func<TParam3> param3Provider) {
		var parameters = CreateArray<Type, ParameterExpression>(t => Expression.Parameter(t), typeof(TParam1), typeof(TParam2), typeof(TParam3));
		var types = CreateArray<Type, Type>(t => t, typeof(TParam1), typeof(TParam2), typeof(TParam3));
		var creator = CreateLambda<TClass, Func<TParam1, TParam2, TParam3, TClass>>(parameters, types);
		if (creator != null) {
			return creator(param1Provider(), param2Provider(), param3Provider());
		}
		throw new Exception("Cannot create instance.");
	}

	public static TClass CreateObject<TClass, TParam1, TParam2, TParam3, TParam4>(Func<TParam1> param1Provider, Func<TParam2> param2Provider, Func<TParam3> param3Provider, Func<TParam4> param4Provider) {
		var parameters = CreateArray<Type, ParameterExpression>(t => Expression.Parameter(t), typeof(TParam1), typeof(TParam2), typeof(TParam3), typeof(TParam4));
		var types = CreateArray<Type, Type>(t => t, typeof(TParam1), typeof(TParam2), typeof(TParam3), typeof(TParam4));
		var creator = CreateLambda<TClass, Func<TParam1, TParam2, TParam3, TParam4, TClass>>(parameters, types);
		if (creator != null) {
			return creator(param1Provider(), param2Provider(), param3Provider(), param4Provider());
		}
		throw new Exception("Cannot create instance.");
	}

	private static TFunc CreateLambda<TClass, TFunc>(ParameterExpression[] parameters, Type[] types) {
		var constructor = typeof(TClass).GetConstructor(types);
		if (constructor == null)
			throw new Exception(string.Format("Type '{0}' has no constructor with the specified parameters.", typeof(TClass).FullName));

		var newExpression = Expression.New(constructor, parameters);
		var creatorExpression = Expression.Lambda(newExpression, parameters) as Expression<TFunc>;
		return creatorExpression.Compile();
	}

	private static TResult[] CreateArray<TSource, TResult>(Func<TSource, TResult> transformer, params TSource[] source) {
		return source.Select(transformer).ToArray();
	}
}
{% endhighlight %}

Now we can create an instance of an object using constructor parameters.

{% highlight csharp %}
namespace Playground.Program {
    class MyTestClass {
        public int I { get; set; }
        public string J { get; set; }
        public MyTestClass() { }
        public MyTestClass(int i) { I = i; }
        public MyTestClass(int i, string j) : this(i) { J = j; }
    }
    class Program {
        static void Main(string[] args) {
            // Use the generic version with parameters
            MyTestClass o1 = ObjectCreator.CreateObject<MyTestClass, int>(() => 10);
            MyTestClass o2 = ObjectCreator.CreateObject<MyTestClass, int, string>(() => 10, () => "Test");
 
            List<int> baseList = new List<int>(new int[] { 1, 2, 3, 4, 5 });
            List<int> list = ObjectCreator.CreateObject<List<int>, IEnumerable<int>>(() => baseList);
        }
    }
}
{% endhighlight %}

Once again, this is just the generic version, but it is no problem to convert it to parameter versions as I've shown in the previous post in this series. But I'll include that when I upload the complete code, which I'll do in one of the following posts. In the next post I'll be saving the generated lambda's so they can be re-used at a later stage.
