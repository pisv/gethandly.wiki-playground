# Step One: Basic Model

In this step we will take you through the entire development process
of a Handly-based model, but for a little bit of functionality. We will
implement a basic model with only two levels: `FooModel` containing `FooProject`s.
While simple, this is going to be a complete, fully functional implementation.
It shall demonstrate the basic structuring and behavior of a Handly-based model
and give you a taste of what it is all about. It shall also serve as a
starting point for our next step. The complete source code for this step
of the running example is available in the [Step One repository]
(https://github.com/pisv/gethandly.1st).

Let's start implementing the model. In Handly, every model element is an
instance of `IHandle`. This interface is specialized with `ISourceElement`
for representing source elements, which is further specialized with
`ISourceFile` and `ISourceConstruct` for representing source files and
their structural elements. Handly provides basic partial implementations
for each of these interfaces. In this part of the article we will deal
solely with `IHandle` and its basic implementation, class `Handle`.
We will get to source elements in the next step.

We begin by defining interfaces for Foo model elements
in the package `org.eclipse.handly.examples.basic.ui.model`
of the `org.eclipse.handly.examples.basic.ui` bundle:

```java
public interface IFooModel
    extends IHandle
{
}

public interface IFooProject
    extends IHandle
{
}
```

Let's define corresponding implementation classes in the package
`org.eclipse.handly.internal.examples.basic.ui.model` of the same bundle:

```java
public class FooModel
    extends Handle
    implements IFooModel
{
}

public class FooProject
    extends Handle
    implements IFooProject
{
}
```

At the moment, the code doesn't yet compile. We need to fill in the blanks
and complete the implementation by providing the appropriate constructors
and overriding the inherited abstract methods. Let's begin with the class
`FooModel`:

```java
/**
 * Represents the root Foo element corresponding to the workspace. 
 */
public class FooModel
    extends Handle
    implements IFooModel
{
    private final IWorkspace workspace;

    /**
     * Constructs a handle for the root Foo element.
     */
    public FooModel()
    {
        super(null, null);
        this.workspace = ResourcesPlugin.getWorkspace();
    }

    @Override
    public IResource getResource()
    {
        return workspace.getRoot();
    }

    @Override
    protected void validateExistence()
    {
        // always exists
    }
```

Every model element (a handle) is a value object, so it must be immutable and
must override equals() and hashCode() appropriately. The basic implementation
of `equals()` and `hashCode()` provided in the class `Handle` is sufficient in
many cases.

Elements that have an underlying resource should return it from the
`getResource()` method. In this case, the workspace root is returned.

Handles can refer to non-existing elements; existence can be tested with
`exists()`. The `validateExistence()` method should throw a `CoreException`
if the element may not begin existence in the model (for example, if its
underlying resource does not exist). In this case, this method does nothing
as the root element always exists.

That leaves us with two more abstract methods: `getHandleManager` and
`buildStructure`. Those methods are central to the implementation of
handle/body idiom in Handly and require some preliminary work
before they can be implemented.

The Handly-provided class `HandleManager` manages handle/body relationships
for a handle-based model. Generally, each model will have its own instance
of the `HandleManager` and each element of the model will return that instance
from its `getHandleManager()` method. In that way, the handle manager is shared
between all elements of the model.

Let's define a special manager class for the Foo model that will hold
the single instance of the `IFooModel` and the single instance of the
`HandleManager` for the model:

```java
// package org.eclipse.handly.internal.examples.basic.ui.model

/**
 * The manager for the Foo Model. 
 *
 * @threadsafe This class is intended to be thread-safe
 */
public class FooModelManager
{
    /**
     * The sole instance of the manager. 
     */
    public static final FooModelManager INSTANCE = new FooModelManager();

    private IFooModel fooModel;
    private HandleManager handleManager;

    public void startup() throws Exception
    {
        fooModel = new FooModel();
        handleManager = new HandleManager(new FooModelCache());
    }

    public void shutdown() throws Exception
    {
        handleManager = null;
        fooModel = null;
    }

    public IFooModel getFooModel()
    {
        if (fooModel == null)
            throw new IllegalStateException();
        return fooModel;
    }

    public HandleManager getHandleManager()
    {
        if (handleManager == null)
            throw new IllegalStateException();
        return handleManager;
    }

    private FooModelManager()
    {
    }
}
```

An instance of the `HandleManager` must be parameterized with an `IBodyCache`--
a strategy for storing handle/body relationships. Each Handly-based model
should provide its own, model-specific implementation of the `IBodyCache`,
which may be as simple as a `HashMap` or as tricky as overflowing LRU cache(s).
We'll get back to it in a moment.

As another prerequisite, we need to define a constructor for the `FooProject`
class:

```java
/**
 * Represents a Foo project.
 */
public class FooProject
    extends Handle
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
    public IResource getResource()
    {
        return project;
    }
}
```

Now we can implement the remaining abstract methods of the `FooModel` class.

First, `getHandleManager()`:

```java
    @Override
    protected HandleManager getHandleManager()
    {
        return FooModelManager.INSTANCE.getHandleManager();
    }
```

That's it. Rather trivial.

Next goes the `buildStructure` method:

```java
    @Override
    protected void buildStructure(Body body, Map<IHandle, Body> newElements)
        throws CoreException
    {
        IProject[] projects = workspace.getRoot().getProjects();
        List<IFooProject> fooProjects =
            new ArrayList<IFooProject>(projects.length);
        for (IProject project : projects)
        {
            if (project.isOpen() && project.hasNature(IFooProject.NATURE_ID))
            {
                fooProjects.add(new FooProject(this, project));
            }
        }
        body.setChildren(fooProjects.toArray(new IHandle[fooProjects.size()]));
    }
```

The central idea of the *handle/body idiom* as implemented in Handly is that
mutable structure and properties of a model element are stored separately
in an internal `Body`, while the handle holds immutable, 'key' information
(recall that handles are value objects). The method `buildStructure` must
initialize the given `Body` based on the element's current contents. In this
case, we set the currently open Foo projects as the children of the `FooModel`.
We don't use the additional parameter `newElements` here because we intend for
the `FooProject` to be responsible for building its structure, rather than have
the `FooModel` to also build the structure of its projects and put it
(as handle/body pairs) in the `newElements` map. The `FooProject` and the
`FooModel` are said to be *openable* elements because they know how to open
themselves (build their structure and properties) when asked to do so. In that
way, the model is populated with its elements lazily, on demand. In contrast,
elements inside a source file are *never* openable because the source file
builds all of its structure in one go by parsing its text contents, as we
shall see in the next step. See the [System Overview]
(http://www.eclipse.org/downloads/download.php?file=/handly/docs/handly-overview.pdf&r=1)
for more information on the architecture.

Now that we have a complete implementation for the class `FooModel`, let's
test it. But before we can do it, a couple of more items require our attention.

First, to have the class `FooProject` compile without errors, we need to
implement the inherited abstract methods:

```java
    @Override
    protected HandleManager getHandleManager()
    {
        return FooModelManager.INSTANCE.getHandleManager();
    }

    @Override
    protected void validateExistence() throws CoreException
    {
        if (!project.exists())
            throw new CoreException(Activator.createErrorStatus(
                MessageFormat.format(
                    "Project ''{0}'' does not exist in workspace", name), null));

        if (!project.isOpen())
            throw new CoreException(
                Activator.createErrorStatus(
                    MessageFormat.format("Project ''{0}'' is not open", name),
                    null));

        if (!project.hasNature(NATURE_ID))
            throw new CoreException(
                Activator.createErrorStatus(MessageFormat.format(
                    "Project ''{0}'' does not have the Foo nature", name), null));
    }

    @Override
    protected void buildStructure(Body body, Map<IHandle, Body> newElements)
        throws CoreException
    {
        // no children for now
    }
```

In this case, `validateExistence` throws a `CoreException` if the underlying
workspace project is not accessible or doesn't have the Foo nature. For the
moment, a `FooProject` won't have any child elements, so its `buildStructure`
method is left empty.

Next, we need to provide an implementation for the `FooModelCache`:

```java
// package org.eclipse.handly.internal.examples.basic.ui.model

/**
 * The Foo Model cache.
 */
class FooModelCache
    implements IBodyCache
{
    private static final int DEFAULT_PROJECT_SIZE = 5;

    private Body modelBody; // Foo model element's body
    private HashMap<IHandle, Body> projectCache; // cache of open Foo projects

    public FooModelCache()
    {
        projectCache = new HashMap<IHandle, Body>(DEFAULT_PROJECT_SIZE);
    }

    @Override
    public Body get(IHandle handle)
    {
        if (handle instanceof IFooModel)
            return modelBody;
        else if (handle instanceof IFooProject)
            return projectCache.get(handle);
        else
            return null;
    }

    @Override
    public Body peek(IHandle handle)
    {
        if (handle instanceof IFooModel)
            return modelBody;
        else if (handle instanceof IFooProject)
            return projectCache.get(handle);
        else
            return null;
    }

    @Override
    public void put(IHandle handle, Body body)
    {
        if (handle instanceof IFooModel)
            modelBody = body;
        else if (handle instanceof IFooProject)
            projectCache.put(handle, body);
    }

    @Override
    public void remove(IHandle handle)
    {
        if (handle instanceof IFooModel)
            modelBody = null;
        else if (handle instanceof IFooProject)
            projectCache.remove(handle);
    }
}
```

Simple enough, right?

Finally, to make writing tests for the model a bit more straightforward,
we need to define some handy methods for our model elements:

```java
/**
 * Represents the root Foo element corresponding to the workspace. 
 * Since there is only one such root element, it is commonly referred to 
 * as <em>the</em> Foo Model element.
 */
public interface IFooModel
    extends IHandle
{
    /**
     * Returns the Foo project with the given name. The given name must be 
     * a valid path segment as defined by {@link IPath#isValidSegment(String)}. 
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
    extends IHandle
{
    /**
     * Foo project nature id.
     */
    String NATURE_ID = FooProjectNature.ID;

    /**
     * Returns the <code>IProject</code> on which this <code>IFooProject</code>
     * was created. This is handle-only method.
     *
     * @return the <code>IProject</code> on which this <code>IFooProject</code>
     *  was created (never <code>null</code>)
     */
    IProject getProject();
}
```

and implement them:

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
        IHandle[] children = getChildren();
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
        return FooModelManager.INSTANCE.getFooModel();
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

## Testing

We provide a test fragment called `org.eclipse.handly.examples.basic.ui.tests`
with a predefined launch configuration in the [Step One repository]
(https://github.com/pisv/gethandly.1st). It is probably a good idea 
to check it out into your workspace if you have not done so already.

The `workspace` folder of this fragment contains some set-up data for tests --
just a few predefined projects for you to run tests against. To use this data,
you would extend your test class from the Handly-provided `WorkspaceTestCase`
and make calls to `setUpProject` from within your `setUp` method, passing
the name of the project as the sole argument:

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
        setUpProject("SimpleProject"); // a predefined project without Foo nature
    }
}
```

The inherited `setUpProject` method creates a new project in the run-time
workspace by copying the project's content from the `workspace` folder of
the test fragment. It returns the created and opened `IProject`.

Let's write the simplest possible test:

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

Run it! (as a JUnit Plug-in Test) You can use the predefined launch
configuration for that.

Hmm, we've got an error:

```
java.lang.IllegalStateException
	at org.eclipse.handly.internal.examples.basic.ui.model.FooModelManager.getFooModel(FooModelManager.java:70)
	at org.eclipse.handly.examples.basic.ui.model.FooModelCore.getFooModel(FooModelCore.java:28)
	at org.eclipse.handly.internal.examples.basic.ui.model.FooModelTest.testFooModel(FooModelTest.java:40)
```

That goes to show you that those little tests may have some value some times!

The reason is that we forgot to start up the `FooModelManager` in the bundle's
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

Green bar! Apparently it works now and it encourages us to add even more
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

        IFooProject fooProject2 = fooModel.getFooProject("Test002");
        assertFalse(fooProject2.exists());
        setUpProject("Test002"); // a second project with Foo nature
        assertTrue(fooProject2.exists());
        fooProjects = fooModel.getFooProjects();
        assertEquals(2, fooProjects.length);
        assertTrue(Arrays.asList(fooProjects).contains(fooProject));
        assertTrue(Arrays.asList(fooProjects).contains(fooProject2));
    }
```

Failure!!!

```
junit.framework.AssertionFailedError: expected:<2> but was:<1>
	at org.eclipse.handly.internal.examples.basic.ui.model.FooModelTest.testFooModel(FooModelTest.java:51)
```

We should have had two Foo projects, but got only one. What's wrong?

Okay, the reason is simple enough. When the first call to `getFooProjects` is
made, the `FooModel` initializes its `Body` with a list of the currently open
Foo projects (only `Test001` at the moment) and puts the initialized body in
the model cache. Then, a new Foo project (`Test002`) is created, but nobody
in the Foo model takes notice of that and nobody is going to update the
cached body! So the second call to `getFooProjects` returns the same (stale)
result: `Test001` only. Clearly, we need *somebody* in the Foo model to take
care of that by updating the model cache in response to resource changes
in the workspace. We need a *Delta Processor*. This brings us to the
next section.

## Keeping the model up-to-date

To keep our model up-to-date, we need to respond to resource changes in the
Eclipse workspace. If you need a refresher on resource change listeners,
[this Eclipse Corner article](https://www.eclipse.org/articles/Article-Resource-deltas/resource-deltas.html)
written by John Arthorne is an excellent reference. I really commend it to you!

First, let's make the `FooModelManager` implement `IResourceChangeListener`
and subscribe to `POST_CHANGE` events in the workspace:

```java
public class FooModelManager
    implements IResourceChangeListener
{
    // ...

    public void startup() throws Exception
    {
        fooModel = new FooModel();
        handleManager = new HandleManager(new FooModelCache());
        fooModel.getWorkspace().addResourceChangeListener(this,
            IResourceChangeEvent.POST_CHANGE);
    }

    public void shutdown() throws Exception
    {
        fooModel.getWorkspace().removeResourceChangeListener(this);
        handleManager = null;
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

Here, we just delegate event processing to a new class
`FooDeltaProcessor` that implements `IResourceDeltaVisitor`:

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

    private boolean processProject(IResourceDelta delta) throws CoreException
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

    private static void addToModel(IHandle element)
    {
        Body parentBody = findBody(element.getParent());
        if (parentBody != null)
            parentBody.addChild(element);
        close(element);
    }

    private static void removeFromModel(IHandle element)
    {
        Body parentBody = findBody(element.getParent());
        if (parentBody != null)
            parentBody.removeChild(element);
        close(element);
    }

    private static Body findBody(IHandle element)
    {
        return ((Handle)element).findBody();
    }

    private static void close(IHandle element)
    {
        ((Handle)element).close();
    }
}
```

The idea behind this code is quite simple, actually. In response to 
a resource change event we update the Foo model by adding/removing 
child elements from the cached bodies or by evicting the element's 
`Body` from the model cache (with all of its children).

Run the test again. Success!

Please note that the actual implementation of `FooDeltaProcessor`
in the [Step One repository](https://github.com/pisv/gethandly.1st)
is a bit more involved, since there are several ways in which a project
can change: it may be closed or (re-)opened, its description may change, etc.
We spare you the details because they add nothing substantial to discussion.
If you are interested, you can study yourself the complete implementation
of the `FooDeltaProcessor` and the corresponding `FooModelTest`.

## Closing Step One

Before we move onto the next step, let's review what we've done so far.

We implemented a simple but complete Handly-based model to give you
a taste of the basic structuring and behavior as well as the entire
development process. We also wrote some tests and implemented a
resource delta processor for keeping the model up-to-date.

The code may seem quite involved for such a simple model, but most
of the code is actually the infrastructure that will not change much
when we will take this basic model and add enough functionality to
build a complete *code model* for the Foo language in the next step,
[[The Rest of the Model|Step Two]].
