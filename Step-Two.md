# Step Two: The Rest of the Model

[[Step One]] has left us with the basics of a model. We have
a two-level hierarchy of `FooModel` containing `FooProject`s
and some resource delta processing for keeping the model up-to-date.
We will use it as a starting point for this step, where we will build
a complete *code model* for the Foo language, from the workspace root level
down to structural elements inside source files. The complete source code
for this step of the running example is available in the [Step Two repository]
(https://github.com/pisv/gethandly.2nd).

Recall that every model element in Handly is an instance of `IHandle`.
This interface is specialized with `ISourceElement` for representing
source elements, which is further specialized with `ISourceFile` and
`ISourceConstruct` for representing source files and their structural
elements. Handly provides basic partial implementations for each
of these interfaces.

As usual, we begin by defining interfaces for the new model elements
(in the package `org.eclipse.handly.examples.basic.ui.model`
of the `org.eclipse.handly.examples.basic.ui` bundle):

```java
/**
 * Represents a Foo source file.
 */
public interface IFooFile
    extends ISourceFile
{
    /**
     * Foo file extension.
     */
    String EXT = "foo";
}

/**
 * Represents a variable declared in a Foo file.
 */
public interface IFooVar
    extends ISourceConstruct
{
}

/**
 * Represents a function defined in a Foo file.
 */
public interface IFooDef
    extends ISourceConstruct
{
}
```

with corresponding implementation classes defined in the package
`org.eclipse.handly.internal.examples.basic.ui.model` of the same bundle:

```java
public class FooFile
    extends SourceFile
    implements IFooFile
{
}

public class FooVar
    extends SourceConstruct
    implements IFooVar
{
}

public class FooDef
    extends SourceConstruct
    implements IFooDef
{
}
```

Once more, we need to complete the implementation by defining the appropriate
constructors and overriding the inherited abstract methods. Let's begin
with the class `FooVar`:

```java
/**
 * Represents a variable declared in a Foo file.
 */
public class FooVar
    extends SourceConstruct
    implements IFooVar
{
    /**
     * Creates a handle for a variable with the given parent element 
     * and the given name.
     * 
     * @param parent the parent of the element (not <code>null</code>)
     * @param name the name of the element (not <code>null</code>)
     */
    public FooVar(FooFile parent, String name)
    {
        super(parent, name);
        if (name == null)
            throw new IllegalArgumentException();
    }
    
    @Override
    protected HandleManager getHandleManager()
    {
        return FooModelManager.INSTANCE.getHandleManager();
    }
}
```

That's all there is to it.

Next goes the class `FooDef`:

```java
/**
 * Represents a function defined in a Foo file.
 */
public class FooDef
    extends SourceConstruct
    implements IFooDef
{
    private final int arity;

    /**
     * Creates a handle for a function with the given parent element, 
     * the given name, and the given arity.
     * 
     * @param parent the parent of the element (not <code>null</code>)
     * @param name the name of the element (not <code>null</code>)
     * @param arity the arity of the function
     */
    public FooDef(FooFile parent, String name, int arity)
    {
        super(parent, name);
        if (name == null)
            throw new IllegalArgumentException();
        this.arity = arity;
    }

    @Override
    public boolean equals(Object obj)
    {
        if (!(obj instanceof FooDef))
            return false;
        return super.equals(obj) && arity == ((FooDef)obj).arity;
    }

    @Override
    public int hashCode()
    {
        final int prime = 31;
        int result = super.hashCode();
        result = prime * result + arity;
        return result;
    }

    @Override
    protected HandleManager getHandleManager()
    {
        return FooModelManager.INSTANCE.getHandleManager();
    }
}
```

As you can see, this model element is special in that its constructor takes
an additional parameter, `arity`. While variables can be identified solely by
their name within the parent Foo file, functions are a different story.
The name alone is not sufficient to address a Foo function -- we need
to also state the number of arguments (its *arity*).

Thus, we need to make `arity` part of the state of a `FooDef` instance,
the handle for a Foo function (as you may remember, handles are value objects
that hold immutable, 'key' information about a model element). We also need
to extend the `equals` and `hashCode` methods to take `arity` into account
(until now the inherited implementation of `equals` and `hashCode` was always
sufficient).

Let's introduce a getter method for `arity` in the `IFooDef` interface:

```java
// IFooDef.java

    /**
     * Returns the number of parameters of this function.
     * This is a handle-only method.
     *
     * @return the number of parameters of this function
     */
    int getArity();
```

with a trivial implementation:

```java
// FooDef.java

    @Override
    public int getArity()
    {
        return arity;
    }
```

To have the class `FooFile` compile without errors, we need to define
a constructor and override the inherited abstract methods:

```java
/**
 * Represents a Foo source file.
 */
public class FooFile
    extends SourceFile
    implements IFooFile
{
    /**
     * Constructs a handle for a Foo file with the given parent element 
     * and the given underlying workspace file.
     * 
     * @param parent the parent of the element (not <code>null</code>)
     * @param file the workspace file underlying the element
     *  (not <code>null</code>)
     * @throws IllegalArgumentException if the handle cannot be constructed 
     *  on the given workspace file
     */
    public FooFile(FooProject parent, IFile file)
    {
        super(parent, file);
        if (!file.getParent().equals(parent.getProject()))
            throw new IllegalArgumentException();
        if (!EXT.equals(file.getFileExtension()))
            throw new IllegalArgumentException();
    }

    @Override
    protected void buildStructure(SourceElementBody body,
        Map<IHandle, Body> newElements, Object ast, String source)
    {
        // empty for now
    }

    @Override
    protected Object createStructuralAst(String source) throws CoreException
    {
        // no AST for now
        return null;
    }

    @Override
    protected HandleManager getHandleManager()
    {
        return FooModelManager.INSTANCE.getHandleManager();
    }
}
```

We will get back to the `buildStructure` and `createStructuralAst` methods
of this class in a moment.

Now we can complete the implementation of the `FooProject` class:

```java
// FooProject.java

    @Override
    protected void buildStructure(Body body, Map<IHandle, Body> newElements)
        throws CoreException
    {
        IResource[] members = project.members();
        List<IFooFile> fooFiles = new ArrayList<IFooFile>(members.length);
        for (IResource member : members)
        {
            if (member instanceof IFile)
            {
                IFile file = (IFile)member;
                if (IFooFile.EXT.equals(file.getFileExtension()))
                {
                    IFooFile fooFile = new FooFile(this, file);
                    if (fooFile != null)
                        fooFiles.add(fooFile);
                }
            }
        }
        body.setChildren(fooFiles.toArray(new IHandle[fooFiles.size()]));
    }
```

As you may remember, we intended Foo project to be an *openable* element
responsible for building its structure. Here we set the currently existing
Foo files immediately contained within the project's resource as children
of the `FooProject` element.

Again, there is no need to use the `newElements` parameter here (recall
that it is designated for placing handle/body pairs for child elements),
because the `FooFile` will, in turn, be an openable element and will build
its own structure lazily, on demand.

Now that we have a complete implementation for the class `FooProject`,
we can test it. But let's first define some handy methods in `IFooProject`:

```java
// IFooProject.java

    /**
     * Returns the Foo file with the given name in this project, or 
     * <code>null</code> if unable to associate the given name 
     * with a Foo file. The name has to be a valid file name. 
     * This is a handle-only method. The Foo file may or may not exist.
     * 
     * @param name the name of the Foo file (not <code>null</code>)
     * @return the Foo file with the given name in this project, 
     *  or <code>null</code> if unable to associate the given name 
     *  with a Foo file 
     */
    IFooFile getFooFile(String name);

    /**
     * Returns the Foo files contained in this project.
     *
     * @return the Foo files contained in this project
     *  (never <code>null</code>)
     * @throws CoreException if this element does not exist or if
     *  an exception occurs while accessing its corresponding resource
     */
    IFooFile[] getFooFiles() throws CoreException;
```

We'll implement them as follows:

```java
// FooProject.java

    @Override
    public IFooFile getFooFile(String name)
    {
        int lastDot = name.lastIndexOf('.');
        if (lastDot < 0)
            return null;
        String fileExtension = name.substring(lastDot + 1);
        if (!IFooFile.EXT.equals(fileExtension))
            return null;
        return new FooFile(this, project.getFile(name));
    }

    @Override
    public IFooFile[] getFooFiles() throws CoreException
    {
        IHandle[] children = getChildren();
        int length = children.length;
        IFooFile[] result = new IFooFile[length];
        System.arraycopy(children, 0, result, 0, length);
        return result;
    }
```

For completeness, we will also add a method to `FooModelCore`:

```java
// FooModelCore.java

    /**
     * Returns the Foo file corresponding to the given file, 
     * or <code>null</code> if unable to associate the given file 
     * with a Foo file.
     *
     * @param file the given file (maybe <code>null</code>)
     * @return the Foo file corresponding to the given file, 
     *  or <code>null</code> if unable to associate the given file 
     *  with a Foo file
     */
    public static IFooFile create(IFile file)
    {
        if (file == null)
            return null;
        if (file.getParent().getType() != IResource.PROJECT)
            return null;
        IFooProject project = create(file.getProject());
        return project.getFooFile(file.getName());
    }
```

We can expand our test case now:

```java
// FooModelTest.java

    public void testFooModel() throws Exception
    {
        IFooProject[] fooProjects = fooModel.getFooProjects();
        assertEquals(1, fooProjects.length);
        IFooProject fooProject = fooProjects[0];
        assertEquals("Test001", fooProject.getName());
        // new code -->
        IFooFile[] fooFiles = fooProject.getFooFiles();
        assertEquals(1, fooFiles.length);
        IFooFile fooFile = fooFiles[0];
        assertEquals("test.foo", fooFile.getName());
        // <-- new code

        IFooProject fooProject2 = fooModel.getFooProject("Test002");
        assertFalse(fooProject2.exists());
        setUpProject("Test002");
        assertTrue(fooProject2.exists());
        fooProjects = fooModel.getFooProjects();
        assertEquals(2, fooProjects.length);
        assertTrue(Arrays.asList(fooProjects).contains(fooProject));
        assertTrue(Arrays.asList(fooProjects).contains(fooProject2));
        // new code -->
        IFooFile[] fooFiles2 = fooProject2.getFooFiles();
        assertEquals(1, fooFiles2.length);
        IFooFile fooFile2 = fooFiles2[0];
        assertEquals("test.foo", fooFile2.getName());
        // <-- new code
    }
```

You may remember that we use some test data in the form of a few projects
predefined in the `workspace` folder of the test fragment and materialized
into the runtime workspace via `setUpProject` calls.

Run the test case. Success! So far, so good.

Let's add some more tests:

```java
// FooModelTest.java

    public void testFooModel() throws Exception
    {
        // ...

        fooFile.getFile().delete(true, null);
        assertFalse(fooFile.exists());
        assertEquals(0, fooFile.getParent().getChildren().length);

        fooFile2.getFile().move(new Path("/Test001/test.foo"), true, null);
        assertFalse(fooFile2.exists());
        assertEquals(0, fooProject2.getFooFiles().length);
        assertEquals(1, fooProject.getFooFiles().length);
    }
```

Now, it fails:

```
junit.framework.AssertionFailedError: expected:<0> but was:<1>
	at org.eclipse.handly.internal.examples.basic.ui.model.FooModelTest.testFooModel(FooModelTest.java:65)
```

As you can see, we have deleted a Foo file and its `exists` method
does return `false` as expected, but somehow the deleted file still appears
among the children of its parent Foo project.

Feels like [[deja vu|Step-One#keeping-the-model-up-to-date]]?
Remember the `FooDeltaProcessor`? Let's make it process *file deltas* too:

```java
// FooDeltaProcessor.java

    @Override
    public boolean visit(IResourceDelta delta) throws CoreException
    {
        switch (delta.getResource().getType())
        {
        case IResource.ROOT:
            return processRoot(delta);

        case IResource.PROJECT:
            return processProject(delta);

        case IResource.FILE:
            return processFile(delta);

        default:
            return true;
        }
    }

    private boolean processFile(IResourceDelta delta)
    {
        switch (delta.getKind())
        {
        case IResourceDelta.ADDED:
            return processAddedFile(delta);

        case IResourceDelta.REMOVED:
            return processRemovedFile(delta);

        case IResourceDelta.CHANGED:
            return processChangedFile(delta);

        default:
            return false;
        }
    }

    private boolean processAddedFile(IResourceDelta delta)
    {
        IFile file = (IFile)delta.getResource();
        IFooFile fooFile = FooModelCore.create(file);
        if (fooFile != null)
            addToModel(fooFile);
        return false;
    }

    private boolean processRemovedFile(IResourceDelta delta)
    {
        IFile file = (IFile)delta.getResource();
        IFooFile fooFile = FooModelCore.create(file);
        if (fooFile != null)
            removeFromModel(fooFile);
        return false;
    }

    private boolean processChangedFile(IResourceDelta delta)
    {
        IFile file = (IFile)delta.getResource();
        IFooFile fooFile = FooModelCore.create(file);
        if (fooFile != null)
        {
            if ((delta.getFlags() &
                ~(IResourceDelta.MARKERS | IResourceDelta.SYNC)) != 0)
            {
                contentChanged(fooFile);
            }
        }
        return false;
    }

    private void contentChanged(IFooFile fooFile)
    {
        close(fooFile);
    }
```

In that way, if a Foo file has been added or removed, we update the list
of children in its parent's body accordingly; if a Foo file has changed,
we evict its (stale) body from the model cache (along with all descendants).

The test case will now pass.

## Building Source File Structure

Let's get back to the `FooFile` and its `buildStructure` and
`createStructuralAst` methods.

The `FooFile` is the last openable element in our model hierarchy.
Elements inside a source file are *never* openable because the
source file always builds all of its inner structure in one go
by parsing the text contents, as we shall soon see.

At the moment, we need to implement this couple of abstract methods
inherited from the `SourceFile`:

```java
protected abstract Object createStructuralAst(String source)
    throws CoreException;

protected abstract void buildStructure(SourceElementBody body,
    Map<IHandle, Body> newElements, Object ast, String source);
```

The method `createStructuralAst` returns a new Abstract Syntax Tree (AST)
object created from the given source string. The AST may contain just enough
information for computing the structure and properties of the source file
and its descendant elements. That's why it is called *structural*. Handly
treats the AST as an opaque `Object`-- it just calls `createStructuralAst`
whenever necessary and passes the result to `buildStructure`.

The method `buildStructure` should initialize the given `SourceElementBody`
based on the given AST and the given source string from which the AST was
created. All the descendant elements of the source file are to be placed
in the given `newElements` map as handle/body pairs.

The class `SourceElementBody` extends the class `Body` and implements the
interface `ISourceElementInfo`. It holds cached structure and properties for
a source element. Those structure and properties relate to a known snapshot
of the source file's contents. There are two predefined properties:

* the full text range of the source element
* the text range of the source element's identifier if there is one

Besides, source elements can define their own, specific properties to be
stored in a `SourceElementBody`.

That was a bit of theory behind these two methods. See the [System Overview]
(http://www.eclipse.org/downloads/download.php?file=/handly/docs/handly-overview.pdf&r=1)
for more information on the architecture, and the Javadocs for a detailed
description of the protocols.

In our case, the Foo language is based on Xtext, so parsing is
Xtext-specific:

```java
// FooFile.java

    /**
     * Returns a new <code>XtextResource</code> loaded from the given source
     * string. The resource is created in a new <code>ResourceSet</code>
     * obtained from the <code>IResourceSetProvider</code> corresponding to
     * this file. 
     * 
     * @return the new <code>XtextResource</code> loaded from the given
     *  source string (never <code>null</code>)
     * @throws CoreException if resource loading failed
     */
    @Override
    protected Object createStructuralAst(String source)
        throws CoreException
    {
        try
        {
            return parse(source, getFile().getCharset());
        }
        catch (IOException e)
        {
            throw new CoreException(Activator.createErrorStatus(
                e.getMessage(), e));
        }
    }

    /**
     * Returns a new <code>XtextResource</code> loaded from the given
     * contents. The resource is created in a new <code>ResourceSet</code>
     * obtained from the <code>IResourceSetProvider</code> corresponding
     * to this file. The resource's encoding is set to the given value.
     * This is a handle-only method.
     *
     * @param contents the contents to parse (not <code>null</code>)
     * @param encoding the encoding to be set for the created
     *  <code>XtextResource</code> (not <code>null</code>)
     * @return the new <code>XtextResource</code> loaded from
     *  the given contents (never <code>null</code>)
     * @throws IOException if resource loading failed 
     */
    protected XtextResource parse(String contents, String encoding)
        throws IOException
    {
        IResourceSetProvider resourceSetProvider =
            getResourceServiceProvider().get(IResourceSetProvider.class);
        ResourceSet resourceSet =
            resourceSetProvider.get(getFile().getProject());
        XtextResource resource =
            (XtextResource)resourceSet.createResource(getResourceUri());
        resource.load(new ByteArrayInputStream(contents.getBytes(encoding)),
            Collections.singletonMap(XtextResource.OPTION_ENCODING,
                encoding));
        return resource;
    }

    /**
     * Returns the <code>IResourceSetProvider</code> corresponding to
     * this file. This is a handle-only method.
     *
     * @return the <code>IResourceSetProvider</code> for this file 
     *  (never <code>null</code>)
     */
    protected IResourceServiceProvider getResourceServiceProvider()
    {
        IResourceServiceProvider provider =
            IResourceServiceProvider.Registry.INSTANCE.
                getResourceServiceProvider(getResourceUri());
        if (provider == null)
            throw new AssertionError();
        return provider;
    }

    /**
     * Returns the EMF resource URI for this file. 
     * This is a handle-only method.
     *
     * @return the resource URI for this file (never <code>null</code>)
     */
    protected URI getResourceUri()
    {
        return URI.createPlatformResourceURI(getPath().toString(), true);
    }
```

If you don't happen to know Xtext, you can safely ignore most of the
implementation details. They needn't concern us here; suffice to know
that `createStructuralAst` returns an EMF `Resource` that contains
an object graph representing the AST.

The method `buildStructure` walks through this object graph and
in the process creates handles for source elements, initializes
the corresponding bodies, and places the handle/body pairs
into the `newElements` map:

```java
// FooFile.java

    @Override
    protected void buildStructure(SourceElementBody body,
        Map<IHandle, Body> newElements, Object ast, String source)
    {
        XtextResource resource = (XtextResource)ast;
        IParseResult parseResult = resource.getParseResult();
        if (parseResult != null)
        {
            EObject root = parseResult.getRootASTElement();
            if (root instanceof Module)
            {
                FooFileStructureBuilder builder =
                    new FooFileStructureBuilder(newElements,
                        resource.getResourceServiceProvider());
                builder.buildStructure(this, body, (Module)root);
            }
        }
    }
```

As you can see, this method delegates all the hard work to the class
`FooFileStructureBuilder`, which extends the Handly-provided `StructureHelper`:

```java
// package org.eclipse.handly.internal.examples.basic.ui.model

/**
 * Builds the inner structure for a {@link FooFile}.
 */
class FooFileStructureBuilder
    extends StructureHelper
{
    private final ILocationInFileProvider locationProvider;

    /**
     * Constructs a new Foo file structure builder.
     * 
     * @param newElements the map to populate with stucture elements 
     *  (not <code>null</code>)
     * @param resourceServiceProvider Xtext {@link IResourceServiceProvider}
     *  for the language (not <code>null</code>)
     */
    FooFileStructureBuilder(Map<IHandle, Body> newElements,
        IResourceServiceProvider resourceServiceProvider)
    {
        super(newElements);
        if (resourceServiceProvider == null)
            throw new IllegalArgumentException();
        this.locationProvider =
            resourceServiceProvider.get(ILocationInFileProvider.class);
    }

    /**
     * Builds the structure for the given {@link FooFile} based on 
     * its {@link Module AST}.
     *
     * @param handle the handle to a Foo file (not <code>null</code>)
     * @param body the body of the Foo file (not <code>null</code>)
     * @param module the AST of the Foo file (not <code>null</code>)
     */
    void buildStructure(FooFile handle, SourceElementBody body,
        Module module)
    {
        for (Var var : module.getVars())
            buildStructure(handle, body, var);
        for (Def def : module.getDefs())
            buildStructure(handle, body, def);
        complete(body);
    }

    private void buildStructure(FooFile parent, Body parentBody, Var var)
    {
        if (var.getName() == null || var.getName().isEmpty())
            return;

        FooVar handle = new FooVar(parent, var.getName());
        SourceElementBody body = new SourceElementBody();
        body.setFullRange(getFullRange(var));
        body.setIdentifyingRange(getIdentifyingRange(var));
        addChild(parentBody, handle, body);
        complete(body);
    }

    private void buildStructure(FooFile parent, Body parentBody, Def def)
    {
        if (def.getName() == null || def.getName().isEmpty())
            return;

        int arity = def.getParams().size();
        FooDef handle = new FooDef(parent, def.getName(), arity);
        SourceElementBody body = new SourceElementBody();
        body.setFullRange(getFullRange(def));
        body.setIdentifyingRange(getIdentifyingRange(def));
        body.set(FooDef.PARAMETER_NAMES, // store a custom property
            def.getParams().toArray(new String[arity]));
        addChild(parentBody, handle, body);
        complete(body);
    }

    private TextRange getFullRange(EObject eObject)
    {
        return toTextRange(locationProvider.getFullTextRegion(eObject));
    }

    private TextRange getIdentifyingRange(EObject eObject)
    {
        return toTextRange(locationProvider.getSignificantTextRegion(
            eObject));
    }

    private static TextRange toTextRange(ITextRegion region)
    {
        if (region == null || region.equals(ITextRegion.EMPTY_REGION))
            return null;
        else
            return new TextRange(region.getOffset(), region.getLength());
    }
}
```

The base class `StructureHelper` provides a couple of handy methods for
building the structure of a model element: `addChild` remembers the given
element as a child of the given parent body and puts the element together
with the given body into the `newElements` map (resolving duplicates
along the way), while `complete` completes the given body by setting
the elements previously remembered by `addChild` as the body's children.

Again, even if you can't fully comprehend Xtext-specific details, you can
well grasp the essence: we just walk the AST and in the process compute and
place into the `newElements` a handle/body pair for each structural element
inside the source file.

Note how the `FooFileStructureBuilder` uses a custom property for storing
the parameter names of a Foo function in the element's `SourceElementBody`.
Here is the property declaration, along with a convenience method
for obtaining its value:

```java
// IFooDef.java

    /**
     * Parameter names property.
     * @see #getParameterNames()
     */
    Property<String[]> PARAMETER_NAMES = new Property<String[]>(
        "parameterNames");

    /**
     * Returns the names of parameters in this function.
     * Returns an empty array if this function has no parameters.
     *
     * @return the names of parameters in this function; an empty array 
     *  if this function has no parameters (never <code>null</code>)
     * @throws CoreException if this element does not exist or if an
     *  exception occurs while accessing its corresponding resource
     */
    String[] getParameterNames() throws CoreException;
```

The method just delegates to a generic accessor in `ISourceElementInfo`:

```java
// FooDef.java

    @Override
    public String[] getParameterNames() throws CoreException
    {
        return getSourceElementInfo().get(PARAMETER_NAMES);
    }
```

We're almost done, except for some additional getter methods in the `IFooFile`:

```java
// IFooFile.java

    /**
     * Returns the variable with the given name declared in this Foo file.
     * This is a handle-only method. The variable may or may not exist.
     *
     * @param name the name of the requested variable in the Foo file
     * @return a handle onto the corresponding variable
     *  (never <code>null</code>). The variable may or may not exist.
     */
    IFooVar getVar(String name);
    
    /**
     * Returns the variables declared in this Foo file in the order in which
     * they appear in the source.
     *
     * @return the variables declared in this Foo file
     *  (never <code>null</code>)
     * @throws CoreException if this element does not exist or if
     *  an exception occurs while accessing its corresponding resource
     */
    IFooVar[] getVars() throws CoreException;
    
    /**
     * Returns the function with the given name and the given arity defined
     * in this Foo file. This is a handle-only method. The function may or
     * may not exist.
     *
     * @param name the name of the requested function in the Foo file
     * @param arity the arity of the requested function in the Foo file
     * @return a handle onto the corresponding function
     *  (never <code>null</code>). The function may or may not exist.
     */
    IFooDef getDef(String name, int arity);
    
    /**
     * Returns the functions defined in this Foo file in the order in which
     * they appear in the source.
     *
     * @return the functions defined in this Foo file
     *  (never <code>null</code>)
     * @throws CoreException if this element does not exist or if
     *  an exception occurs while accessing its corresponding resource
     */
    IFooDef[] getDefs() throws CoreException;
```

which are implemented as follows:

```java
// FooFile.java

    @Override
    public IFooVar getVar(String name)
    {
        return new FooVar(this, name);
    }

    @Override
    public IFooVar[] getVars() throws CoreException
    {
        return getChildren(IFooVar.class);
    }

    @Override
    public IFooDef getDef(String name, int arity)
    {
        return new FooDef(this, name, arity);
    }

    @Override
    public IFooDef[] getDefs() throws CoreException
    {
        return getChildren(IFooDef.class);
    }
```

The inherited `getChildren(Class)` method returns the immediate children
of the element that have the given type.

Okay, that's enough for now. Let's test the new functionality:

```java
// package org.eclipse.handly.internal.examples.basic.ui.model

/**
 * <code>FooFile</code> tests.
 */
public class FooFileTest
    extends WorkspaceTestCase
{
    private IFooFile fooFile;

    @Override
    protected void setUp() throws Exception
    {
        super.setUp();
        IFooProject fooProject = FooModelCore.create(
            setUpProject("Test002"));
        fooFile = fooProject.getFooFile("test.foo");
    }

    public void testFooFile() throws Exception
    {
        assertTrue(fooFile.exists());

        assertEquals(5, fooFile.getChildren().length);

        IFooVar[] vars = fooFile.getVars();
        assertEquals(2, vars.length);
        assertEquals(fooFile.getVar("x"), vars[0]);
        assertEquals(fooFile.getVar("y"), vars[1]);

        IFooDef[] defs = fooFile.getDefs();
        assertEquals(3, defs.length);
        assertEquals(fooFile.getDef("f", 0), defs[0]);
        assertEquals(fooFile.getDef("f", 1), defs[1]);
        assertEquals(fooFile.getDef("f", 2), defs[2]);
        
        assertEquals(0, defs[0].getParameterNames().length);
        
        String[] parameterNames = defs[1].getParameterNames();
        assertEquals(1, parameterNames.length);
        assertEquals("x", parameterNames[0]);
        
        parameterNames = defs[2].getParameterNames();
        assertEquals(2, parameterNames.length);
        assertEquals("x", parameterNames[0]);
        assertEquals("y", parameterNames[1]);
    }
}
```

Run it as a JUnit Plug-in Test. You can use the launch configuration 
predefined in the test fragment.

Amazingly, it works! But how *fast*?

Let's run the body of the `testFooFile` method 100,000 times in a loop:

```java
// FooFileTest.java

    public void testFooFile() throws Exception
    {
        for (int i = 0; i < 100000; i++)
        {
            // ...
        }
    }
```

It might seem it would never return! (Okay, it took about 5 minutes
on our machine...) Do you feel it could do better than this?

If you set a breakpoint in the `FooFile.createStructuralAst` method and
rerun the `FooFileTest` under debugger, you will see that every request
for the Foo file's body leads to rebuilding the whole structure
of the source file and hence, to re-parsing. It is called six (!) times
within each iteration of the loop -- once for each non-handle-only
method call.

Hopefully, you already know how to deal with it. That's what
the `FooModelCache` is for. The reason for constant re-parsing
is simply that we don't cache source element bodies yet.

Let's fix it:

```java
class FooModelCache
    implements IBodyCache
{
    private static final int DEFAULT_PROJECT_SIZE = 5;
    // new code -->
    private static final int DEFAULT_FILE_SIZE = 100;
    private static final int DEFAULT_CHILDREN_SIZE = DEFAULT_FILE_SIZE * 20;
        // average 20 children per file
    // <-- new code

    private Body modelBody; // Foo model element's body
    private HashMap<IHandle, Body> projectCache; // Foo projects
    // new code -->
    private ElementCache fileCache; // Foo files
    private HashMap<IHandle, Body> childrenCache; // children of Foo files
    // <-- new code

    public FooModelCache()
    {
        projectCache = new HashMap<IHandle, Body>(DEFAULT_PROJECT_SIZE);
        // new code -->
        fileCache = new ElementCache(DEFAULT_FILE_SIZE);
        childrenCache = new HashMap<IHandle, Body>(DEFAULT_CHILDREN_SIZE);
        // <-- new code
    }

    @Override
    public Body get(IHandle handle)
    {
        if (handle instanceof IFooModel)
            return modelBody;
        else if (handle instanceof IFooProject)
            return projectCache.get(handle);
        // new code -->
        else if (handle instanceof IFooFile)
            return fileCache.get(handle);
        else
            return childrenCache.get(handle);
        // <-- new code
    }

    @Override
    public Body peek(IHandle handle)
    {
        if (handle instanceof IFooModel)
            return modelBody;
        else if (handle instanceof IFooProject)
            return projectCache.get(handle);
        // new code -->
        else if (handle instanceof IFooFile)
            return fileCache.peek(handle);
        else
            return childrenCache.get(handle);
        // <-- new code
    }

    @Override
    public void put(IHandle handle, Body body)
    {
        if (handle instanceof IFooModel)
            modelBody = body;
        else if (handle instanceof IFooProject)
        {
            projectCache.put(handle, body);
            // new code -->
            fileCache.ensureSpaceLimit(body, handle);
            // <-- new code
        }
        // new code -->
        else if (handle instanceof IFooFile)
            fileCache.put(handle, body);
        else
            childrenCache.put(handle, body);
        // <-- new code
    }

    @Override
    public void remove(IHandle handle)
    {
        if (handle instanceof IFooModel)
            modelBody = null;
        else if (handle instanceof IFooProject)
        {
            projectCache.remove(handle);
            // new code -->
            fileCache.resetSpaceLimit(DEFAULT_FILE_SIZE, handle);
            // <-- new code
        }
        // new code -->
        else if (handle instanceof IFooFile)
            fileCache.remove(handle);
        else
            childrenCache.remove(handle);
        // <-- new code
    }
}
```

Note that an `ElementCache` is used for `FooFile`s themselves, whereas
their child elements are stored in a simple `HashMap`.

The `ElementCache` is quite interesting in that it is an *overflowing
LRU cache*. It attempts to maintain a size equal or less than its space limit
by removing the least recently used elements. The cache will remove elements
which successfully close and all elements which are explicitly removed.
If the cache cannot remove enough old elements to add new elements, it will
grow beyond its space limit. Later, it will attempt to shrink back to the
space limit. This cache permits to put a constraint on the amount of memory
consumed by the model.

We don't need to use an LRU cache for child elements of Foo files (a `HashMap`
is sufficient) because these elements always come and go together with
their parent `FooFile` element.

If you rerun this simple performance test now, it will take dramatically
less time (about 2 seconds on our machine). *2 seconds vs 5 minutes*.
You might call it a difference!

## Closing Step Two

In this step we dived deeper into Handly and built a complete *code model*
for our simple programming language, touching along the way most of the
areas you will encounter as you implement your own Handly-based models.
We also wrote some tests to make sure our model works correctly
and with good performance.

We will put this model to use and add even more functionality expected
of a code model in our next step: [[Viewing the Model|Step Three]].