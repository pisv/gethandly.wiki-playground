# Step Two: The Rest of the Model

[[Step One]] has left us with the basics of a model. We have got
a two-level hierarchy of `FooModel` containing `FooProject`(s)
and some resource delta processing for keeping the model up-to-date.
We will use it as a starting point for this step, where we will build
a complete *code model* for the Foo language, from the workspace root level
down to structural elements inside source files. The complete source code
for this step of the running example is available in the
[Step Two repository](https://github.com/pisv/gethandly.2nd).

As usual, we begin by defining the interfaces for the new model elements.
We will make these interfaces extend the relevant 'extension interfaces' such as
`IElementExtension`, `ISourceElementExension`, and `ISourceFileExtension`
to introduce a number of generally useful default methods for convenience,
but could as well define the model element interfaces entirely from scratch.

**Note:** In any case, it is usually a good idea for the model API to extend
the relevant common interfaces for model elements, such as `IElement`,
`ISourceElement`, `ISourceFile`, and `ISourceConstruct` (the 'extension
interfaces' we are extending already extend the corresponding common
interfaces). This will make the model easier to use with APIs expressed
in terms of the common element interfaces. (An important API expressed
in terms of the common interfaces is the class `Elements` that provides
static methods for generic access to elements of any Handly-based model).
Otherwise, explicit casts might be necessary when interacting with such APIs.
Since each of the common interfaces for model elements is just a marker interface
and contains no members, there is generally no drawback in extending them.

```java
// package org.eclipse.handly.examples.basic.ui.model

/**
 * Represents a Foo source file.
 */
public interface IFooFile
    extends ISourceFileExtension, ISourceElementExtension, IElementExtension
{
    /**
     * Foo filename extension.
     */
    String EXT = "foo";

    @Override
    default IFooProject getParent()
    {
        return (IFooProject)IElementExtension.super.getParent();
    }
}

/**
 * Represents a variable declared in a Foo file.
 */
public interface IFooVar
    extends ISourceConstruct, ISourceElementExtension, IElementExtension
{
    @Override
    default IFooFile getParent()
    {
        return (IFooFile)IElementExtension.super.getParent();
    }
}

/**
 * Represents a function defined in a Foo file.
 */
public interface IFooDef
    extends ISourceConstruct, ISourceElementExtension, IElementExtension
{
    @Override
    default IFooFile getParent()
    {
        return (IFooFile)IElementExtension.super.getParent();
    }
}
```

The corresponding implementation classes are:

```java
// package org.eclipse.handly.internal.examples.basic.ui.model

public class FooFile
    extends WorkspaceSourceFile
    implements IFooFile, IFooElementInternal
{
}

public class FooVar
    extends SourceConstruct
    implements IFooVar, IFooElementInternal
{
}

public class FooDef
    extends SourceConstruct
    implements IFooDef, IFooElementInternal
{
}
```

The classes `SourceFile` and `SourceConstruct` provide a useful base
implementation for representing source files and their structural elements
in a Handly-based model. Our model will only support source files
residing in the Eclipse workspace, so it is more convenient for us to extend
`WorkspaceSourceFile` rather than its resource-agnostic superclass,
`SourceFile`. (Since version 1.3, Handly also provides out-of-the-box support
for source files outside the Eclipse workspace that have an underlying
`IFileStore`. See `BaseSourceFile` and `FsSourceFile`).

Once again, we need to complete the implementation by defining the appropriate
constructors and overriding the inherited abstract methods. Let's begin
with the class `FooVar`:

```java
/**
 * Represents a variable declared in a Foo file.
 */
public class FooVar
    extends SourceConstruct
    implements IFooVar, IFooElementInternal
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
    implements IFooDef, IFooElementInternal
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
}
```

As you can see, this model element is special in that its constructor takes
an additional parameter, `arity`. While variables can be identified solely by
their name within the parent Foo file, functions are a different story.
The name alone is not sufficient to identify a Foo function -- we also need
to state the number of arguments (its *arity*).

Therefore, we need to make `arity` part of the state of a `FooDef` instance,
the handle for a Foo function (as you may remember, handles are value objects
that hold immutable, 'key' information about a model element). We also need
to extend the `equals` and `hashCode` methods to take `arity` into account
(until now the inherited implementation of `equals` and `hashCode` was
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
    extends WorkspaceSourceFile
    implements IFooFile, IFooElementInternal
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
    public void buildSourceStructure_(IContext context,
        IProgressMonitor monitor)
    {
        // empty for now
    }
}
```

We will get back to the `buildSourceStructure_` method in a moment.

Now we can complete the implementation of the `FooProject` class:

```java
// FooProject.java

    @Override
    public void buildStructure_(IContext context,
        IProgressMonitor monitor) throws CoreException
    {
        IResource[] members = project.members();
        List<IFooFile> fooFiles = new ArrayList<>(members.length);
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
        Body body = new Body();
        body.setChildren(fooFiles.toArray(Elements.EMPTY_ARRAY));
        context.get(NEW_ELEMENTS).put(this, body);
    }
```

As you may remember, we intended Foo project to be an *openable* element
responsible for building its structure. Here we set the currently existing
Foo files immediately contained within the project's resource as children
of the created body for the `FooProject` element.

Again, there is no need to do anything else here, because each `FooFile` will,
in turn, be an openable element and will build its own structure on demand
in its `buildSourceStructure_` method.

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
        IElement[] children = getChildren();
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

Recall that we use some test data in the form of a few projects predefined
in the `workspace` folder of the test fragment and materialized into the
runtime workspace via `setUpProject` calls.

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
	at org.eclipse.handly.internal.examples.basic.ui.model.FooModelTest.testFooModel
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

        // new code -->
        case IResource.FILE:
            return processFile(delta);
        // <-- new code

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

Let's get back to the `FooFile` and its `buildSourceStructure_` method.

The `FooFile` is the innermost openable element in our model.
Elements inside a source file are *never* openable because the
source file always builds all of its inner structure in one go
by parsing the text contents, as we'll soon see.

The method `buildSourceStructure_` of a source file is supposed to create and
initialize a body for the source file itself and also for each of its descendant
elements, and place the initialized bodies into the `NEW_ELEMENTS` map in the
given context.

The class `SourceElementBody` extends the class `Body` and implements the
interface `ISourceElementInfo`. It holds cached structure and properties for
a source element. Those structure and properties correspond to a snapshot
of the source file's contents. There are two predefined properties:

* the full text range of the source element
* the text range of the source element's identifier (if there is one)

Also, source elements can define their own, specific properties to be stored
in a `SourceElementBody`, as will be shown in a moment.

The bodies need to be initialized based on the `SOURCE_AST` or the
`SOURCE_STRING` provided in the given context. In general, either or both
may be present (in the latter case the `SOURCE_AST` is guaranteed to be created
from the `SOURCE_STRING`).

The reason for such a contract is that the `buildSourceStructure_` method may
be invoked from different framework layers and, depending on a framework
configuration, the AST may be already available and can efficiently be reused.
However, Handly is flexible enough to allow for framework configurations where
even something like SAX can be used, skipping the AST creation altogether.

Thus, in general, the `buildSourceStructure_` method should check for the
`SOURCE_AST` first and, if it is not available, parse the `SOURCE_STRING`
using whatever means it sees fit. Actually, it is not as complicated as it
may sound.

The `SOURCE_AST` is typed as `Object`, since the core framework knows nothing
about a specific representation of the AST; it can just propagate the AST
from an upper layer. The `SOURCE_STRING` is, well, a `String`.

In our case, the Foo language is based on Xtext, so the `SOURCE_AST` would
actually be represented by `XtextResource` and parsing of the `SOURCE_STRING`
would be Xtext-specific. Here's the relevant code:

```java
// FooFile.java

    @Override
    public void buildSourceStructure_(IContext context,
        IProgressMonitor monitor) throws CoreException
    {
        Map<IElement, Object> newElements = context.get(NEW_ELEMENTS);
        SourceElementBody body = new SourceElementBody();

        XtextResource resource = (XtextResource)context.get(SOURCE_AST);
        if (resource == null)
        {
            try
            {
                resource = parse(context.get(SOURCE_CONTENTS),
                    getFile().getCharset());
            }
            catch (IOException e)
            {
                throw new CoreException(Activator.createErrorStatus(
                    e.getMessage(), e));
            }
        }

        IParseResult parseResult = resource.getParseResult();
        if (parseResult != null)
        {
            EObject root = parseResult.getRootASTElement();
            if (root instanceof Module)
            {
                FooFileStructureBuilder builder = new FooFileStructureBuilder(
                    newElements, resource.getResourceServiceProvider());
                builder.buildStructure(this, body, (Module)root);
            }
        }

        newElements.put(this, body);
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
        return URI.createPlatformResourceURI(getFile_().getFullPath().toString(),
            true);
    }
```

If you don't happen to know Xtext, you can safely ignore most of the
implementation details. The `XtextResource` contains an object graph
representing the AST. The method `buildSourceStructure_` walks through
this object graph and in the process creates handles for source elements,
initializes the corresponding bodies, and places the handle/body pairs
into the `NEW_ELEMENTS` map. It delegates all the hard work to the class
`FooFileStructureBuilder`, which uses the Handly-provided `StructureHelper`
internally:

```java
// package org.eclipse.handly.internal.examples.basic.ui.model

/**
 * Builds the inner structure for a {@link FooFile}.
 */
class FooFileStructureBuilder
{
    private final Map<IElement, Object> newElements;
    private final ILocationInFileProvider locationProvider;
    private final StructureHelper helper = new StructureHelper();

    /**
     * Constructs a new Foo file structure builder.
     *
     * @param newElements the map to populate with structure elements
     *  (not <code>null</code>)
     * @param resourceServiceProvider Xtext's {@link IResourceServiceProvider}
     *  for the language (not <code>null</code>)
     */
    FooFileStructureBuilder(Map<IElement, Object> newElements,
        IResourceServiceProvider resourceServiceProvider)
    {
        if (newElements == null)
            throw new IllegalArgumentException();
        this.newElements = newElements;
        if (resourceServiceProvider == null)
            throw new IllegalArgumentException();
        locationProvider = resourceServiceProvider.get(
            ILocationInFileProvider.class);
    }

    /**
     * Builds the structure for the given {@link FooFile} based on
     * its {@link Module AST}.
     *
     * @param handle the handle to a Foo file (not <code>null</code>)
     * @param body the body of the Foo file (not <code>null</code>)
     * @param module the AST of the Foo file (not <code>null</code>)
     */
    void buildStructure(FooFile handle, SourceElementBody body, Module module)
    {
        for (Var var : module.getVars())
            buildStructure(handle, body, var);
        for (Def def : module.getDefs())
            buildStructure(handle, body, def);
        body.setChildren(helper.popChildren(body).toArray(
            Elements.EMPTY_ARRAY));
    }

    private void buildStructure(FooFile parent, Body parentBody, Var var)
    {
        if (var.getName() == null || var.getName().isEmpty())
            return;

        FooVar handle = new FooVar(parent, var.getName());
        helper.resolveDuplicates(handle);
        SourceElementBody body = new SourceElementBody();
        body.setFullRange(getFullRange(var));
        body.setIdentifyingRange(getIdentifyingRange(var));
        newElements.put(handle, body);
        helper.pushChild(parentBody, handle);
    }

    private void buildStructure(FooFile parent, Body parentBody, Def def)
    {
        if (def.getName() == null || def.getName().isEmpty())
            return;

        int arity = def.getParams().size();
        FooDef handle = new FooDef(parent, def.getName(), arity);
        helper.resolveDuplicates(handle);
        SourceElementBody body = new SourceElementBody();
        body.setFullRange(getFullRange(def));
        body.setIdentifyingRange(getIdentifyingRange(def));
        body.set(FooDef.PARAMETER_NAMES, def.getParams().toArray(
            new String[arity]));
        newElements.put(handle, body);
        helper.pushChild(parentBody, handle);
    }

    private TextRange getFullRange(EObject eObject)
    {
        return toTextRange(locationProvider.getFullTextRegion(eObject));
    }

    private TextRange getIdentifyingRange(EObject eObject)
    {
        return toTextRange(locationProvider.getSignificantTextRegion(eObject));
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

The class `StructureHelper` provides helper methods for building the structure
of a model element: `pushChild` remembers the given element as a child of the
given parent body, `popChildren` retrieves and forgets the child elements
previously remembered for the given body, and `resolveDuplicates` resolves
duplicate source constructs by incrementing their occurrence count.

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
    Property<String[]> PARAMETER_NAMES = Property.get("parameterNames",
        String[].class);

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

And last but not least, we need to update the cache implementation:

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

    private Object modelBody; // Foo model element's body
    private Map<IElement, Object> projectCache; // Foo projects
    // new code -->
    private ElementCache fileCache; // Foo files
    private Map<IElement, Object> childrenCache; // children of Foo files
    // <-- new code

    public FooModelCache()
    {
        projectCache = new HashMap<>(DEFAULT_PROJECT_SIZE);
        // new code -->
        fileCache = new ElementCache(DEFAULT_FILE_SIZE);
        childrenCache = new HashMap<>(DEFAULT_CHILDREN_SIZE);
        // <-- new code
    }

    @Override
    public Object get(IElement element)
    {
        if (element instanceof IFooModel)
            return modelBody;
        else if (element instanceof IFooProject)
            return projectCache.get(element);
        // new code -->
        else if (element instanceof IFooFile)
            return fileCache.get(element);
        else
            return childrenCache.get(element);
        // <-- new code
    }

    @Override
    public Object peek(IElement element)
    {
        if (element instanceof IFooModel)
            return modelBody;
        else if (element instanceof IFooProject)
            return projectCache.get(element);
        // new code -->
        else if (element instanceof IFooFile)
            return fileCache.peek(element);
        else
            return childrenCache.get(element);
        // <-- new code
    }

    @Override
    public void put(IElement element, Object body)
    {
        if (element instanceof IFooModel)
            modelBody = body;
        else if (element instanceof IFooProject)
        {
            projectCache.put(element, body);
            // new code -->
            fileCache.ensureMaxSize(((Body)body).getChildren().length, element);
            // <-- new code
        }
        // new code -->
        else if (element instanceof IFooFile)
            fileCache.put(element, body);
        else
            childrenCache.put(element, body);
        // <-- new code
    }

    @Override
    public void remove(IElement element)
    {
        if (element instanceof IFooModel)
            modelBody = null;
        else if (element instanceof IFooProject)
        {
            projectCache.remove(element);
            // new code -->
            fileCache.resetMaxSize(DEFAULT_FILE_SIZE, element);
            // <-- new code
        }
        // new code -->
        else if (element instanceof IFooFile)
            fileCache.remove(element);
        else
            childrenCache.remove(element);
        // <-- new code
    }
}
```

Note that an `ElementCache` is used for `FooFile`s themselves, whereas
their child elements are stored in a simple `HashMap`.

The `ElementCache` is quite interesting in that it is an *overflowing
LRU cache*. It attempts to maintain a size equal or less than its max size
by removing the least recently used elements. The cache will remove elements
that successfully close. If the cache cannot remove enough old elements to add
new elements, it will grow beyond its max size. Later, it will attempt to shrink
back to the max size. Such a cache allows you to put a constraint on the amount
of memory consumed by the model.

We don't need to use an LRU cache for child elements of Foo files (a `HashMap`
is sufficient) because these elements always come and go together with
their parent `FooFile` element.

Let's test the new functionality:

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

## Closing Step Two

In this step we dived deeper into Handly and built a complete *code model*
for our simple programming language, touching along the way most of the
areas you will encounter as you implement your own Handly-based models.
We also wrote some tests to make sure our model works correctly
and with good performance.

We will put this model to use and add even more functionality expected
of a code model in our next step: [[Viewing the Model|Step Three]].
