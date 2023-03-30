---
title: C#基础——反射
date: 2021-02-13 17:00:50
tags:
- C#
categories:
- Tech
---

> 通过`System.Reflection`命名空间中包含的类型，可以写代码来反射（或者说“解析”）这些元数据表。实际上，这个命名空间中的类型为程序及或模块中包含的元数据提供了一个对象模型。
> ——《CLR via C#》

简单来说，反射就是程序通过读取“编译器创建的元数据表”来获得程序集、模块、类型本身信息（属性）的功能。

<!--more-->

通过反射，程序可以获得对象的类型，动态地创建实例，访问其中的字段和属性，调用其中的方法。以及在运行时访问类型的特性【Attribute】(本文未提及)。

以下为使用C#的反射特性时常用的接口和实现方式：


```c#
using System;
using System.Collections.Generic;
using System.Reflection;
using System.Text;

namespace Reflection
{
    public class Class
    {
        public class Student
        {
            public string Name { get; set; }
        }
    }

    //基类
    class BaseClass
    {
        public int BaseField = 0;
    }
    //派生类
    class DerivedClass : BaseClass
    {
        public int DerivedField = 0;
    }

    public class InvokeClass
    {
        private string _testString;
        private long _testInt;

        public InvokeClass()
        {

        }

        public InvokeClass(string abc)
        {
            _testString = abc;
        }
        public InvokeClass(StringBuilder abc)
        {
            _testString = abc.ToString();
        }

        public InvokeClass(string abc, long def)
        {
            _testString = abc;
            _testInt = def;
        }

        public void TestMethod()
        {
            Console.WriteLine("我调用了InvokeClass的TestMethod方法");
        }

        public string TestFormat(string content)
        {
            return $"我调用了InvokeClass的TestFormat方法，传入了{content}参数";
        }
    }

    class Program
    {
        delegate int delegateOperate(int a, int b);

        private static void PrintTypeName(Type t)
        {
            Console.WriteLine($"NameSpace: {t.Namespace}");
            Console.WriteLine($"Name :{t.Name}");
            Console.WriteLine($"FullName: {t.FullName}");
        }

        public static int StaticSum(int a, int b)
        {
            return a + b;
        }
        public int InstanceSum(int a, int b)
        {
            return a + b;
        }

        static void Main(string[] args)
        {
            #region 获取类型和其中的字段

            var bc = new BaseClass();
            var dc = new DerivedClass();
            BaseClass[] bca = new BaseClass[] { bc, dc };
            foreach (var v in bca)
            {
                //获取类型
                Type t = v.GetType();
                Console.WriteLine("Object Type: {0}", t.Name);
                //获取类中的字段
                FieldInfo[] fi = t.GetFields();
                foreach (var f in fi)
                    Console.WriteLine("     Field:{0}", f.Name);
                Console.WriteLine();
            }
            Console.WriteLine("End!\n");
            Console.ReadKey();

            /*
            输出结果：

            Object Type: BaseClass
            Field:BaseField

            Object Type: DerivedClass
                Field:DerivedField
                Field:BaseField

            End!
            */

            #endregion

            #region 获取数组类型

            var intArray = typeof(int).MakeArrayType();
            var int3Array = typeof(int).MakeArrayType(3);

            Console.WriteLine($"是否是int 数组 intArray == typeof(int[]) ：{intArray == typeof(int[]) }");
            Console.WriteLine($"是否是int 3维数组 intArray3 == typeof(int[]) ：{int3Array == typeof(int[]) }");
            Console.WriteLine($"是否是int 3维数组 intArray3 == typeof(int[,,])：{int3Array == typeof(int[,,]) }");

            //数组元素的类型
            Type elementType = intArray.GetElementType();
            Type elementType2 = int3Array.GetElementType();

            Console.WriteLine($"{intArray}类型元素类型：{elementType }");
            Console.WriteLine($"{int3Array}类型元素类型：{elementType2 }");

            //获取数组的维数
            var rank = int3Array.GetArrayRank();
            Console.WriteLine($"{int3Array}类型维数：{rank }");
            Console.WriteLine("End!\n");
            Console.ReadKey();

            /*
            输出结果：

            是否是int 数组 intArray == typeof(int[]) ：True
            是否是int 3维数组 intArray3 == typeof(int[]) ：False
            是否是int 3维数组 intArray3 == typeof(int[,,])：True
            System.Int32[]类型元素类型：System.Int32
            System.Int32[,,]类型元素类型：System.Int32
            System.Int32[,,]类型维数：3
            End!
            */

            #endregion

            #region 嵌套类型

            var classType = typeof(Class);

            foreach (var t in classType.GetNestedTypes())
            {
                Console.WriteLine($"NestedType ={t}");
                //获取一个值，该值指示 System.Type 是否声明为公共类型。
                Console.WriteLine($"{t}访问 {t.IsPublic}");
                //获取一个值，通过该值指示类是否是嵌套的并且声明为公共的。
                Console.WriteLine($"{t}访问 {t.IsNestedPublic}");
            }

            Console.WriteLine("End!\n");
            Console.ReadKey();

            /*
            输出结果：

            NestedType =Reflection.Class+Student
            Reflection.Class+Student访问 False
            Reflection.Class+Student访问 True
            End! 
            */

            #endregion

            #region 获取类型名称

            var type = typeof(Class);
            Console.WriteLine($"\n------------一般类型-------------");
            PrintTypeName(type);

            //嵌套类型
            Console.WriteLine($"\n------------嵌套类型-------------");
            foreach (var t in type.GetNestedTypes())
            {
                PrintTypeName(t);
            }

            var type2 = typeof(Dictionary<,>);            //非封闭式泛型
            var type3 = typeof(Dictionary<string, int>);  //封闭式泛型

            Console.WriteLine($"\n------------非封闭式泛型-------------");
            PrintTypeName(type2);
            Console.WriteLine($"\n------------封闭式泛型-------------");
            PrintTypeName(type3);

            Console.WriteLine("End!\n");
            Console.ReadKey();

            /* 
            输出结果：

            ------------一般类型-------------
            NameSpace: Reflection
            Name :Class
            FullName: Reflection.Class

            ------------嵌套类型-------------
            NameSpace: Reflection
            Name :Student
            FullName: Reflection.Class+Student

            ------------非封闭式泛型-------------
            NameSpace: System.Collections.Generic
            Name :Dictionary`2
            FullName: System.Collections.Generic.Dictionary`2

            ------------封闭式泛型-------------
            NameSpace: System.Collections.Generic
            Name :Dictionary`2
            FullName: System.Collections.Generic.Dictionary`2[[System.String, mscorlib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089],[System.Int32, mscorlib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089]]
            End!
            */

            #endregion

            #region 获取基类和所继承的接口

            var base1 = typeof(System.String).BaseType;

            foreach (var iType in typeof(Int32).GetInterfaces())
            {
                Console.WriteLine($"iType : {iType.Name}");
            }

            #endregion

            //实际上，通过反射实例化对象、调用方法或委托、获取设置字段值的效率相当低下
            //除此之外，通过反射使用方法或字段会因为避开了编译器检查而成为Bug之源
            //日常需要避免使用

            #region 实例化对象

            //方法1，CreateInstance。但是如果类中没有参数列表对应的构造函数程序将报错
            var dateTime1 = (DateTime)Activator.CreateInstance(typeof(DateTime),
                2019, 6, 19);
            Console.WriteLine($"DateTime1: {dateTime1}");

            //方法2，使用ConstructInfo指定构造函数
            //找到“只有一个参数为string”的构造函数
            ConstructorInfo constructorInfo = typeof(InvokeClass).GetConstructor(
                new[] { typeof(string) });
            //使用该构造函数传入一个null参数
            InvokeClass obj4 = (InvokeClass)constructorInfo.Invoke(
				new object[] { null });

            //可利用ConstructInfo查找对应的构造函数
            ConstructorInfo[] constructorInfos = 
				typeof(InvokeClass).GetConstructors();
            var constructorInfoArray2 = Array.FindAll(constructorInfos,
                x => x.GetParameters().Length == 2);
            //找到第二个参数是long类型的构造函数
            var constructorInfo2 = Array.Find(constructorInfoArray2,
                x => x.GetParameters()[1].ParameterType == typeof(long));
            //如果存在，就创建对象
            if (constructorInfo2 != null)
            {
                var obj5 = (InvokeClass)constructorInfo2.Invoke(
                    new object[] { "abc", 123 });
            }

            #endregion

            #region 实例化委托

            //静态方法的委托
            Delegate staticD = Delegate.CreateDelegate(typeof(delegateOperate),
				typeof(Program), "StaticSum");
            //实例化对象方法的委托
            Delegate instanceD = Delegate.CreateDelegate(typeof(delegateOperate), 
				new Program(), "InstanceSum");

            Console.WriteLine($"staticD：{staticD.DynamicInvoke(1, 2)}");
            Console.WriteLine($"instanceD：{instanceD.DynamicInvoke(10, 20)}");

            #endregion

            #region 泛型的实例化

            Type closed = typeof(List<int>);
            Type unBound = typeof(List<>);

            //通过MakeGenericType把未封闭的泛型转为封闭的
            Type newClosed = unBound.MakeGenericType(typeof(int));
            //通过GetGenericTypeDefinition获得未封闭的泛型
            Type newUnBound = closed.GetGenericTypeDefinition();

            Console.WriteLine($"List<int> 类型{closed}");
            Console.WriteLine($"List<> 类型{unBound}");
            Console.WriteLine($"List<> MakeGenericType执行后 类型{newClosed}");
            Console.WriteLine($"List<int> GetGenericTypeDefinition执行后 类型{newUnBound}");

            #endregion

            #region 调用类中方法

            //获得一个类型的时候需要完整的"命名空间"+"类名"
            //ex.CLR只知"System.Int32"不知"int"
            classType = Type.GetType(typeof(InvokeClass).FullName);
            InvokeClass instance = (InvokeClass)Activator.
				CreateInstance(classType);

            //通过反射调用的方法必须为public，否则返回null
            //调用无参数的TestMethod
            MethodInfo method = classType.GetMethod(
				"TestMethod", new Type[] { });
            method.Invoke(instance, null);

            //调用有一个string参数的TestFormat方法
            method = classType.GetMethod("TestFormat", 
				new Type[] { typeof(string) });
            string result = method.Invoke(instance, 
				new object[] { "ViE" }).ToString();
            Console.WriteLine(result);
            
            Console.WriteLine("End!\n");
            Console.ReadKey();

            /*
            输出结果：
            
            iType : IComparable
            iType : IFormattable
            iType : IConvertible
            iType : IComparable`1
            iType : IEquatable`1
            DateTime1: 2019/6/19 0:00:00
            staticD：3
            instanceD：30
            List<int> 类型System.Collections.Generic.List`1[System.Int32]
            List<> 类型System.Collections.Generic.List`1[T]
            List<> MakeGenericType执行后 类型System.Collections.Generic.List`1[System.Int32]
            List<int> GetGenericTypeDefinition执行后 类型System.Collections.Generic.List`1[T]
            我调用了InvokeClass的TestMethod方法
            我调用了InvokeClass的TestFormat方法，传入了ViE参数
            End! 
            */

            #endregion
        }
    }
}

```
