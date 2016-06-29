##Writing a schema
Example:
```c
//example IDL file
namespace MyGame;

//define a attribute
attribute "priority";
//define a enum type
enum Color : byte{ Red = 1,Green,Blue }
//define a union
union Any { Monster,Weapon,Pickup }
//define a struct
struct Vector3{
    x:float;
    y:float;
    z:float;
}
//define a power table
table Monster{
    pos:Vector3;
    mana:short = 150; //give a default value
    hp:short:100;
    name:string;
    friendly:bool = false(deprecated,priority:1)
    inventory:[ubyte]; //define a ubyte array
    color:Color = Blue;
    test:Any;
}
//finally,must define a root type
root_type Monster;
```

###1.Tables
Tables are the main way of defining objects in FlatBuffers,and consist of a name and a list of field.Each field has a name,a type,and optionally a default value(default to `0` or `NULL`).
* add new fields in the schema **ONLY** at the end of a table definition.Older data will read correctly,and give you the default value when read.Older code will simply ignore the new field.as can manually assign ids,see the `id` attribute below.
* cannot delete field don't use anymore from the schema,but can simply stop writing them into data for the same effect.Additionally can mark as `deprecated`,will prevent the generation of accessors in the generated C++.
* You may change field names and table names,if you're ok with code breaking until renamed them there too.

###2.Structs
* Structs all field must `required`,and field may not be added or be deprecated.
* Structs only contains `sclaras` or `other structs`.
* Structs use **less memory** than tables and are **faster** to access(use no virtual table).

###3.Types
Built-in scalar types:
* 8bit:`byte`,`ubyte`,`bool`
* 16bit:`short`,`ushort`
* 32bit:`int`,`uint`,`float`
* 64bit:`long`,`ulong`,`double`

Built-in non-scalar types:
* Array of any other type(`[type]`),but **nesting vector** is not supported.
* `string`,only hold UTF-8 or 7-bit ASCII.For other text encoding or general binary data use vectors(`[byte]` or `[ubyts]`) instead.
* References tor other tables or structs,enums or unions.

###4.(Default)Values
* only scalar values can have default,non-scalar(string/vector/table) field default to `NULL`.

###5.Enums
* The default first value is `0`.
* can define a enum type,such as`enum color:byte`
* enum values should only ever be added,never removed.

###6.Unions
* Unions share a lot of properties with enums,but instead of new names for constants.Unions hold a References to any of those types,and additionally a hidden field with the suffix `_type` is geneeated that holds the corresponding enum value,allow to know which type to cast to at runtime.
* Unions are a good way to be able to send multiple message type as a FlatBuffer.
* A Union field is really two field,it must always be part of a table,it cannot be the root of a FlatBuffer.

###7.Namespaces
* use keyword `namespace` declared a namespaces.
* namespaces can use `.` to specify nested namespaces/packages.

###8.Includes
use keyword `include`,include other schemas file.Example:`include "mydefinitions.fbs"`;
This makes it easier to refer to types defined elsewherer.`include` automatically ensures each file is parsed just once,even when referred to more than once.When using the `flatc` compiler to generate code for schema definitions,only definitions in the current file will be generated,not those from included file(those still generated separately).

###9.Root type
This declares what you consider to be the root `table` (or `struct`) of the serialized data.

###10.File identification and extension
* Typically,a FlatBuffer binary buffer is not **self-describing**.specify in a schema that intend for this type of FlatBuffer to be used as a file format:`file_identifier "MYFI";`,identifiers must always be exactly **4 characters long**.
* by default `flatc` will output binary files as `.bin`.use `file_extension` define a custom extension:`file_extension "ext";`

###11.RPC interface declarations
Declare RPC calls in a schema,that define a set of functions that takes a Flatbuffer as an argument and return a Flatbuffer as the response(both of which must be table typse).
```c
rpc_service MonsterStorage{
    Store(Monster):StoreResponse;
    Retrieve(MonsterId):Monster;
}
```
###12.Comments & documentation
* a triple comment(*///*) on a line.

###13.Attribute
attribute may be attached to a declaration,behind a field,or after the name of a table/struct/enum/union.
* **id:n** (on a table field):manually set the field identifier to `n`.if use this a attribute,must use it on ALL field of this table,and the number must be a contiguous range from 0 onwards.
* **deprecated** (on a field):do not generated accessors for this field anymore,code should stop using this data.
* **required** (on a non-scalar table field):this field must always be set.By specifying this field,force code that constructs FlatBuffer to ensure this field is initialized.
* **original_order** (on a table):elements in a table do not need to be stored in any particular order,ofen optimized for space by sorting them to size.
* **force_align:size**(on a struct):force the alignment of this struct to be something higher than what it is naturally aligned to.
* **bit_flags** (on an enum):the values of this field indicate bit.
* **nested_flatbuffer:"table_name"** (on a field):this indicates that the field(which must be a vector of ubyte) contains flatbuffer data.
* **key** (on a field):this field is meant to be used as a key when sorting a vector of the type of table it sits in.(use in-place binary search).

###14.JSON Parsing
