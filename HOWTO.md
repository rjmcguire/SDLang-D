How to use SDLang-D (Tutorial / API Overview)
=============================================

SDLang-D offers two ways to work with SDL: DOM style and StAX/Pull style. DOM style is easier and more convenient and can both read and write SDL. StAX/Pull style is faster and more efficient, although it's only used for reading SDL, not writing it.

This document explains how to use SDLang-D in the DOM style. If you're familiar with StAX/Pull style parsing for other languages, such as XML, then SDLang-D's StAX/Pull parser should be fairly straightforward to understand. See [pullParseFile](http://semitwist.com/sdlang-d/sdlang/parser/pullParseFile.html) and [pullParseSource](http://semitwist.com/sdlang-d/sdlang/parser/pullParseSource.html) in the [API reference](http://semitwist.com/sdlang-d/sdlang.html) for details. You can also see SDLang-D's source as a real-world example, as the DOM parser is built directly on top of the StAX/Pull parser (just search ```parser.d``` for ```DOMParser```).

Installation
------------

The list of officially supported D compiler versions is always available in [.travis.yml](https://github.com/Abscissa/SDLang-D/blob/master/.travis.yml).

The recommended way to use SDLang-D is via [DUB](http://code.dlang.org/getting_started). Just add a dependency to ```sdlang-d``` in your project's ```dub.json``` or ```dub.sdl``` file [as shown here](http://code.dlang.org/packages/sdlang-d). Then simply build your project with DUB as usual.

Alternatively, you can ```git clone``` both SDLang-D and the latest version of [libInputVisitor](https://github.com/Abscissa/libInputVisitor), and include ```-I{path to SDLang-D}/src -I{path to libInputVisitor}/src``` when running the compiler.

Note that ```-inline``` currently causes some problems. On DMD 2.063.2 and below, it caused compilation to fail, likely due to [DMD #5776](https://issues.dlang.org/show_bug.cgi?id=5776) and/or [DMD #11377](https://issues.dlang.org/show_bug.cgi?id=11377). On DMD 2.064 and up, it causes a segfault when parsing a Base64 value (currently being investigated).

Importing
---------

To use SDL, first import the module ```sdlang```:

```d
import sdlang;
```

If you're not using DUB, then you must also include the path the SDLang-D sources when you compile:

```
rdmd --build-only -I{path to sdlang}/src -I{path to libInputVisitor}/src {other flags} yourProgram.d
```

Example
-------

[example.d](https://github.com/Abscissa/SDLang-D/blob/master/example.d):
```d
import std.stdio;
import sdlang;

int main()
{
    Tag root;
    
    try
    {
        // Or:
        // root = parseFile("myFile.sdl");
        root = parseSource(`
            welcome "Hello world"

            // Uncomment this for an error:
            // badSuffix 12Q

            myNamespace:person name="Joe Coder" {
                age 36
            }
        `);
    }
    catch(SDLangParseException e)
    {
        // Messages will be of the form:
        // myFile.sdl(5:28): Error: Invalid integer suffix.
        stderr.writeln(e.msg);
        return 1;
    }
    
    // Value is a std.variant.Algebraic
    Value welcome = root.tags["welcome"][0].values[0];
    assert(welcome.type == typeid(string));
    writeln(welcome);
    
    Tag person = root.namespaces["myNamespace"].tags["person"][0];
    writeln("Name: ", person.attributes["name"][0].value);
    
    int age = person.tags["age"][0].values[0].get!int();
    writeln("Age: ", age);
    
    // Output back to SDL
    writeln("The full SDL:");
    writeln(root.toSDLDocument());
    
    return 0;
}
```

Compile and run:
```console
> rdmd --build-only -I./src example.d
> example
Hello world
Name: Joe Coder
Age: 36
The full SDL:
welcome "Hello world"
myNamespace:person name="Joe Coder" {
	age 36
}
```

Main Interface: Parsing SDL
---------------------------

The main interface for SDLang-D is the two parse functions:

```d
/// Returns root tag.
Tag parseFile(string filename);

/// Returns root tag. The optional 'filename' parameter can be included
/// so that the SDL document's filename (if any) can be displayed with
/// any syntax error messages.
Tag parseSource(string source, string filename=null);
```

Beyond that, your interactions with SDL will be via ```class Tag```,
```struct Attribute``` and ```alias Value``` (an instantiation of [std.variant.Algebraic](http://dlang.org/phobos/std_variant.html)).

Tag and Attribute API Summary
-----------------------------

Ultimately, the Tag and Attribute APIs work like this (where ```{...}``` means "optional", and ```|``` means "or"):

```d
// Constructors:
Attribute.this(string namespace, string name, Value value,
               Location location = Location(0, 0, 0))
Attribute.this(string name, Value value,
               Location location = Location(0, 0, 0))

Attribute.namespace  // string: "" if no namespace
Attribute.name       // string: "" if anonymous
Attribute.fullName   // string: Read-only, returns "namespace:name" if there's a namespace
Attribute.location   // Location: filename, line, column and index in original SDL file
Attribute.value      // Value
Attribute.parent     // Tag: Read-only

// Constructors:
Tag.this(Tag parent = null)
Tag.this(string namespace, string name,
         Value[] values=null, Attribute[] attributes=null, Tag[] children=null)
Tag.this(Tag parent,
         string namespace, string name,
         Value[] values=null, Attribute[] attributes=null, Tag[] children=null)

Tag.namespace  // string: "" if no namespace
Tag.name       // string: "" if anonymous
Tag.fullName   // string: Read-only, returns "namespace:name" if there's a namespace
Tag.location   // Location: filename, line, column and index in original SDL file
Tag.values     // Value[]
Tag.parent     // Tag: Read-only

Tag.remove()   // Removes this tag from its parent
Tag.add(Tag   | Attribute   | Value  )   // Adds a member to this tag
Tag.add(Tag[] | Attribute[] | Value[])   // Adds multiple members to this tag

// Optional '.maybe' implies "If I lookup (by string) a name or namespace
// that doesn't exist, then return an empty range instead of throwing."
Tag{.maybe}.namespaces                       // RandomAccessRange
Tag{.maybe}.namespaces[string or index]      // Access child tags/attributes (see below)
Tag{.maybe}.namespaces[startIndex..endIndex] // Slicable
string (in|!in) Tag{.maybe}.namespaces       // Check if namespace exists

// Access child tags/attributes:
// - Optional '.maybe' explained above.
// - Optional '.all' means "All namespaces".
// - The default namespace is "".
Tag{.maybe}{.all | .namespaces[string or index]}.tags        // RandomAccessRange
Tag{.maybe}{.all | .namespaces[string or index]}.attributes  // RandomAccessRange

(tags|attributes)[index]                // Normal RandomAccessRange indexing
(tags|attributes)[startIndex..endIndex] // Slicable (but the slice can't use 'in' or '[string]')
string (in|!in) (tags|attributes)       // Check if a tag/attribute name exists
(tags|attributes)[string]               // Slicable RandomAccessRange of
                                        // tags/attributes with a specific name
```

All of the interfaces above are shallow. For example:

* ```Tag{.maybe}.namespaces``` only contains namespaces used by the current tag's attributes and immediate children - not descendants.
* ```.maybe```-ness ends when you reach another Tag. So, ```Tag.maybe.namespaces["invalid-namespace"].tags["invalid-name"]``` is ok and won't throw, but the following will throw: ```Tag.maybe.namespaces["ok-namespace"].tags["ok-name"][0].tags["invalid-name"]```.

Ranges will be invalidated if you add/remove/rename any child tags, attributes or namespaces, on the Tag which the Range operates on. But, if assertions and struct invariants are enabled, then this will be detected and any further attempt to use the invalidated range will throw an assertion failure.

Since this library is designed primarily for reading and writing SDL files, it's optimized for building and navigating trees rather than manipulating them. Keep in mind that removing or renaming tags, attributes or namespaces may be slow. If you're concerned about speed, it might be best to minimize direct manipulations and prefer using use the SDLang-D data structures as pure input/output.

Value
-----

The type ```Value``` is an instantiation of [std.variant.Algebraic](http://dlang.org/phobos/std_variant.html). It's defined like this:

```
SDL's datatypes map to D's datatypes as described below.
Most are straightforward, but take special note of the date/time-related types.

Boolean:                       bool
Null:                          typeof(null)
Unicode Character:             dchar
Double-Quote Unicode String:   string
Raw Backtick Unicode String:   string
Integer (32 bits signed):      int
Long Integer (64 bits signed): long
Float (32 bits signed):        float
Double Float (64 bits signed): double
Decimal (128+ bits signed):    real
Binary (standard Base64):      ubyte[]
Time Span:                     Duration

Date (with no time at all):           Date
Date Time (no timezone):              DateTimeFrac
Date Time (with a known timezone):    SysTime
Date Time (with an unknown timezone): DateTimeFracUnknownZone
```
```d
alias Algebraic!(
    bool,
    string, dchar,
    int, long,
    float, double, real,
    Date, DateTimeFrac, SysTime, DateTimeFracUnknownZone, Duration,
    ubyte[],
    typeof(null),
) Value;
```

Outputting SDL
--------------

To output SDL, simply call ```Tag.toSDLDocument()``` on whichever tag is your "root" tag. The root tag is simply used as a collection of tags. As such, its namespace must be blank and it cannot have any values or attributes. It can, however, have any name (which will be ignored), and it is allowed to have a parent (also ignored).

Additionally, tags, attributes and values all have a ```toSDLString()``` function, to convert just one Tag (any tag, not just a root tag), Attribute or Value to an SDL string.

The ```Tag.toSDLDocument()``` function and ```toSDLString()``` functions can optionally take an OutputRange sink instead of allocating and returning a string. The Tag-based functions also have optional parameters to customize the indent style and starting depth.

```d
class Tag
{
...
	/// Treats 'this' as the root tag. Note that root tags cannot have
	/// values or attributes, and cannot be part of a namespace.
	/// If this isn't a valid root tag, 'SDLangValidationException' will be thrown.
	string toSDLDocument()(string indent="\t", int indentLevel=0);
	void toSDLDocument(Sink)(ref Sink sink, string indent="\t", int indentLevel=0)
		if(isOutputRange!(Sink,char));
	
	/// Output this entire tag in SDL format. Does *not* treat 'this' as
	/// a root tag. If you intend this to be the root of a standard SDL
	/// document, use 'toSDLDocument' instead.
	string toSDLString()(string indent="\t", int indentLevel=0);
	void toSDLString(Sink)(ref Sink sink, string indent="\t", int indentLevel=0)
		if(isOutputRange!(Sink,char));
...
}

struct Attribute
{
...
	string toSDLString()();
	void toSDLString(Sink)(ref Sink sink) if(isOutputRange!(Sink,char));
...
}

string toSDLString(T)(T value) if(
	is( T : Value  ) ||
	is( T : bool   ) ||
	is( T : string ) ||
	/+...etc...+/
);
void toSDLString(Sink)(Value value, ref Sink sink) if(isOutputRange!(Sink,char));
void toSDLString(Sink)(typeof(null) value, ref Sink sink) if(isOutputRange!(Sink,char));
void toSDLString(Sink)(bool value, ref Sink sink) if(isOutputRange!(Sink,char));
void toSDLString(Sink)(string value, ref Sink sink) if(isOutputRange!(Sink,char));
//...etc...
```

Tag and Attribute Example
-------------------------

Consider the following SDL adapted from the [SDL Language Guide](http://sdl.ikayzo.org/display/SDL/Language+Guide) [[mirror](http://semitwist.com/sdl-mirror/Language+Guide.html)]

```
person "Akiko" "Johnson" dimensions:height=68 {
    son "Nouhiro" "Johnson"
    daughter "Sabrina" "Johnson" location="Italy" {
        info:hobbies "swimming" "surfing"
        info:languages "English" "Italian"
        info:smoker false
    }
}
```

That can be navigated like this:

[example2.d](https://github.com/Abscissa/SDLang-D/blob/master/example2.d):
```d
Tag root = parseSource(theSdlExampleAbove);

// Get the person tag:
//
// SDL supports multiple tags/attributes with the same name,
// therefore the [0] is needed.
Tag akiko = root.tags["person"][0];
assert( akiko.namespace == "" ); // Default namespace is ""
assert( akiko.name == "person" );
assert( akiko.values.length == 2 );
assert( akiko.values[0] == Value("Akiko") );
assert( akiko.values[1] == Value("Johnson") );

// Anonymous tags are named "": (Note: This is different from the Java SDL
// where anonymous tags are named "content".)
assert( root.tags[""][0].values[0].get!string().startsWith("This is ") );
assert( root.tags[""][0].values[1] == Value(123) );
assert( root.tags[""][1].values[0] == Value("Another anon tag") );

// Get Akiko-san's height attribute:
//
// Her attribute "height" is in the namespace "dimensions",
// so it must be accessed that way:
assert( "height" !in akiko.attributes );
assert( "height" in akiko.namespaces["dimensions"].attributes );
assert( "height" in akiko.all.attributes );  // 'all' looks in all namespaces
assert( akiko.all.attributes["height"].length == 1 );

Attribute akikoHeight = akiko.all.attributes["height"][0];
assert( akikoHeight.namespace == "dimensions" );
assert( akikoHeight.name == "height" );
assert( akikoHeight.value == Value(68) );
assert( akikoHeight.parent is akiko );

// She has no "weight" attribute:
assertThrown!SDLangRangeException( akiko.attributes["weight"] );
assertThrown!SDLangRangeException( akiko.all.attributes["weight"] );

// Use 'maybe' to get an empty range instead of an exception.
// This works on tags and namespaces, too.
assert( akiko.maybe.attributes["weight"].empty );
assert( akiko.maybe.all.attributes["weight"].empty );
assert( akiko.maybe.attributes["height"].empty );
assert( akiko.maybe.all.attributes["height"].length == 1 );
assert( akiko.maybe.namespaces["foo"].attributes["bar"].empty );
assert( akiko.maybe.namespaces["foo"].tags["bar"].empty );

// Show Akiko-san's child tags:
foreach(Tag child; akiko.tags)
	writeln(child.name); // Output: son daughter
writeln("--------------");

foreach(Tag child; akiko.namespaces["pet"].tags)
	writeln(child.name); // Output: kitty
writeln("--------------");

foreach(Tag child; akiko.all.tags)
	writeln(child.name); // Output: son kitty daughter
writeln("--------------");

// Get Akiko-san's daughter:
Tag daughter = akiko.tags["daughter"][0];

// You can also manually specify "default namepace",
// or lookup by index insetad of name. This works on attributes, too:
assert(daughter is akiko.namespaces[""].tags["daughter"][0]);
assert(daughter is akiko.tags[1]);      // Second child of Akiko-san
assert(daughter is akiko.all.tags[2]);  // Third if you include pets

// Akiko-san's namespaces, in order of first appearance in the SDL file:
assert(akiko.namespaces[0].name == "dimensions"); // First found in attribute "height"
assert(akiko.namespaces[1].name == "");           // First found in child "son"
assert(akiko.namespaces[2].name == "pet");        // First found in child "kitty"

// Everything is a random-access range:
// (Although 'Tag.values' is currently just a plain-old array)
auto allDaughters = akiko.all.tags.filter!(c => c.name == "daughter")();
assert( array(allDaughters).length == 1 );
assert( allDaughters.front is daughter );

// Everything can be safely modified. If assertions and struct invariants
// are enabled, any already-existing ranges will automatically detect when
// they've been potentially invalidated and throw an assertion failure.
//
// Keep in mind, the library is optimized for lookups, so removing and
// renaming tags, attributes or namespaces may be slow.
daughter.attributes["location"][0].value = Value("England");

auto kitty = akiko.all.tags["kitty"][0];
kitty.name = "cat";
assert( "kitty" !in akiko.all.tags );
assert( kitty is akiko.all.tags["cat"][0] );

akikoHeight.namespace = "stats";
assert( "dimensions" !in akiko.namespaces );
assert( "stats" in akiko.namespaces );
assert( akikoHeight == akiko.namespaces["stats"].attributes["height"][0] );

// Add/remove child tag. Also works with attributes.
Tag son = akiko.tags["son"][0];
Tag hobbies = daughter.tags["hobbies"][0];
// 'hobbies' is already attached to a parent tag.
assertThrown!SDLangValidationException( son.add(hobbies) );
hobbies.remove(); // Remove from daughter
son.add(hobbies); // Ok

/*
Output the modified SDL document:

"This is an anonymous tag with two values" 123
"Another anon tag"
person "Akiko" "Johnson" stats:height=68 {
	son "Nouhiro" "Johnson" {
		hobbies "swimming" "surfing"
	}
	pet:cat "Neko"
	daughter "Sabrina" "Johnson" location="England" {
		languages "English" "Italian"
		smoker false
	}
}
*/
stdout.rawWrite( root.toSDLDocument() );
writeln("--------------");

// Root tags cannot be part of a namespace or contain any values or attributes
assertThrown!SDLangValidationException( daughter.toSDLDocument() );
assertThrown!SDLangValidationException( kitty.toSDLDocument() );

root.add( new Attribute("attributeNamespace", "attributeName", Value(3)) );
assertThrown!SDLangValidationException( root.toSDLDocument() );

// But you can still convert such tags, or any other Tag, Attribute or Value,
// to an SDL string with 'toSDLString':
// 
// pet:cat "Neko"
stdout.rawWrite( kitty.toSDLString() );
```
