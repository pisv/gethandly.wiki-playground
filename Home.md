# Get a Handle on Handly!
**A step-by-step guide to getting started with Eclipse Handly
<https://eclipse.org/handly/>**

Probably the best way to tell you about Handly is to walk you through the
development of an example. We'll use a simple Xtext-based language as the
basis for the running example. While nothing in the Handly Core depends
on Xtext, going this way will be a bit easier for you to get started.
The complete example is available as part of the Examples feature of Handly;
see the plug-ins `org.eclipse.handly.examples.basic` and
`org.eclipse.handly.examples.basic.ui`.

Let's start with a description of our language called Foo.
Here's its Xtext grammar:

```antlr
// Foo.xtext

Module:
  vars += Var*
  defs += Def*
;

Var:
  'var' name=ID ';'
;

Def:
  'def' name=ID '(' (params+=ID)? (',' params+=ID)* ')' '{' '}'
;
```

Even if you don't know Xtext, you should have no difficulties in understanding
the syntax of this language. Here's a code sample:

```scala
// sample.foo

var x;
var y;
def f() {}
def f(x) {}
def f(x, y) {}
```

The language is intentionally contrived to have only the simplest structural
elements: declarations of variables and functions. There are no type definitions
or symbolic references; variables don't have initializing expressions and
function bodies are always empty. Nevertheless, this language is a fertile ground
for a Handly example because it is simple so we won't have to explain a lot of
stuff that isn't Handly-related, but at the same time it is sufficient for
understanding the main aspects of implementation of a Handly-based model.
We further assume for simplification that `.foo` source files may only reside
directly under a Foo project, i.e. there is no concept of 'build path' involved.

Here's a sample structure of the workspace rendered from the Foo language's
point of view:

    /            - Foo model
     sample/     - Foo project
      sample.foo - Foo source file
       x         - Foo variable
       f(x)      - Foo function

As you can see, there are several kinds of elements here: **Foo model**
(the root element; corresponds to the workspace root), **Foo project**
(a project with the Foo nature), **Foo source file** (a file with the Foo
content type), **Foo variable** and **Foo function** (structural elements
inside a Foo file). These elements constitute a *code model* for the
Foo language, from the workspace root level down to structural elements
inside source files. In this article we'll walk you through the process
of implementing this model with Handly.

The article is organized as a step-by-step guide, each step taking you
around Handly in increasing detail.

1. [[Step Zero]] gets you set up for further development of the running example.
2. [[Step One]] makes the simplest possible Handly-based model.
3. [[Step Two]] takes the basic model and adds the functionality
expected of a code model for a programming language.
4. [[Step Three]] displays the finished model in a view.
5. [[Step Four]] brings the model truly to life with the working copy facility.