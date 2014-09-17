# Step Four: Working Copy

In [[Step Three]] we have built a Navigator view for our Handly-based model.
It works fine, except that it cannot display the outline of the editor's
current contents for a Foo file -- it only "sees" the file's saved contents.
In this step we will employ the so-called "working copy" to make the view
even more alive. We will also use the resulting infrastructure to build
the Outline page for the Foo editor -- another view of our model.
The complete source code for this step of the running example is available
in the [Step Four repository](https://github.com/pisv/gethandly.4th).

In Handly, a source file (as a model element) can be switched into a
*working copy* mode. Switching to working copy means that the source file's
structure and properties will no longer correspond to the file's saved contents.
Instead, those structure and properties will reflect the current contents
of the source file's editor. To enable this, the editor needs to be integrated
with Handly.

Fortunately, our Foo language is Xtext-based, and Handly provides
out-of-the-box integration of the working copy facility with the Xtext editor.
To make use of it, we need to introduce a few bindings in the Xtext UI module
for the language:

```java
// FooUiModule.java

    @Override
    public Class<? extends IReconciler> bindIReconciler()
    {
        return HandlyXtextReconciler.class;
    }

    public Class<? extends XtextDocument> bindXtextDocument()
    {
        return HandlyXtextDocument.class;
    }

    public Class<? extends DirtyStateEditorSupport>
        bindDirtyStateEditorSupport()
    {
        return HandlyDirtyStateEditorSupport.class;
    }

    public void configureXtextEditorCallback(Binder binder)
    {
        binder.bind(IXtextEditorCallback.class).annotatedWith(
            Names.named(HandlyXtextEditorCallback.class.getName())).to(
            HandlyXtextEditorCallback.class);
    }

    public Class<? extends ISourceFileFactory> bindISourceFileFactory()
    {
        return FooFileFactory.class;
    }
```

The only missing piece is the `FooFileFactory`. This class should implement
the `ISourceFileFactory` and provide the `ISourceFile` element corresponding
to a given `IFile`:

```java
// package org.eclipse.handly.internal.examples.basic.ui.model

/**
 * Implementation of <code>ISourceFileFactory</code> that must be bound 
 * in the Xtext UI module for the language. It's a required component of 
 * Handly/Xtext integration.
 * </p>
 */
public class FooFileFactory
    implements ISourceFileFactory
{
    @Override
    public ISourceFile getSourceFile(IFile file)
    {
        return FooModelCore.create(file);
    }
}
```

That's it. Nothing more is required to enable Handly/Xtext integration.

Let's test it. Launch a runtime workbench with the workspace where
you have imported the `Test002` project in the [[previous step|Step Three]].
Ensure that you have opened the **Foo Navigator** and expand the `Test002`
and `test.foo` elements in the view. Then, open the `test.foo` file
with the Xtext editor and try declaring a new variable: `var z;`.

Hmm, our view didn't get updated. What's wrong?

Well, we need to notify the view about changes in the working copy.
We will use our existing [[change notification mechanism|Step-Three#model-change-events]]
for that. In that way, every `IElementChangeListener` registered with the
Foo model will get notified about working copy changes. Even better, the
notifications will describe exactly how the working copy changed
(i.e. what variables/functions were added/removed etc.)

To enable notifications about working copy changes, we will override the method
`getReconcileOperation()` in the class `FooFile` and return a *notifying*
reconcile operation that will use a `HandleDeltaBuilder` to build a delta tree
between the "pre-reconciled" and "post-reconciled" state of the working copy.
It will then fire the delta as a `POST_RECONCILE` event:

```java
// FooFile.java

    @Override
    public ReconcileOperation getReconcileOperation()
    {
        return new NotifyingReconcileOperation();
    }

    private class NotifyingReconcileOperation
        extends ReconcileOperation
    {
        @Override
        public void reconcile(Object ast, NonExpiringSnapshot snapshot,
            boolean forced) throws CoreException
        {
            HandleDeltaBuilder deltaBuilder =
                new HandleDeltaBuilder(FooFile.this);

            super.reconcile(ast, snapshot, forced);

            deltaBuilder.buildDelta();
            if (!deltaBuilder.getDelta().isEmpty())
            {
                FooModelManager.INSTANCE.fireElementChangeEvent(
                    new ElementChangeEvent(
                        ElementChangeEvent.POST_RECONCILE,
                        deltaBuilder.getDelta()));
            }
        }
    }
```

The Handly-provided class `HandleDeltaBuilder` builds a delta tree between
the version of a model element at the time the builder was created and
the current version of the element. It performs this operation by locally
caching the contents of the element when the builder is created. When the
method `buildDelta()` is called, it creates a delta over the cached contents
and the new contents.

Launch the runtime workbench and repeat the test. Declare a new variable
and it will now be shown instantly in the **Foo Navigator**. Great,
now close the editor without saving changes. The new variable will
still be displayed in the view. Not good!

To fix it, we need to fire events about switching a Foo file to/from the
working copy mode. To do that, we will extend the `SourceFile`'s method
`workingCopyChanged()`:

```java

    @Override
    protected void workingCopyModeChanged()
    {
        super.workingCopyModeChanged();

        HandleDelta delta = new HandleDelta(getRoot());
        if (file.exists())
            delta.insertChanged(this, HandleDelta.F_WORKING_COPY);
        else if (isWorkingCopy())
            delta.insertAdded(this, HandleDelta.F_WORKING_COPY);
        else
            delta.insertRemoved(this, HandleDelta.F_WORKING_COPY);
        FooModelManager.INSTANCE.fireElementChangeEvent(
            new ElementChangeEvent(ElementChangeEvent.POST_CHANGE, delta));
    }
```

This should be self-explanatory. And while we are here, let's also handle
the working copy as a special case in the `FooDeltaProcessor`:

```java
// FooDeltaProcessor.java

    private void contentChanged(IFooFile fooFile)
    {
        // new code -->
        if (fooFile.isWorkingCopy())
        {
            currentDelta.insertChanged(fooFile, IHandleDelta.F_CONTENT
                | IHandleDelta.F_UNDERLYING_RESOURCE);
            return;
        }
        // <-- new code

        close(fooFile);
        currentDelta.insertChanged(fooFile, IHandleDelta.F_CONTENT);
    }
```

Everything should work fine now.

## Outline View

Now that our model has truly come to life with all that infrastructure
in place, it should be very easy to create a Foo Outline page so that
it would display the same kind of elements as the **Foo Navigator**.
Basically, we've got a model and would like to have a number of views of it.

The `FooOutlinePage` will use existing content and label providers, and
will refresh itself when its contents is affected by a change in the Foo model.
It will also reset its input when the corresponding editor input changes.

```java
// package org.eclipse.handly.internal.examples.basic.ui.outline

/**
 * Foo Outline page.
 */
public class FooOutlinePage
    extends ContentOutlinePage
    implements IXtextEditorAware, IElementChangeListener
{
    private XtextEditor editor;
    private IPropertyListener editorInputListener = new IPropertyListener()
    {
        public void propertyChanged(Object source, int propId)
        {
            if (propId == IEditorPart.PROP_INPUT)
            {
                getTreeViewer().setInput(computeInput());
            }
        }
    };

    @Inject
    private FooContentProvider contentProvider;
    @Inject
    private FooLabelProvider labelProvider;

    @Override
    public void setEditor(XtextEditor editor)
    {
        this.editor = editor;
    }

    @Override
    public void createControl(Composite parent)
    {
        super.createControl(parent);
        getTreeViewer().setContentProvider(contentProvider);
        getTreeViewer().setLabelProvider(labelProvider);
        getTreeViewer().setInput(computeInput());
        editor.addPropertyListener(editorInputListener);
        FooModelCore.getFooModel().addElementChangeListener(this);
    }

    @Override
    public void dispose()
    {
        FooModelCore.getFooModel().removeElementChangeListener(this);
        editor.removePropertyListener(editorInputListener);
        editor.outlinePageClosed();
        super.dispose();
    }

    @Override
    public void elementChanged(IElementChangeEvent event)
    {
        if (affects(event.getDelta(), (IHandle)getTreeViewer().getInput()))
        {
            final Control control = getTreeViewer().getControl();
            control.getDisplay().asyncExec(new Runnable()
            {
                public void run()
                {
                    if (!control.isDisposed())
                    {
                        refresh();
                    }
                }
            });
        }
    }

    private boolean affects(IHandleDelta delta, IHandle element)
    {
        if (delta.getElement().equals(element))
            return true;
        IHandleDelta[] children = delta.getAffectedChildren();
        for (IHandleDelta child : children)
        {
            if (affects(child, element))
                return true;
        }
        return false;
    }

    private Object computeInput()
    {
        IEditorInput editorInput = editor.getEditorInput();
        if (editorInput instanceof IFileEditorInput)
        {
            IFile file = ((IFileEditorInput)editorInput).getFile();
            return FooModelCore.create(file);
        }
        return null;
    }

    private void refresh()
    {
        Control control = getControl();
        control.setRedraw(false);
        BusyIndicator.showWhile(control.getDisplay(), new Runnable()
        {
            public void run()
            {
                TreePath[] treePaths =
                    getTreeViewer().getExpandedTreePaths();
                getTreeViewer().refresh();
                getTreeViewer().setExpandedTreePaths(treePaths);
            }
        });
        control.setRedraw(true);
    }
}
```

For this code to compile, we have to add `org.eclipse.ui.ide` to the list
of required bundles of the `org.eclipse.handly.examples.basic.ui` plug-in:

```
Require-Bundle: org.eclipse.handly.examples.basic,
 org.eclipse.handly,
 org.eclipse.handly.xtext.ui,
 org.eclipse.xtext.ui,
 org.eclipse.xtext.ui.shared,
 org.eclipse.ui.navigator,
 org.eclipse.ui.ide
```

The Outline page needs to be bound in the Xtext UI module for the language:

```java
// FooUiModule.java

    @Override
    public Class<? extends IContentOutlinePage> bindIContentOutlinePage()
    {
        return FooOutlinePage.class;
    }
```

Launch the runtime workbench and test what we have done so far.

Okay, it works, but it needs some *link-with-editor* functionality to become
really useful. Let's add it.

We will encapsulate this new functionality in the inner class `LinkingHelper`:

```java
// FooOutlinePage.java

    private LinkingHelper linkingHelper;

    @Override
    public void createControl(Composite parent)
    {
        super.createControl(parent);
        getTreeViewer().setContentProvider(contentProvider);
        getTreeViewer().setLabelProvider(labelProvider);
        getTreeViewer().setInput(computeInput());
        // new code -->
        linkingHelper = new LinkingHelper();
        // <-- new code
        editor.addPropertyListener(editorInputListener);
        FooModelCore.getFooModel().addElementChangeListener(this);
    }

    @Override
    public void dispose()
    {
        FooModelCore.getFooModel().removeElementChangeListener(this);
        editor.removePropertyListener(editorInputListener);
        // new code -->
        if (linkingHelper != null)
            linkingHelper.dispose();
        // <-- new code
        editor.outlinePageClosed();
        super.dispose();
    }
```

We are going to need a couple of utility methods for dealing with
source elements:

```java
// package org.eclipse.handly.internal.examples.basic.ui

/**
 * Utilities for <code>ISourceElement</code>s.
 * <p>
 * Note how this code is free from specifics of the Foo Model, 
 * thanks to the uniform API provided by Handly.
 * </p>
 */
public class SourceElementUtil
{
    /**
     * Returns the smallest element within the given source file 
     * that includes the given source position, or <code>null</code> 
     * if the given position is not within the text range of this 
     * source file, or if the source file does not exist or an 
     * exception occurs while accessing its corresponding resource. 
     * If no finer grained element is found at the position, 
     * the source file itself is returned.
     * <p>
     * As a side effect, reconciles the given source file.
     * </p>
     *
     * @param sourceFile the source file (not <code>null</code>)
     * @param position a source position inside the source file (0-based)
     * @return the innermost element enclosing the given source position, 
     *  or <code>null</code> if none (including the source file itself)
     */
    public static ISourceElement getSourceElement(ISourceFile sourceFile,
        int position)
    {
        try
        {
            sourceFile.reconcile(false, null);
        }
        catch (CoreException e)
        {
            Activator.log(e.getStatus());
            return null;
        }
        return sourceFile.getElementAt(position, null);
    }

    /**
     * Selects and reveals the identifying range of the given source element
     * in the given text editor. Returns <code>false</code> if the 
     * identifying range is not set or cannot be obtained (e.g., the 
     * element does not exist). 
     *
     * @param textEditor not <code>null</code>
     * @param element not <code>null</code>
     * @return <code>true</code> if the element was successfully revealed 
     *  in the editor; <code>false</code> otherwise
     */
    public static boolean revealInTextEditor(ITextEditor textEditor,
        ISourceElement element)
    {
        ISourceElementInfo info;
        try
        {
            info = element.getSourceElementInfo();
        }
        catch (CoreException e)
        {
            if (!element.exists())
                ; // this is considered normal
            else
                Activator.log(e.getStatus());
            return false;
        }
        TextRange identifyingRange = info.getIdentifyingRange();
        if (identifyingRange.isNull())
            return false;
        textEditor.selectAndReveal(identifyingRange.getOffset(),
            identifyingRange.getLength());
        return true;
    }

    private SourceElementUtil()
    {
    }
}
```

The class `LinkingHelper` will extend the Platform-provided class
`OpenAndLinkWithEditorHelper`:


```java
// FooOutlinePage.java

   private class LinkingHelper
        extends OpenAndLinkWithEditorHelper
    {
        private ISelectionChangedListener editorListener =
            new ISelectionChangedListener()
            {
                public void selectionChanged(SelectionChangedEvent event)
                {
                    if (!getTreeViewer().getControl().isFocusControl())
                    {
                        linkToOutline(event.getSelection());
                    }
                }
            };

        public LinkingHelper()
        {
            super(getTreeViewer());
            setLinkWithEditor(true);
            ISelectionProvider selectionProvider =
                editor.getSite().getSelectionProvider();
            if (selectionProvider instanceof IPostSelectionProvider)
                ((IPostSelectionProvider)selectionProvider).
                    addPostSelectionChangedListener(editorListener);
            else
                selectionProvider.addSelectionChangedListener(
                    editorListener);
        }

        @Override
        public void dispose()
        {
            ISelectionProvider selectionProvider =
                editor.getSite().getSelectionProvider();
            if (selectionProvider instanceof IPostSelectionProvider)
                ((IPostSelectionProvider)selectionProvider).
                    removePostSelectionChangedListener(editorListener);
            else
                selectionProvider.removeSelectionChangedListener(
                    editorListener);
            super.dispose();
        }

        @Override
        public void setLinkWithEditor(boolean enabled)
        {
            super.setLinkWithEditor(enabled);
            if (enabled)
                linkToOutline(
                    editor.getSite().getSelectionProvider().getSelection());
        }

        @Override
        protected void activate(ISelection selection)
        {
            linkToEditor(selection);
        }

        @Override
        protected void open(ISelection selection, boolean activate)
        {
            linkToEditor(selection);
        }

        @Override
        protected void linkToEditor(ISelection selection)
        {
            if (selection == null || selection.isEmpty())
                return;
            Object element =
                ((IStructuredSelection)selection).getFirstElement();
            if (!(element instanceof ISourceElement))
                return;
            SourceElementUtil.revealInTextEditor(editor,
                (ISourceElement)element);
        }

        @SuppressWarnings("unchecked")
        protected void linkToOutline(ISelection selection)
        {
            if (selection == null || selection.isEmpty())
                return;
            IStructuredSelection linkedSelection = null;
            if (selection instanceof ITextSelection)
                linkedSelection = getLinkedSelection(
                    (ITextSelection)selection);
            if (linkedSelection != null)
            {
                IStructuredSelection currentSelection =
                    (IStructuredSelection)getTreeViewer().getSelection();
                if (currentSelection == null
                    || !currentSelection.toList().containsAll(
                        linkedSelection.toList()))
                {
                    getTreeViewer().setSelection(linkedSelection, true);
                }
            }
        }

        private IStructuredSelection getLinkedSelection(
            ITextSelection selection)
        {
            Object input = getTreeViewer().getInput();
            if (!(input instanceof ISourceElement))
                return null;
            ISourceFile sourceFile = 
                ((ISourceElement)input).getSourceFile();
            ISourceElement element =
                SourceElementUtil.getSourceElement(sourceFile,
                    selection.getOffset());
            if (element == null)
                return null;
            return new StructuredSelection(element);
        }
    }
```

Basically, it listens to selection changes in the editor and updates
the outline's selection accordingly, and vice versa.

What's really interesting here is that the class `LinkingHelper` performs
its operations in terms of generic `ISourceElement`s -- it doesn't need
to depend on specific elements of the Foo model. Hence, it is a good
candidate for reuse, and could have been generalized and provided
as part of a class library, thanks to the uniform Handly API.

Launch the runtime workbench and test the newly added functionality.

## Closing Words

We would like to close this guide with a quote from the classic book
_Contributing to Eclipse_ by Erich Gamma and Kent Beck. It deeply
resonates with our intent:

> Overcoming this feeling of dislocation is the primary goal...
> When you are finished, you won't have a complete map, but you'll know
> at least one place to get each of your basic needs met. It's as if we draw
> you a map ... marked with six streets, a restaurant, and a hotel. You won't
> know everything, but you'll know enough to survive, and enough to learn more.

We hope that this tutorial did help you get started in a similar way.
You should know enough now to explore Handly and venture out on your own.
To dive deeper, let us point you to the project's [documentation page]
(https://wiki.eclipse.org/Handly) and, of course, [source code]
(http://git.eclipse.org/c/handly/org.eclipse.handly.git).

If you have found an error in the text or in a code example,
or would like to suggest an enhancement, please [raise an issue]
(https://github.com/pisv/gethandly/issues) against the
[guide's repository](https://github.com/pisv/gethandly).
We also accept pull requests. It is open source, after all! ;-)

If you could not find the answer to your question or would like to provide
feedback about this tutorial or about Handly in general, consider using the
[project's forum](http://eclipse.org/forums/eclipse.handly).

*Put Handly to work for you!*