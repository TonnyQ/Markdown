# Lambda表达式总结

## 1.输入参数
在Lambda表达式中，输入参数是Lambda运算符的坐标部分。包含参数的数量可以是0，1或者多个。只有当参数为1时，Lambda表达式左边的括号才可以省略。输入参数的数量大于或者等于2时，Lambda表达式左边的括号中的多个参数使用逗号(,)分隔。

Example:创建一个Lambda表达式，输入参数的数量为0，表达式将显示“hello，lambda!”。
```csharp
() => System.Console.WriteLine("hello,lambda!"); //仅仅参数为1时，括号可以省略
```
Example2：创建一个Lambda表达式，输入参数包含一个参数:m。
```csharp
m=>m*2; //equals (m) => m * 2;
```
Example3:创建一个Lambda表达式，输入参数包含m和n。
```csharp
(m,n) => m * n;
```
## 2.表达式或语句块
多个Lambda表达式可以构成Lambda语句块。语句块可以放到运算符的右边，作为Lambda的主体。根据主体的不同，Lambda表达式可以分为表达式Lambda和语句Lambda。语句块可以包含多条语句，并且可以包含循环、方法调用和if语句等;

Exampel:创建一个Lambda表达式，右边部分是一个表达式。
```csharp
m => m * m; //如果表达式的右边部分是一个语句块，那么该语句块应该被“{}”包围
```
Example2:输入参数为m,n;表达式的右边包含两个语句。
```csharp
(m,n) => {
    int result = m * n;
    System.Console.WriteLine("m * n = " + result);
}
```
## 3.查询表达式
查询表达式是一种使用查询语法表示的表达式，它用于查询和转换来自任意支持LINQ的数据源中的数据。由一组类似于SQL或XQuery的声明性语法编写的子句组成。每一个子句可以包含一个或多个C#表达式。这些C#表达式本身也可以是查询表达式或包含查询表达式。

查询表达式必须以`from`子句开头，以`select`或`group`子句结束。第一个`from`子句和最后一个`select`或`group`子句之间，可以包含一个或者多个`where`,`let`,`join`,`orderby`,`group`,`from`子句。
* **from**：指定查询操作的数据和范围变量;
* **select**:指定查询结果的类型和表达形式;
* **where**:指定筛选元素的逻辑条件;
* **let**:引入用来临时保存查询表达式中字表达式结果的范围变量;
* **orderby**:对查询结果进行排序操作，包括升序和降序;
* **group**:对查询结果进行分组;
* **into**:提供一个临时标识符；join，group或select可以通过该标识符引用查询操作中的中间结果;
* **join**:连接多个用于查询操作的数据源;

Example:创建一个查询表达式query，该查询表达式查询arr数组中的每一个元素;
```csharp
int arr[] = new int{1,2,34,4,5343,3};
var query1 = from n in arr select n;
```
Example2:创建一个查询表达式query2，查询数组中大于6的元素;
```csharp
//query2只是保存查询操作，而不是保存查询的结果。
var query2 = from n in arr
                where n > 6
                select n;
```
