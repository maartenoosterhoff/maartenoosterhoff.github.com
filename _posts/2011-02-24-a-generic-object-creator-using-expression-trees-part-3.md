---
layout: post
title: A generic object creator using expression trees, part 3
description: "Creating objects using expression trees"
permalink: a-generic-object-creator-using-expression-trees-part-3
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

* [Create an instance of an object using the constructor with no parameters](/a-generic-object-creator-using-expression-trees-part-1)
* [Create an instance of an object using the constructor with parameters](/a-generic-object-creator-using-expression-trees-part-2)
* [Caching the compiled expression tree](/a-generic-object-creator-using-expression-trees-part-3)

In the previous posts in this series I showed how to create an instance of an object, using expression trees, both without parameters, and with parameters. Now, unless the compiled expression tree is saved somewhere it's not going to be very quick, since the compilation of the expression tree into a lambda is the most expensive action.

For this purpose, I'll create a simple class LambdaCacher which will keep a dictionary containing the lambdas with their keys. The key for a lambda is the return type in combination with the types of the constructor arguments. To create a key is simple: just concatenate all Type.FullName values, and I'm using a pipe `|` to separate the full names.

If you look back at the previous post there is a method called CreateLambda in the ObjectCreator class, which creates the lambda. This will be the perfect place to use the new LambdaCacher class.

The LambdaCacher class will be static, and will have two static methods from an external point of view, each with two overloads. A LoadLambda method to retrieve an earlier compiled lambda, and a SaveLambda method to store a compiled lambda.

Internally the LambdaCacher class will have a dictionary of some key, and the lambdas as value. Since the lambdas are specified using generics, the dictionary will have a key of type string, and a value of type object. Some things are added to enforce thread safety.

{% highlight csharp %}
using System.Linq;
 
internal static class LambdaCacher {
	private static readonly object _lock = new object();
	private static readonly Dictionary<string, object> _lambdas = new Dictionary<string, object>();

	private static string GenerateKey(Type type, Type[] types) {
		string key = type.FullName + "|";
		if (types != null) {
			key += string.Join("|", types.Select(t => t.FullName).ToArray());
		}
		return key;
	}

	public static TFunc LoadLambda<TFunc>(Type type) where TFunc : class {
		return LoadLambda<TFunc>(type, null);
	}

	public static TFunc LoadLambda<TFunc>(Type type, Type[] types) where TFunc : class {
		lock (_lock) {
			string key = GenerateKey(type, types);
			if (_lambdas.ContainsKey(key)) {
				return (TFunc)_lambdas[key];
			}
			return null;
		}
	}

	public static void SaveLambda<TFunc>(TFunc func, Type type) {
		SaveLambda<TFunc>(func, type, null);
	}

	public static void SaveLambda<TFunc>(TFunc func, Type type, Type[] types) {
		lock (_lock) {
			string key = GenerateKey(type, types);
			if (!_lambdas.ContainsKey(key)) {
				_lambdas.Add(key, func);
			} else {
				throw new ArgumentException("A lambda is already saved for this type (and parameter types is given).");
			}
		}
	}
}
{% endhighlight %}

And now we have to use the new CreateLambda method in the ObjectCreate class.

{% highlight csharp %}
...
        private static TFunc CreateLambda<TBase, TFunc>(Type type, ParameterExpression[] parameters, Type[] types)
            where TBase : class
            where TFunc : class {
            var constructor = type.GetConstructor(types);
            if (constructor == null)
                throw new Exception(string.Format("Type '{0}' has no constructor with the specified parameters.", type.FullName));
 
            TFunc lambda = LambdaCacher.LoadLambda<TFunc>(type, types);
            if (lambda != null)
                return lambda;
 
            Expression expression = Expression.New(constructor, parameters);
            if (type != typeof(TBase)) {
                expression = Expression.Convert(expression, typeof(TBase));
            }
            expression = Expression.Lambda(expression, parameters);
            var creatorExpression = expression as Expression<TFunc>;
 
            lambda = creatorExpression.Compile();
            LambdaCacher.SaveLambda<TFunc>(lambda, type, types);
            return lambda;
        }
...
{% endhighlight %}

Of course, the question is now if it actually makes any difference. A test is in order.

* The first test will create 1 object.
* Running without the lambda cacher: 0.0103251 seconds
* Running with the lambda cacher: 0.0272546 seconds

* The first test will create 10 object.
* Running without the lambda cacher: 0.0429771 seconds
* Running with the lambda cacher: 0.0177748 seconds

* The second test will create 100.000 objects.
* Running without the lambda cacher: 25.31 seconds
* Running with the lambda cacher: 0.0693901 seconds

In the first two tests I noticed that the times were not always identical. I suspect that the times necessary are too small to make a real difference, and some startup costs might be different each time. But the third test does show how effective the caching the of lambdas is.

In the next post I'll do some performance testing comparing Reflection and the usage of expression trees.

The code as it is now can be downloaded [here](https://sites.google.com/site/dotnettravels/ObjectCreator_20110224.zip).

