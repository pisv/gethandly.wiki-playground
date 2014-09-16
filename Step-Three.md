# Step Three: Viewing the Model

In [[Step Two]] we have built a complete code model for
our programming language, Foo. Now, let's put it into real use!
In this step we will display the model in a kind of Navigator view.
It should also serve as a starting point for our next step,
where we will apply the working copy to bring our model and view
even further to life. The complete source code for this step of
the running example is available in the [Step Three repository]
(https://github.com/pisv/gethandly.3rd).

We'll start building the view in a moment, but first, some preliminary
work needs to be done.

For convenience, we will introduce a marker interface `IFooElement`:

```java
// package org.eclipse.handly.examples.basic.ui.model

public interface IFooElement
    extends IHandle
{
}
```

for all elements provided by the Foo model:

```java
public interface IFooModel
    extends IHandle, IFooElement

public interface IFooProject
    extends IHandle, IFooElement

public interface IFooFile
    extends ISourceFile, IFooElement

public interface IFooVar
    extends ISourceConstruct, IFooElement

public interface IFooDef
    extends ISourceConstruct, IFooElement
```

We will also define a new helper method in the `FooModelCore`:

```java
// FooModelCore.java

    /**
     * Returns the Foo element corresponding to the given resource, or 
     * <code>null</code> if unable to associate the given resource 
     * with an element of the Foo Model.
     *
     * @param resource the given resource (maybe <code>null</code>)
     * @return the Foo element corresponding to the given resource, or 
     *  <code>null</code> if unable to associate the given resource 
     *  with an element of the Foo Model
     */
    public static IFooElement create(IResource resource)
    {
        if (resource == null)
            return null;
        int type = resource.getType();
        switch (type)
        {
        case IResource.PROJECT:
            return create((IProject)resource);
        case IResource.FILE:
            return create((IFile)resource);
        case IResource.ROOT:
            return getFooModel();
        default:
            return null;
        }
    }
```

Now we can proceed with building a Navigator view for our model.

First, let's add the Common Navigator Framework a.k.a. CNF
(`org.eclipse.ui.navigator`) to the list of required bundles
of the `org.eclipse.handly.examples.basic.ui` plug-in:

```
Require-Bundle: org.eclipse.handly.examples.basic,
 org.eclipse.handly,
 org.eclipse.handly.xtext.ui,
 org.eclipse.xtext.ui,
 org.eclipse.xtext.ui.shared,
 org.eclipse.ui.navigator
```

The view is defined declaratively in the `plugin.xml`
via a number of extension points:

```xml
   <extension
         point="org.eclipse.ui.views">
      <category
            id="org.eclipse.handly.examples.basic.ui.fooCategory"
            name="Foo (Handly Examples)">
      </category>
      <view
            id="org.eclipse.handly.examples.basic.ui.views.fooNavigator"
            name="Foo Navigator"
            class=
"org.eclipse.handly.internal.examples.basic.ui.navigator.FooNavigator"
            category="org.eclipse.handly.examples.basic.ui.fooCategory"
            restorable="true">
      </view>
   </extension>
   <extension
         point="org.eclipse.ui.navigator.viewer">
      <viewer
            viewerId=
"org.eclipse.handly.examples.basic.ui.views.fooNavigator">
      </viewer>
      <viewerContentBinding
            viewerId=
"org.eclipse.handly.examples.basic.ui.views.fooNavigator">
         <includes>
            <contentExtension
                  pattern=
"org.eclipse.handly.examples.basic.ui.navigator.fooContent">
            </contentExtension>
         </includes>
      </viewerContentBinding>
   </extension>
   <extension
         point="org.eclipse.ui.navigator.navigatorContent">
      <navigatorContent
            id="org.eclipse.handly.examples.basic.ui.navigator.fooContent"
            name="Foo Content"
            contentProvider=
"org.eclipse.handly.internal.examples.basic.ui.FooContentProvider"
            labelProvider=
"org.eclipse.handly.internal.examples.basic.ui.FooLabelProvider">
         <triggerPoints>
            <or>
               <instanceof
                     value=
"org.eclipse.handly.examples.basic.ui.model.IFooElement">
               </instanceof>
            </or>
         </triggerPoints>
         <possibleChildren>
            <or>
               <instanceof
                     value=
"org.eclipse.handly.examples.basic.ui.model.IFooElement">
               </instanceof>
            </or>
         </possibleChildren>
      </navigatorContent>
   </extension>
```

The CNF-specific details are largely irrelevant to our discussion.
In a nutshell, we have defined a view named **Foo Navigator**
in the category **Foo (Handly Examples)**. The view will be implemented
by the class `FooNavigator`:

```java
// package org.eclipse.handly.internal.examples.basic.ui.navigator

/**
 * Foo Navigator view.
 */
public class FooNavigator
    extends CommonNavigator
{
    /**
     * Foo Navigator view id.
     */
    public static final String ID =
        "org.eclipse.handly.examples.basic.ui.views.fooNavigator";

    @Override
    protected Object getInitialInput()
    {
        return FooModelCore.getFooModel();
    }
}
```
and will use the following content and label providers:

```java
// package org.eclipse.handly.internal.examples.basic.ui

/**
 * Foo content provider.
 */
public class FooContentProvider
    implements ITreeContentProvider
{
    protected static final Object[] NO_CHILDREN = new Object[0];

    @Override
    public Object[] getElements(Object inputElement)
    {
        return getChildren(inputElement);
    }

    @Override
    public Object[] getChildren(Object parentElement)
    {
        if (parentElement instanceof IFooElement)
        {
            try
            {
                return ((IFooElement)parentElement).getChildren();
            }
            catch (CoreException e)
            {
            }
        }
        return NO_CHILDREN;
    }

    @Override
    public Object getParent(Object element)
    {
        if (element instanceof IFooElement)
            return ((IFooElement)element).getParent();
        return null;
    }

    @Override
    public boolean hasChildren(Object element)
    {
        return getChildren(element).length > 0;
    }

    @Override
    public void inputChanged(Viewer viewer,
        Object oldInput, Object newInput)
    {
    }

    @Override
    public void dispose()
    {
    }
}

/**
 * Foo label provider.
 */
public class FooLabelProvider
    extends LabelProvider
{
    private ResourceManager resourceManager = new LocalResourceManager(
        JFaceResources.getResources());

    @Override
    public String getText(Object element)
    {
        if (element instanceof IFooDef)
        {
            IFooDef def = (IFooDef)element;
            if (def.getArity() == 0)
                return def.getName() + "()";
            try
            {
                return def.getName()
                    + '('
                    + Strings.concat(", ",
                        Arrays.asList(def.getParameterNames())) + ')';
            }
            catch (CoreException e)
            {
            }
        }
        if (element instanceof IFooElement)
            return ((IFooElement)element).getName();
        return super.getText(element);
    };

    @Override
    public Image getImage(Object element)
    {
        if (element instanceof IFooDef)
            return Activator.getImage(Activator.IMG_OBJ_DEF);
        if (element instanceof IFooVar)
            return Activator.getImage(Activator.IMG_OBJ_VAR);
        IResource resource = null;
        if (element instanceof IFooProject || element instanceof IFooFile)
            resource = ((IFooElement)element).getResource();
        if (resource != null)
        {
            IWorkbenchAdapter adapter =
                (IWorkbenchAdapter)resource.getAdapter(
                    IWorkbenchAdapter.class);
            if (adapter != null)
                return (Image)resourceManager.get(
                    adapter.getImageDescriptor(resource));
        }
        return super.getImage(element);
    }

    @Override
    public void dispose()
    {
        resourceManager.dispose();
        super.dispose();
    }
}
```

That should be pretty straightforward.

The missing piece is a couple of icons in the plug-in image registry:

```java
// package org.eclipse.handly.internal.examples.basic.ui

public class Activator
    extends FooActivator
{
    public static final String PLUGIN_ID =
        "org.eclipse.handly.examples.basic.ui";

    public static final String T_OBJ16 = "/obj16/";

    public static final String IMG_OBJ_DEF = 
        PLUGIN_ID + T_OBJ16 + "def.gif";
    public static final String IMG_OBJ_VAR = 
        PLUGIN_ID + T_OBJ16 + "var.gif";

    // ...

    public static Image getImage(String symbolicName)
    {
        return getInstance().getImageRegistry().get(symbolicName);
    }

    public static ImageDescriptor getImageDescriptor(String symbolicName)
    {
        return getInstance().getImageRegistry().getDescriptor(symbolicName);
    }

    @Override
    protected void initializeImageRegistry(ImageRegistry reg)
    {
        reg.put(IMG_OBJ_DEF, imageDescriptorFromSymbolicName(IMG_OBJ_DEF));
        reg.put(IMG_OBJ_VAR, imageDescriptorFromSymbolicName(IMG_OBJ_VAR));
    }

    private static ImageDescriptor imageDescriptorFromSymbolicName(
        String symbolicName)
    {
        String path = "/icons/" + symbolicName.substring(
            PLUGIN_ID.length());
        return imageDescriptorFromPlugin(PLUGIN_ID, path);
    }
```

The code will now compile without errors.

Let's test what we've done so far. Launch a runtime workbench and open
the **Resource** perspective. Import the `Test002` project from the
[Step Three repository](https://github.com/pisv/gethandly.3rd).
(*Hint:* Use **File/Import.../General/Existing Projects into Workspace**,
select the `workspace` folder of the project `org.eclipse.handly.examples.basic.ui.tests`
in a local clone of the git repository, choose the option **Copy projects
into workspace** and select `Test002`.)

It's time to open the **Foo Navigator** view (**Window/Show View/Other...**).
You should see the `Test002` project in it. Now you can browse all elements
of the Foo model by expanding tree items in the view. So far, so good.

You might have a temptation to create a new Foo file in the `Test002` project.
Don't resist it! (To do that, use **File/New/File**, select `Test002` as the
parent folder and enter a file name -- it must end with `.foo`)

Okay, but where is the new file? As you can see, it is shown in the **Project
Explorer**, but is absent in our view. To make it appear in the **Foo
Navigator**, you would have to restart the runtime workbench.

Apparently, the model changed, but the view wasn't listening. But what
kind of change events our view should actually listen to?

In [[Step One|Step-One#keeping-the-model-up-to-date]] we implemented
the `FooDeltaProcessor` that listened to resource change events in the
workspace to keep the model up-to-date. But were our view a resource
change listener, we would need to duplicate much of the logic of the
`FooDeltaProcessor` for analysis of whether and how the underlying resouce
changes affect the Foo model. We would need to duplicate it in this view,
and also in every other view of the model. Besides, as we will see in
[[Step Four]], resource change events don't describe changes in working copies
(source files being edited).

It would be nice if our view could listen to a kind of change events
described in terms of model elements rather than workspace resources.
This brings us to the next section.

## Model Change Events

Handly provides an infrastructure for dealing with *change notifications*
in Handly-based models. It is largely similar in design to corresponding
facilities of the JDT Java model and the Eclipse Platform resource model,
so there is a good chance that you might be familiar with the core concepts.
If not, the eclipse.org article [How You've Changed!](https://www.eclipse.org/articles/Article-Resource-deltas/resource-deltas.html)
written by John Arthorne could be a great introduction to the problematics
of efficiently notifying clients of changes in a handle-based model.
A detailed description of the infrastructure would take an article of its own,
but hopefully the Javadocs can provide the necessary details on the protocols
involved, and here we will give you a taste of what it is about.

The primary interfaces are `IElementChangeListener`, `IElementChangeEvent`,
and `IHandleDelta`. There are also some useful implementations, namely
`ElementChangeEvent`, `HandleDelta`, and `HandleDeltaBuilder`, that can be
subclassed if needed.

The interface `IElementChangeListener` should be implemented by clients
that are interested in receiving notifications of changes to elements of
a Handly-based model. These listeners are installed using a model-specific
subscription mechanism and are given after-the-fact notification of exactly
what elements of the model changed and how they changed.

The object passed to an element change listener is an instance of
`IElementChangeEvent`. The most important bits of information in the event 
are the event type, and the handle delta. The event type is simply an integer
that describes what kind of event occurred. We will focus on `POST_CHANGE`
event type here, i.e. those events that occur during corresponding `POST_CHANGE`
resource change notifications.

The handle delta is actually the root of a tree of `IHandleDelta` objects.
The tree of deltas is structured much like the tree of `IHandle` objects
that makes up the model, so that each delta object corresponds to exactly
one element of the model. The delta hierarchy will include deltas for all
affected elements that existed prior to the model changing operation, and
all affected elements that existed after the operation. Think of it as
the union of the model contents before and after a particular operation,
with all unchanged sub-trees pruned out.

Each delta object provides the following information:

* The element it corresponds to.
* The kind of modification (added, removed, or changed).
* The precise nature of the change (the change flags).
* Deltas for any added, removed, or changed children.
* A summary of what markers changed on the element's corresponding resource.
* A summary of changes to children of the element's corresponding resource
that cannot be described in terms of handle deltas.

In the case where an element has moved, the `ADDED` delta for the destination
also supplies the source element, and the `REMOVED` delta for the source
supplies the destination element. This allows listeners to accurately track
moved elements.

Okay, it's time to put the theory in practice and implement
a change notification mechanism for our model.

First, let's add a subscription facility for installing change listeners
on the Foo model:

```java
// IFooModel.java

    /**
     * Adds the given listener for changes to elements in the Foo Model. 
     * Has no effect if an identical listener is already registered. 
     * <p>
     * Once registered, a listener starts receiving notification
     * of changes to elements in the Foo Model. The listener continues
     * to receive notifications until it is removed.
     * </p>
     *
     * @param listener the listener (not <code>null</code>)
     * @see #removeElementChangeListener(IElementChangeListener)
     */
    void addElementChangeListener(IElementChangeListener listener);

    /**
     * Removes the given element change listener.
     * Has no effect if an identical listener is not registered.
     *
     * @param listener the listener (not <code>null</code>)
     */
    void removeElementChangeListener(IElementChangeListener listener);
```

The implementation just delegates to the `FooModelManager`:

```java
// FooModel.java

    @Override
    public void addElementChangeListener(IElementChangeListener listener)
    {
        FooModelManager.INSTANCE.addElementChangeListener(listener);
    }

    @Override
    public void removeElementChangeListener(IElementChangeListener listener)
    {
        FooModelManager.INSTANCE.removeElementChangeListener(listener);
    }
```

The `FooModelManager` is already a resource change listener, so it makes
sense to encapsulate the change notification mechanism in there:

```java
// FooModelManager.java

    private ListenerList listenerList;

    public void startup() throws Exception
    {
        fooModel = new FooModel();
        handleManager = new HandleManager(new FooModelCache());
        // new code -->
        listenerList = new ListenerList();
        // <-- new code
        fooModel.getWorkspace().addResourceChangeListener(this,
            IResourceChangeEvent.POST_CHANGE);
    }

    public void shutdown() throws Exception
    {
        fooModel.getWorkspace().removeResourceChangeListener(this);
        // new code -->
        listenerList = null;
        // <-- new code
        handleManager = null;
        fooModel = null;
    }

    public void addElementChangeListener(IElementChangeListener listener)
    {
        if (listenerList == null)
            throw new IllegalStateException();
        listenerList.add(listener);
    }

    public void removeElementChangeListener(IElementChangeListener listener)
    {
        if (listenerList == null)
            throw new IllegalStateException();
        listenerList.remove(listener);
    }

    public void fireElementChangeEvent(final IElementChangeEvent event)
    {
        if (listenerList == null)
            throw new IllegalStateException();
        Object[] listeners = listenerList.getListeners();
        for (final Object listener : listeners)
        {
            SafeRunner.run(new ISafeRunnable()
            {
                public void handleException(Throwable exception)
                {
                    // already logged by Platform
                }

                public void run() throws Exception
                {
                    ((IElementChangeListener)listener).elementChanged(
                        event);
                }
            });
        }
    }
```

We are going to fire an element change event in response to those
resource changes in the workspace that actually affect our model. To build
this element change event, we need a mechanism for translating resource deltas
into Foo element deltas. Let the `FooDeltaProcessor` be responsible for that:

```java
// FooDeltaProcessor.java

    private HandleDelta currentDelta = new HandleDelta(
        FooModelCore.getFooModel());

    /**
     * Returns the Foo element delta built from the resource delta. 
     * Returns an empty delta if no Foo elements were affected 
     * by the resource change.
     * 
     * @return Foo element delta (never <code>null</code>)
     */
    public HandleDelta getDelta()
    {
        return currentDelta;
    }
```

Now we can complete the implementation of the `FooModelManager`:

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
        // new code -->
        if (!deltaProcessor.getDelta().isEmpty())
        {
            fireElementChangeEvent(new ElementChangeEvent(
                ElementChangeEvent.POST_CHANGE, deltaProcessor.getDelta()));
        }
        // <-- new code
    }
```

So far, so good, but what about the actual delta translation?

Let's start with writing some tests:

```java
// package org.eclipse.handly.internal.examples.basic.ui.model

/**
 * Foo element change notification tests.
 */
public class FooModelNotificationTest
    extends WorkspaceTestCase
{
    private IFooModel fooModel = FooModelCore.getFooModel();
    private FooModelListener listener = new FooModelListener();

    @Override
    protected void setUp() throws Exception
    {
        super.setUp();
        setUpProject("Test001");
        fooModel.addElementChangeListener(listener);
    }

    @Override
    protected void tearDown() throws Exception
    {
        fooModel.removeElementChangeListener(listener);
        super.tearDown();
    }

    public void testFooModelNotification() throws Exception
    {
        IFooProject fooProject1 = fooModel.getFooProject("Test001");
        IFooProject fooProject2 = fooModel.getFooProject("Test002");

        setUpProject("Test002");
        assertEquality(newDelta().insertAdded(fooProject2), listener.delta);

        IFooFile fooFile1 = fooProject1.getFooFile("test.foo");
        fooFile1.getFile().touch(null);
        assertEquality(
            newDelta().insertChanged(fooFile1, HandleDelta.F_CONTENT),
            listener.delta);

        fooFile1.getFile().delete(true, null);
        assertEquality(newDelta().insertRemoved(fooFile1), listener.delta);
    }

    private HandleDelta newDelta()
    {
        return new HandleDelta(fooModel);
    }

    private static void assertEquality(IHandleDelta expected,
        IHandleDelta actual)
    {
        if (expected == null)
        {
            assertNull(actual);
            return;
        }
        assertNotNull(actual);
        assertEquals(expected.getElement(), actual.getElement());
        assertEquals(expected.getKind(), actual.getKind());
        assertEquals(expected.getFlags(), actual.getFlags());
        assertEquals(expected.getMovedToElement(),
            actual.getMovedToElement());
        assertEquals(expected.getMovedFromElement(),
            actual.getMovedFromElement());
        IHandleDelta[] expectedChildren = expected.getAffectedChildren();
        IHandleDelta[] actualChildren = actual.getAffectedChildren();
        assertEquals(expectedChildren.length, actualChildren.length);
        for (int i = 0; i < expectedChildren.length; i++)
            assertEquality(expectedChildren[i], actualChildren[i]);
    }

    private static class FooModelListener
        implements IElementChangeListener
    {
        public HandleDelta delta;

        @Override
        public void elementChanged(IElementChangeEvent event)
        {
            delta = (HandleDelta)event.getDelta();
        }
    }
}
```

For convenience, we define the `assertEquality` method that checks if
the two handle deltas are equal.

Let's run the test, only to see it fail. It should come as no surprise.
Since we have not implemented the delta translation logic yet,
no Foo element change event would ever be fired. Now that
we have a failing test, let's try to make it pass.

We begin by defining some handy methods for delta conversion:

```java
// FooDeltaProcessor.java

    private void translateAddedDelta(IResourceDelta delta,
        IFooElement element)
    {
        if ((delta.getFlags() & IResourceDelta.MOVED_FROM) == 0)
        {
            // regular addition
            currentDelta.insertAdded(element);
        }
        else
        {
            IFooElement movedFromElement =
                FooModelCore.create(getResource(delta.getMovedFromPath(),
                    delta.getResource().getType()));
            if (movedFromElement == null)
                currentDelta.insertAdded(element);
            else
                currentDelta.insertMovedTo(element, movedFromElement);
        }
    }

    private void translateRemovedDelta(IResourceDelta delta,
        IFooElement element)
    {
        if ((delta.getFlags() & IResourceDelta.MOVED_TO) == 0)
        {
            // regular removal
            currentDelta.insertRemoved(element);
        }
        else
        {
            IFooElement movedToElement =
                FooModelCore.create(getResource(delta.getMovedToPath(),
                    delta.getResource().getType()));
            if (movedToElement == null)
                currentDelta.insertRemoved(element);
            else
                currentDelta.insertMovedFrom(element, movedToElement);
        }
    }

    private void contentChanged(IFooFile fooFile)
    {
        close(fooFile);
        // new code -->
        currentDelta.insertChanged(fooFile, IHandleDelta.F_CONTENT);
        // <-- new code
    }

    private static IResource getResource(IPath fullPath, int resourceType)
    {
        IWorkspaceRoot root = ResourcesPlugin.getWorkspace().getRoot();
        switch (resourceType)
        {
        case IResource.ROOT:
            return root;

        case IResource.PROJECT:
            return root.getProject(fullPath.lastSegment());

        case IResource.FOLDER:
            return root.getFolder(fullPath);

        case IResource.FILE:
            return root.getFile(fullPath);

        default:
            return null;
        }
    }
```

The code is pretty straightforward. The only subtle part is handling
move operations as opposed to regular additions and removals.

With that in place, we can implement the actual delta translation logic.
It will be scattered throughout the existing delta processing methods,
so these methods are listed here in their entirety:

```java
// FooDeltaProcessor.java

    private boolean processAddedProject(IResourceDelta delta)
        throws CoreException
    {
        IProject project = (IProject)delta.getResource();
        if (project.hasNature(IFooProject.NATURE_ID))
        {
            IFooProject fooProject = FooModelCore.create(project);
            addToModel(fooProject);
            // new code -->
            translateAddedDelta(delta, fooProject);
            // <-- new code
        }
        return false;
    }

    private boolean processRemovedProject(IResourceDelta delta)
        throws CoreException
    {
        IProject project = (IProject)delta.getResource();
        if (wasFooProject(project))
        {
            IFooProject fooProject = FooModelCore.create(project);
            removeFromModel(fooProject);
            // new code -->
            translateRemovedDelta(delta, fooProject);
            // <-- new code
        }
        return false;
    }

    private boolean processAddedFile(IResourceDelta delta)
    {
        IFile file = (IFile)delta.getResource();
        IFooFile fooFile = FooModelCore.create(file);
        if (fooFile != null)
        {
            addToModel(fooFile);
            // new code -->
            translateAddedDelta(delta, fooFile);
            // <-- new code
        }
        return false;
    }

    private boolean processRemovedFile(IResourceDelta delta)
    {
        IFile file = (IFile)delta.getResource();
        IFooFile fooFile = FooModelCore.create(file);
        if (fooFile != null)
        {
            removeFromModel(fooFile);
            // new code -->
            translateRemovedDelta(delta, fooFile);
            // <-- new code
        }
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
```

That's it. The method `processChangedFile` didn't actually change (it is
the aforementioned `contentChanged` method that did), but we listed it
for completeness.

Well, in this case it was quite simple, but for models with
a more complicated mapping to workspace resources the implementation
would be much more elaborate.

Also, please note that the actual implementation of `FooDeltaProcessor`
in the [Step Three repository](https://github.com/pisv/gethandly.3rd)
is a bit more involved, since there are several ways in which a project
can change: it may be closed or (re-)opened, its description may change, etc.
We spare you the details because they add nothing substantial to discussion.
If you are interested, you can study yourself the complete implementation
of the `FooDeltaProcessor` and the corresponding `FooModelNotificationTest`.

Run the test again. Success!

Great, now we need to have our view respond to Foo element changes
for keeping its contents up-to-date with the model.

Let's make the `FooNavigator` implement `IElementChangeListener`,
subscribe to notifications of element changes with the Foo model,
and do a full refresh whenever there are any changes:

```java
// package org.eclipse.handly.internal.examples.basic.ui.navigator

public class FooNavigator
    extends CommonNavigator
    implements IElementChangeListener
{
    // ...

    @Override
    public void init(IViewSite site) throws PartInitException
    {
        super.init(site);
        FooModelCore.getFooModel().addElementChangeListener(this);
    }

    @Override
    public void dispose()
    {
        FooModelCore.getFooModel().removeElementChangeListener(this);
        super.dispose();
    }

    @Override
    public void elementChanged(IElementChangeEvent event)
    {
        // NOTE: don't hold on the event or its delta.
        // The delta is only valid during the dynamic scope
        // of the notification. In particular, don't pass it
        // to another thread (e.g. via asyncExec). 
        final Control control = getCommonViewer().getControl();
        control.getDisplay().asyncExec(new Runnable()
        {
            public void run()
            {
                if (!control.isDisposed())
                {
                    // full refresh should suffice for our example,
                    // but not for production code!
                    refresh();
                }
            }
        });
    }

    private void refresh()
    {
        Control control = getCommonViewer().getControl();
        control.setRedraw(false);
        BusyIndicator.showWhile(control.getDisplay(), new Runnable()
        {
            public void run()
            {
                TreePath[] treePaths = 
                    getCommonViewer().getExpandedTreePaths();
                getCommonViewer().refresh(); 
                getCommonViewer().setExpandedTreePaths(treePaths);
            }
        });
        control.setRedraw(true);
    }
 }
```

It looks deceptively simple, but were it production code, we would anylize
the Foo element delta and update the view's contents incrementally.
We leave it as an exercise to the reader.

Launch the runtime workbench, expand the `Test002` project in the **Foo
Navigator**, and create a `.foo` file in the project (**File/New/File**).
The new file will now be shown instantly in the view.

## Closing Step Three

In this step we have displayed the Foo model in a Navigator-like view
and made it alive by firing model change notifications that the view
can subscribe and respond to.

That's great, but if you open a Foo file in the editor and begin changing
its structure (for example, by adding new variables and functions), you will
soon see that our view shows the outline of the file's saved contents
rather than that of the editor's current contents. We are going to fix it
in the next step: [[Working Copy|Step Four]].