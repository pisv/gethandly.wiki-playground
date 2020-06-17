# Step One: Basic Model

In this step we will take you through the entire development process
of a Handly-based model, but with minimal functionality. We will
implement a basic model with only two levels: `FooModel` containing `FooProject`(s).
While simple, this will be a complete, fully functional implementation.
It will demonstrate the basic structuring and behavior of a Handly-based model
and give you a taste of what it is all about. It will also serve as a
starting point for our next step. The complete source code for this step
of the running example is available in the
[Step One repository](https://github.com/pisv/gethandly.1st).

We begin by defining the interfaces for Foo model elements
in the package:

`org.eclipse.handly.examples.basic.ui.model`

of the

`org.eclipse.handly.examples.basic.ui`

bundle:

```java
public interface IFooModel
{
}

public interface IFooProject
{
}
```

There are no methods yet, just the blank interface declarations.

Let's define the corresponding implementation classes in the package:

`org.eclipse.handly.internal.examples.basic.ui.model`

of the same bundle:

```java
public class FooModel
    extends Element
    implements IFooModel, IModelImpl
{
}

public class FooProject
    extends Element
    implements IFooProject
{
}
```

As you can see, we extend the class `Element`, a useful base implementation
provided by Handly for handle-based model elements.

Also, we make our root element, represented by the class `FooModel`,
implement `IModelImpl`, indicating that the root element will also serve
as the *model object* in our model. The model object is the common owner
of all elements of a model. In general, there is no requirement for the
model object itself to be an element, but in this case we merge it with
the root element for convenience.

At the moment, the code doesn't yet compile. We need to fill in the blanks
and complete the implementation by providing the appropriate constructors
and overriding the inherited abstract methods.

Let's begin with the class `FooModel`:

```java
/**
 * Represents the root Foo element corresponding to the workspace.
 */
public class FooModel
    extends Element
    implements IFooModel, IModelImpl
{
    private final IWorkspace workspace;

    /**
     * Constructs a handle for the root Foo element.
     */
    public FooModel()
    {
        super(null /*no parent*/, null /*no name*/);
        this.workspace = ResourcesPlugin.getWorkspace();
    }

    @Override
    public int getModelApiLevel_()
    {
        return ApiLevel.CURRENT;
    }

    @Override
    public IContext getModelContext_()
    {
        return Contexts.EMPTY_CONTEXT;
    }

    @Override
    public IResource getResource_()
    {
        return workspace.getRoot();
    }

    @Override
    public void validateExistence_(IContext context) throws CoreException
    {
        // always exists
    }
}
```

Every model element (a handle) is a value object, so it must be immutable and
must override equals() and hashCode() appropriately. The basic implementation
of `equals()` and `hashCode()` provided in the class `Element` is sufficient in
many cases -- it uses the element's name and parent as the basis for computation.

The model object is a bit specific in that it must implement the `getModelApiLevel_`
method, which returns the Handly *API level* supported by the model. By checking
the API level, generic code can dynamically discover which API methods are
supported by the model it is dealing with. It is recommended to always return
`ApiLevel.CURRENT`, which corresponds to the version of Handly the model was
built against.

The model object must also implement the `getModelContext_` method.
A context, represented in Handly by the `IContext` interface, supplies
values (data objects or services) associated with keys. Two kinds of keys
are supported: `org.eclipse.handly.util.Property` and `java.lang.Class`,
both of which allow the client to specify the type of requested value,
which makes contexts completely type safe. The *model context* provides
information and services pertaining to the model. We currently have none,
so we just return `EMPTY_CONTEXT`.

Elements that have an underlying resource should return it from the
`getResource_()` method. In this case, the workspace root is returned.

Handles can refer to non-existing elements. The `validateExistence_` method
should throw a `CoreException` if the element may not begin existence in the
model (for example, if its underlying resource does not exist). In this case,
this method does nothing as the root element always exists.

As you can see, the methods inherited from the class `Element` use a naming
convention based on the `_` suffix. This effectively separates methods
defined by the framework from model-specific methods defined by the model
implementor and significantly reduces the possibility of a method conflict
in the inheritance hierarchy.

```java
/**
 * Represents a Foo project.
 */
public class FooProject
    extends Element
    implements IFooProject
{
    private final IProject project;

    /**
     * Constructs a handle for a Foo project with the given parent element
     * and the given underlying workspace project.
     *
     * @param parent the parent of the element (not <code>null</code>)
     * @param project the workspace project underlying the element
     *  (not <code>null</code>)
     */
    public FooProject(FooModel parent, IProject project)
    {
        super(parent, project.getName());
        if (parent == null)
            throw new IllegalArgumentException();
        this.project = project;
    }

    @Override
    public IResource getResource_()
    {
        return project;
    }

    @Override
    public void validateExistence_(IContext context) throws CoreException
    {
        if (!project.hasNature(NATURE_ID))
            throw newDoesNotExistException_();
    }
}
```

For the Foo project, `validateExistence_` throws a `CoreException` when the
underlying project resource is not accessible or doesn't have the Foo nature.

If you need a reference on what exactly a project nature is, the Eclipse Corner
article [Project Builders and Natures](https://www.eclipse.org/articles/Article-Builders/builders.html)
authored by John Arthorne is a great resource. The project nature is contributed
via an extension point:

```xml
<!-- plugin.xml -->

   <extension
         id="fooNature"
         name="Foo Project Nature"
         point="org.eclipse.core.resources.natures">
      <runtime>
         <run
               class=
"org.eclipse.handly.internal.examples.basic.ui.model.FooProjectNature">
         </run>
      </runtime>
      <requires-nature
            id="org.eclipse.xtext.ui.shared.xtextNature">
      </requires-nature>
   </extension>
```

with its runtime behavior provided by a class implementing `IProjectNature`:

```java
// package org.eclipse.handly.internal.examples.basic.ui.model

/**
 * Foo project nature.
 */
public class FooProjectNature
    implements IProjectNature
{
    /**
     * Foo nature id.
     */
    public static final String ID = Activator.PLUGIN_ID + ".fooNature";

    private IProject project;

    @Override
    public void configure() throws CoreException
    {
    }

    @Override
    public void deconfigure() throws CoreException
    {
    }

    @Override
    public IProject getProject()
    {
        return project;
    }

    @Override
    public void setProject(IProject project)
    {
        this.project = project;
    }
}
```

The Foo nature doesn't configure any specific builders; the Xtext builder
will be installed by the required Xtext nature.

We still need to implement a couple of remaining abstract methods for our
model elements: `buildStructure_` and `getModelManager_`. These methods are
central to implementation of the *handle/body idiom* in Handly.

The basic idea is that mutable structure and properties of a model element
are stored separately in an internal 'body', while the handle holds immutable,
'key' information about the element (recall that handles are value objects).

The method `buildStructure_` is responsible for creating a body for the element
and initializing it based on the element's current contents. For the `FooModel`,
we set the currently open Foo projects as the children of the created body:

```java
// FooModel.java

    @Override
    public void buildStructure_(IContext context, IProgressMonitor monitor)
        throws CoreException
    {
        IProject[] projects = workspace.getRoot().getProjects();
        List<IFooProject> fooProjects = new ArrayList<>(projects.length);
        for (IProject project : projects)
        {
            if (project.isOpen() && project.hasNature(IFooProject.NATURE_ID))
            {
                fooProjects.add(new FooProject(this, project));
            }
        }
        Body body = new Body();
        body.setChildren(fooProjects.toArray(Elements.EMPTY_ARRAY));
        context.get(NEW_ELEMENTS).put(this, body);
    }
```

As you can see, the `buildStructure_` method has an *operation context*
as one its parameters. We are using the context-provided `NEW_ELEMENTS` map
to associate the created body with the element, as required by the method's
contract.

In general, any `Object` can be used as the body of an `Element`. Usually,
an instance of the class `Body` or its subclass is used. Otherwise, we would
also need to override the `getChildrenFromBody_(Object)` method, which returns
the element's children stored in the given body.

Note that we only create and intitialize a body for the `FooModel` itself
rather than let the `FooModel` build the whole structure for its child
projects and put it (as handle/body pairs) into the `NEW_ELEMENTS` map.
Each `FooProject` will be responsible for building its own structure.

The `FooModel` and `FooProject` elements are said to be *openable* because
they know how to open themselves (build their structure and properties)
when asked to do so. In that way, the model is populated with its elements
lazily, on demand. In contrast, elements inside a source file are *never*
openable because the source file builds all of its inner structure in one go
by parsing the text contents, as we'll see in the next step.

The Handly-provided class `ElementManager` manages handle/body relationships
for a handle-based model. Generally, each model will have its own instance
of the `ElementManager` that must be shared between all elements of the model.

Let's define a special manager class for the Foo model that will hold
the single instance of the `IFooModel` and the single instance of the
`ElementManager` for the model:

```java
// package org.eclipse.handly.internal.examples.basic.ui.model

/**
 * The manager for the Foo Model.
 *
 * @threadsafe This class is intended to be thread-safe
 */
public class FooModelManager
    implements IModelManager
{
    /**
     * The sole instance of the manager.
     */
    public static final FooModelManager INSTANCE = new FooModelManager();

    private FooModel fooModel;
    private ElementManager elementManager;

    public void startup() throws Exception
    {
        fooModel = new FooModel();
        elementManager = new ElementManager(new FooModelCache());
    }

    public void shutdown() throws Exception
    {
        elementManager = null;
        fooModel = null;
    }

    @Override
    public FooModel getModel()
    {
        if (fooModel == null)
            throw new IllegalStateException();
        return fooModel;
    }

    @Override
    public ElementManager getElementManager()
    {
        if (elementManager == null)
            throw new IllegalStateException();
        return elementManager;
    }

    private FooModelManager()
    {
    }
}
```

An instance of the `ElementManager` is parameterized with an `IBodyCache` --
a strategy for storing handle/body relationships. Each Handly-based model
should provide its own, model-specific implementation of the `IBodyCache`,
which may be as tricky as overflowing LRU cache(s) or as simple as a `HashMap`:

```java
// package org.eclipse.handly.internal.examples.basic.ui.model

/**
 * The Foo Model cache.
 */
class FooModelCache
    implements IBodyCache
{
    private static final int DEFAULT_PROJECT_SIZE = 5;

    private Object modelBody; // Foo model element's body
    private Map<IElement, Object> projectCache; // open Foo projects

    public FooModelCache()
    {
        projectCache = new HashMap<>(DEFAULT_PROJECT_SIZE);
    }

    @Override
    public Object get(IElement element)
    {
        if (element instanceof IFooModel)
            return modelBody;
        else if (element instanceof IFooProject)
            return projectCache.get(element);
        else
            return null;
    }

    @Override
    public Object peek(IElement element)
    {
        if (element instanceof IFooModel)
            return modelBody;
        else if (element instanceof IFooProject)
            return projectCache.get(element);
        else
            return null;
    }

    @Override
    public void put(IElement element, Object body)
    {
        if (element instanceof IFooModel)
            modelBody = body;
        else if (element instanceof IFooProject)
            projectCache.put(element, body);
    }

    @Override
    public void remove(IElement element)
    {
        if (element instanceof IFooModel)
            modelBody = null;
        else if (element instanceof IFooProject)
            projectCache.remove(element);
    }
}
```

Now we can implement the remaining abstract method `getModelManager_`
of the `FooModel` class. However, since every element of our model would
implement that method in exactly the same way, it might be useful to extract
the implementation to an internal 'mix-in' interface:

```java
interface IFooElementInternal
    extends IModelManager.Provider
{
    @Override
    default IModelManager getModelManager_()
    {
        return FooModelManager.INSTANCE;
    }
}
```
and have the `FooModel` implement it:

```java
public class FooModel
    extends Element
    implements IFooModel, IFooElementInternal, IModelImpl
{
    // ...
}
```

Now that we have a complete implementation for the class `FooModel`, let's
test it. But before we can do this, two more items require our attention.

First, to have the class `FooProject` compile without errors, we need to
make it implement `IFooElementInternal` and provide an implementation for
the `buildStructure_` method:

```java
public class FooProject
    extends Element
    implements IFooProject, IFooElementInternal
{
    // ...

    @Override
    public void buildStructure_(IContext context, IProgressMonitor monitor)
        throws CoreException
    {
        // no children for now
    }
}
```

For the moment, a `FooProject` won't have any child elements, so its
`buildStructure_` method is left empty.

Next, to make writing tests for the model a bit more straightforward,
we need to define some handy methods in the interfaces for our model elements.
Note that our interfaces extend the interface `IElementExtension`, which
introduces a number of generally useful default methods for model elements,
effectively acting like a trait. We could just as well define our model API
entirely from scratch -- the model interfaces are not required to extend
any Handly interfaces -- but have chosen to extend `IElementExtension`
for convenience.

```java
/**
 * Represents the root Foo element corresponding to the workspace.
 * Since there is only one such root element, it is commonly referred to
 * as <em>the</em> Foo Model element.
 */
public interface IFooModel
    extends IElementExtension
{
    /**
     * Returns the Foo project with the given name. The given name must be a
     * valid path segment as defined by {@link IPath#isValidSegment(String)}
     * This is a handle-only method. The project may or may not exist.
     *
     * @param name the name of the Foo project (not <code>null</code>)
     * @return the Foo project with the given name (never <code>null</code>)
     */
    IFooProject getFooProject(String name);

    /**
     * Returns the Foo projects in this Foo Model.
     *
     * @return the Foo projects in this Foo Model (never <code>null</code>)
     * @throws CoreException if this request fails
     */
    IFooProject[] getFooProjects() throws CoreException;

    /**
     * Returns the workspace associated with this Foo Model.
     * This is a handle-only method.
     *
     * @return the workspace associated with this Foo Model
     *  (never <code>null</code>)
     */
    IWorkspace getWorkspace();
}

/**
 * Represents a Foo project.
 */
public interface IFooProject
    extends IElementExtension
{
    /**
     * Foo project nature id.
     */
    String NATURE_ID = FooProjectNature.ID;

    @Override
    default IFooModel getParent()
    {
        return (IFooModel)IElementExtension.super.getParent();
    }

    /**
     * Returns the <code>IProject</code> on which this
     * <code>IFooProject</code> was created.
     * This is handle-only method.
     *
     * @return the <code>IProject</code> on which this
     * <code>IFooProject</code> was created
     * (never <code>null</code>)
     */
    IProject getProject();
}
```

The newly defined methods are implemented as follows:

```java
// FooModel.java

    @Override
    public IFooProject getFooProject(String name)
    {
        return new FooProject(this, workspace.getRoot().getProject(name));
    }

    @Override
    public IFooProject[] getFooProjects() throws CoreException
    {
        IElement[] children = getChildren();
        int length = children.length;
        IFooProject[] result = new IFooProject[length];
        System.arraycopy(children, 0, result, 0, length);
        return result;
    }

    @Override
    public IWorkspace getWorkspace()
    {
        return workspace;
    }
```

```java
// FooProject.java

    @Override
    public IProject getProject()
    {
        return project;
    }
```

We will also use a facade to the Foo model in the form of the `FooModelCore`
class:

```java
// package org.eclipse.handly.examples.basic.ui.model

/**
 * Facade to the Foo Model.
 */
public class FooModelCore
{
    /**
     * Returns the root Foo element.
     *
     * @return the root Foo element (never <code>null</code>)
     */
    public static IFooModel getFooModel()
    {
        return FooModelManager.INSTANCE.getModel();
    }

    /**
     * Returns the Foo project corresponding to the given project.
     * <p>
     * Note that no check is done at this time on the existence
     * or the nature of this project.
     * </p>
     *
     * @param project the given project (maybe <code>null</code>)
     * @return the Foo project corresponding to the given project,
     *  or <code>null</code> if the given project is <code>null</code>
     */
    public static IFooProject create(IProject project)
    {
        if (project == null)
            return null;
        return getFooModel().getFooProject(project.getName());
    }

    private FooModelCore()
    {
    }
}
```

That's all for now. Let's test it!

## Testing the Model

We provide a test fragment called `org.eclipse.handly.examples.basic.ui.tests`
in the [Step Zero repository](https://github.com/pisv/gethandly.0).
It is probably a good idea to checkout this fragment into your workspace
and use it as a starting point for writing tests for the model.

The `workspace` folder of this fragment contains some set-up data for tests --
just a few predefined projects for you to run tests against. To use this data,
you would extend your test class from the Handly-provided `WorkspaceTestCase`
and make calls to `setUpProject` from within your `setUp` method, passing
the name of the predefined project as the sole argument:

```java
// package org.eclipse.handly.internal.examples.basic.ui.model

/**
 * <code>FooModel</code> tests.
 */
public class FooModelTest
    extends WorkspaceTestCase
{
    @Override
    protected void setUp() throws Exception
    {
        super.setUp();
        setUpProject("Test001"); // a predefined project with Foo nature
        setUpProject("SimpleProject"); // another one without Foo nature
    }
}
```

The inherited `setUpProject` method creates a new project in the runtime
workspace by copying the project's contents from the `workspace` folder of
the test fragment. It returns the created and opened `IProject`.

Let's write a simple test:

```java
// FooModelTest.java

    public void testFooModel() throws Exception
    {
        IFooModel fooModel = FooModelCore.getFooModel();
        IFooProject[] fooProjects = fooModel.getFooProjects();
        assertEquals(1, fooProjects.length);
        IFooProject fooProject = fooProjects[0];
        assertEquals("Test001", fooProject.getName());
    }
```

Run it as a JUnit Plug-in Test. You can use the predefined launch
configuration in the test fragment for that.

Hmm, we've got an error!

```
java.lang.IllegalStateException
	at org.eclipse.handly.internal.examples.basic.ui.model.FooModelManager.getFooModel
	at org.eclipse.handly.examples.basic.ui.model.FooModelCore.getFooModel
	at org.eclipse.handly.internal.examples.basic.ui.model.FooModelTest.testFooModel
```

That goes to show you that those little tests may have some value some times ;-)

It appears that we forgot to start up the `FooModelManager` in the bundle's
Activator. Let's not forget, then, to stop it too.

```java
// package org.eclipse.handly.internal.examples.basic.ui

/**
 * Hand-written subclass of the Xtext-generated {@link FooActivator}.
 */
public class Activator
    extends FooActivator
{
    public static final String PLUGIN_ID =
        "org.eclipse.handly.examples.basic.ui"; //$NON-NLS-1$

    public static void log(IStatus status)
    {
        getInstance().getLog().log(status);
    }

    public static IStatus createErrorStatus(String msg, Throwable e)
    {
        return new Status(IStatus.ERROR, PLUGIN_ID, 0, msg, e);
    }

    @Override
    public void start(BundleContext bundleContext) throws Exception
    {
        super.start(bundleContext);
        FooModelManager.INSTANCE.startup();
    }

    @Override
    public void stop(BundleContext bundleContext) throws Exception
    {
        try
        {
            FooModelManager.INSTANCE.shutdown();
        }
        finally
        {
            super.stop(bundleContext);
        }
    }
}
```

Green bar! Apparently it works now and it encourages us to add more complex
tests:

```java
// FooModelTest.java

    public void testFooModel() throws Exception
    {
        IFooModel fooModel = FooModelCore.getFooModel();
        IFooProject[] fooProjects = fooModel.getFooProjects();
        assertEquals(1, fooProjects.length);
        IFooProject fooProject = fooProjects[0];
        assertEquals("Test001", fooProject.getName());

        // new code -->
        IFooProject fooProject2 = fooModel.getFooProject("Test002");
        assertFalse(fooProject2.exists());
        setUpProject("Test002"); // a second project with Foo nature
        assertTrue(fooProject2.exists());
        fooProjects = fooModel.getFooProjects();
        assertEquals(2, fooProjects.length);
        assertTrue(Arrays.asList(fooProjects).contains(fooProject));
        assertTrue(Arrays.asList(fooProjects).contains(fooProject2));
        // <-- new code
    }
```

Failure!!!

```
junit.framework.AssertionFailedError: expected:<2> but was:<1>
	at org.eclipse.handly.internal.examples.basic.ui.model.FooModelTest.testFooModel
```

We should have had two Foo projects, but got only one. What's wrong?

Okay, the reason is simple enough. When the first call to `getFooProjects` is
made, the `FooModel` initializes its `Body` with a list of the currently open
Foo projects (only `Test001` at the moment) and puts the initialized body in
the model cache. Then, a new Foo project (`Test002`) is created, but nothing
in the Foo model takes notice of that and is going to update the cached body!
So the second call to `getFooProjects` returns the same (stale) result:
`Test001` only.

Clearly, we need *something* in the Foo model to take care of it by updating
the model cache in response to resource changes in the workspace. We need
a *Delta Processor*. This brings us to the next section.

## Keeping the Model Up-to-Date

To keep our model up-to-date, we need to respond to resource changes in the
Eclipse workspace. If you need a refresher on resource change listeners,
the eclipse.org article [How You've Changed!](https://www.eclipse.org/articles/Article-Resource-deltas/resource-deltas.html)
written by John Arthorne is an excellent reference. We really commend it to you!

First, let's make the `FooModelManager` implement `IResourceChangeListener`
and subscribe to `POST_CHANGE` events in the workspace:

```java
public class FooModelManager
    implements IModelManager, IResourceChangeListener
{
    // ...

    public void startup() throws Exception
    {
        fooModel = new FooModel();
        elementManager = new ElementManager(new FooModelCache());
        // new code -->
        fooModel.getWorkspace().addResourceChangeListener(this,
            IResourceChangeEvent.POST_CHANGE);
        // <-- new code
    }

    public void shutdown() throws Exception
    {
        // new code -->
        fooModel.getWorkspace().removeResourceChangeListener(this);
        // <-- new code
        elementManager = null;
        fooModel = null;
    }
```

Next, we need to implement the inherited `resourceChanged` method:

```java
// FooModelManager.java

    @Override
    public void resourceChanged(IResourceChangeEvent event)
    {
        if (event.getType() != IResourceChangeEvent.POST_CHANGE)
            return;
        FooDeltaProcessor deltaProcessor = new FooDeltaProcessor();
        try
        {
            event.getDelta().accept(deltaProcessor);
        }
        catch (CoreException e)
        {
            Activator.log(e.getStatus());
        }
    }
```

Here, we just delegate delta processing to the class `FooDeltaProcessor`
that implements `IResourceDeltaVisitor`:

```java
// package org.eclipse.handly.internal.examples.basic.ui.model

/**
 * This class is used by the <code>FooModelManager</code> to process
 * resource deltas and update the Foo Model accordingly.
 */
class FooDeltaProcessor
    implements IResourceDeltaVisitor
{
    @Override
    public boolean visit(IResourceDelta delta) throws CoreException
    {
        switch (delta.getResource().getType())
        {
        case IResource.ROOT:
            return processRoot(delta);

        case IResource.PROJECT:
            return processProject(delta);

        default:
            return true;
        }
    }

    private boolean processRoot(IResourceDelta delta) throws CoreException
    {
        return true;
    }

    private boolean processProject(IResourceDelta delta)
        throws CoreException
    {
        switch (delta.getKind())
        {
        case IResourceDelta.ADDED:
            return processAddedProject(delta);

        case IResourceDelta.REMOVED:
            return processRemovedProject(delta);

        default:
            return true;
        }
    }

    private boolean processAddedProject(IResourceDelta delta)
        throws CoreException
    {
        IProject project = (IProject)delta.getResource();
        if (project.hasNature(IFooProject.NATURE_ID))
        {
            IFooProject fooProject = FooModelCore.create(project);
            addToModel(fooProject);
        }
        return false;
    }

    private boolean processRemovedProject(IResourceDelta delta)
        throws CoreException
    {
        IProject project = (IProject)delta.getResource();
        IFooProject fooProject = FooModelCore.create(project);
        removeFromModel(fooProject);
        return false;
    }

    private static void addToModel(IElement element)
    {
        Body parentBody = findBody(Elements.getParent(element));
        if (parentBody != null)
            parentBody.addChild(element);
        close(element);
    }

    private static void removeFromModel(IElement element)
    {
        Body parentBody = findBody(Elements.getParent(element));
        if (parentBody != null)
            parentBody.removeChild(element);
        close(element);
    }

    private static Body findBody(IElement element)
    {
        return (Body)((Element)element).findBody_();
    }

    private static void close(IElement element)
    {
        ((Element)element).close_();
    }
}
```

The basic idea behind this code is actually quite simple. In response
to a resource change event we update the Foo model by either adding/removing
child elements from the cached bodies or altogether evicting the element's
`Body` from the model cache (with all descendants).

The test case will now pass.

Please note that the actual implementation of `FooDeltaProcessor`
in the [Step One repository](https://github.com/pisv/gethandly.1st)
is a bit more involved, since there are several ways in which a project
can change: it may be closed or (re-)opened, its description may change, etc.
We spare you the details because they add nothing substantial to our discussion.
If you are interested, you can study the complete implementation of the
`FooDeltaProcessor` and the corresponding `FooModelTest`.

## Closing Step One

Before we move onto the next step, let's review what we've done so far.

We implemented a simple but complete Handly-based model to give you
a taste of the basic structuring and behavior as well as of the entire
development process. We also wrote some tests and implemented a
resource delta processor that keeps the model up-to-date.

The code may seem quite involved for such a simple model, but most
of the code is actually the infrastructure that will not change much
when we take this basic model and add enough functionality to build a
complete *code model* for the Foo language in the next step:
[[The Rest of the Model|Step Two]].
