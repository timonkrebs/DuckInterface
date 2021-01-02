# DuckInterface

This repository contains my attempt to enable duck typing support in C#. It is powered by Roslyn and the new C# 9 feature the Source Generators. 
I would say it is purely academic/just for fun stuff, but for some scenarios, it can be useful.

[![Nuget](https://img.shields.io/badge/nuget-DuckInterface-blue?style=flat-square&logo=nuget)](https://www.nuget.org/packages/DuckInterface/)

# How to use it

Let's suppose that you have the next declaration:
``` cs 
public interface ICalculator
{
  float Calculate(float a, float b);
}

public class AddCalculator
{
  float Calculate(float a, float b);
}
```
It is important to notice that the ``` AddCalculator ``` doesn't implement a ``` ICalculator ``` in any way. It just has an identical method declaration.
If we try to use it like in the next snippet we will get a compilation error:

``` cs
var addCalculator = new AddCalculator();

var result = Do(addCalculator, 10, 20);

float Do(ICalculator calculator, float a, float b)
{
  return calculator.Calculate(a, b);
}

```
In this case, duck typing can be helpful, because it will allow us to pass ``` AddCalculator ``` with ease. The ``` DuckInterface ``` may help with it. 
You will need to install NuGet package and update the interface declaration like that:

``` cs 
[Duckable]
public interface ICalculator
{
  float Calculate(float a, float b);
}
```

Then we will need to update the ``` Do ``` method. Repace the ``` ICalculator ``` with a ``` DICalculator ```. 
The ``` DICalculator ``` it is a class that was generated by ``` DuckInterface ```. The ``` DICalculator ``` has a public interface identical to ``` ICalculator ``` and can contain implicit conversion operators for any class. Those implicit conversion operators are generated by the Source Generator too. The generation happens on a fly as you typing in IDE and depends on the ``` DICalculator ``` usage. 

The final snippet:
``` cs
var addCalculator = new AddCalculator();

var result = Do(addCalculator, 10, 20);

float Do(DICalculator calculator, float a, float b)
{
  return calculator.Calculate(a, b);
}

```
And it's done. The compilation errors are gone and everything works as expected.

# How it works

There are two independent source generators. The first one looks for ``` Duckable ``` attribute and generates 'base' class for the interface. 
For example, for the ``` ICalculator ``` it will look like that:
``` cs 
 public partial class DICalculator : ICalculator 
{
  [System.Diagnostics.DebuggerBrowsable(System.Diagnostics.DebuggerBrowsableState.Never)] 
  private readonly Func<float, float, float> _Calculate;        

  [System.Diagnostics.DebuggerStepThrough]
  public float Calculate(float a, float b)
  {
      return _Calculate(a, b);
  }
}
```

The second one looks for method call and variable assignments to undestand how the duckable interface may be used. 
For example, lets look for next snippet:
``` cs
var result = Do(addCalculator, 10, 20);
``` 

The analyzer will see that the ``` Do ``` method has argument with type ``` DICalculator ```, then it will check the type of ``` addCalculator ``` variable.
If the type has all required members, the source generator will extend the ``` DICalculator ```. The extension will look like that:
``` cs
public partial class DICalculator
{
  private DICalculator(global::AddCalculator value) 
  {
       _Calculate = value.Calculate;
  }

  public static implicit operator DICalculator(global::AddCalculator value)
  {
      return new DICalculator(value);
  }
}
```

Because the ``` DICalculator ``` is a partial class we can execute this trick as much time as we want.