---
Author: Chris Ellis
Date: 2014-08-15
Category: blog/development
Code: java
---
# Java 8

I've been using Java 8 since it was released earlier this year and have found 
some of the new features game changing.  To the extent I've now moved most of 
my projects to make use of new Java 8 features.  Support for Java 8 in Eclipse 
Luna is good and I've not run into any major issues using Java 8.

## Lambda Expressions

The single biggest feature in Java 8 is support for Lambda expressions, this
further supports more functional programming styles in Java. Java has always had 
closures via its anonymous classes functionality, these have often been used to 
implement callbacks.

Lambda expressions are extremely useful when working with collections.  As a 
simple example, filtering a List prior to Java 8 would require something along 
the lines of:

    // List<String input;
    List<String> filtered = new LinkedList<String>();
    for (String e : input)
    {
        if ("filter".equals(e))
            filtered.add(e);
    }

With Java 8 we can transform this into:

    // List<String input;
    List<String> filtered = input.stream()
                            .filter((e) -> { return "filter".equals(e); })
                            .collect(Collectors.toList());

Certainly a major improvement in semantics and readability.  While using Lambda 
expressions would be slightly slower.  Java 8 has taken care to implement them 
as efficiently as possible, by making use of `InvokeDynamic` and compiling each 
Lambda expression to a `synthetic` method.

Having been using Java 8 for the last few months, I can honestly say that 
Lambda expressions have changed how I code.  The addition of Lambda expressions 
has made as much of an impact as adding generics in Java 5 did.

## Default Methods

Default methods allow concrete functionality to be added to interface methods.  
Prior to Java 8 methods of an interface could only be abstract.  Interfaces 
defined how objects should be interacted with only, they were Java's solution 
to multi-inheritance while attempting to avoid some of the issues with it.

I've always liked the simplicity of the Java object model, however at times it 
was a straitjacket for certain use cases.

Default methods seem like a good compromise between flexibility and simplicity.  
I've found them useful for avoiding having to copy and paste trivial code.

For example:

    public interface Parameterised
    {
        List<Parameter> getParameters();
        
        default Parameter getParameter(String name)
        {
            return this.getParameters().stream()
                    .filter((p) -> {return name.equals(p.getName());})
                    .findFirst()
                    .get();
        }
    }

## Repeatable Annotations

I really like annotations in Java, they allow metadata to be added to code 
elements.  This is really handy for frameworks which can then use this 
information to customise how objects are interacted with, allowing for more 
declarative coding.

Since annotations were added in Java 5, I've never understood why they were not 
repeatable, it seems obvious that they should be.  Its a shame that it has taken 
until Java 8 to address this limitation.

I make heavy use of annotations in Balsa to declare routes (a route handles a 
specific HTTP request for an application).  Annotations give a rather nice way 
to declare this routing information, making it simple and readable to declare 
routes.  Allowing developers to focus upon the actual functionality of the 
application.

Prior to Java 8 to make annotations repeatable, you needed to define another 
annotation to contain them.  The user would then need to define both annotations 
on whatever they were annotating.

For example, the API developer defines the following annotations:

    public @interface RequirePermission
    {
        String value();
    }

    public @interface RequirePermissions
    {
        RequirePermission[] value();
    }

To consume the API, we would then do:

    @RequirePermissions({
        @RequirePermission("ui.access"),
        @RequirePermission("ui.read")
    })
    public void myHttpRoute()
    {
    }

With Java 8, the API developer only needs to annotate the singular annotation as 
repeatable:

    @Repeatable(RequirePermissions.class)
    public @interface RequirePermission
    {
        String value();
    }

This has the advantage for the API developer that it doesn't alter how they 
process the annotations.  However for the API consumer life is a little easier, 
as we can now do:

    @RequirePermission("ui.access")
    @RequirePermission("ui.read")
    public void myHttpRoute()
    {
    }

That makes things a fair bit easier and doesn't have any backwards compatibility 
problems, quite a clever solution really.

# Nashorn

Nashorn is a new Javascript engine for Java, it is fast and easy to work with.  
It boasts performance comparable to that of Google's V8 and has the massive 
advantage of being able to make use of any Java APIs from Javascript, including 
threading.  Again it makes use of `InvokeDynamic` for performance.  It is 
usable via the `ScriptEngine` API as well as directly from the command line.

The quickest way to have a play with Nashorn is via `jjs` on the command line:

    jjs> print("Hello World");
    Hello World
    jjs> exit();

It isn't that hard to execute a script from Java either:

    // create the script engine
    ScriptEngineManager factory = new ScriptEngineManager();
    ScriptEngine script = factory.getEngineByName("nashorn");
    // execute
    script.eval("print(\"Hello World\");");

To pass variables into the `ScriptEngine`, we need to setup some bindings:

    SimpleBindings bindings = new SimpleBindings();
    bindings.put("message", "Hello World");
    script.setBindings(bindings, ScriptContext.ENGINE_SCOPE);

Variables are contained by a `ScriptEngine` context, and are not shared across 
different `ScriptEngine` instances, we can change the previous example to:

    // create the script engine
    ScriptEngineManager factory = new ScriptEngineManager();
    ScriptEngine script = factory.getEngineByName("nashorn");
    // bindings
    SimpleBindings bindings = new SimpleBindings();
    bindings.put("message", "Hello World");
    script.setBindings(bindings, ScriptContext.ENGINE_SCOPE);
    // execute
    script.eval("print(message);");

As mentioned Nashorn allows Javascript to inter-operate with Java, Javascript 
can invoke Java methods and Java can invoke Javascript functions.  Nashorn also 
automatically maps functions to single method interfaces.  For example, we can 
create a new thread to print `Hello World` twice a second:

    jjs> new java.lang.Thread(function() { while (1) { print("Hello World"); java.lang.Thread.sleep(500); } }).start();
