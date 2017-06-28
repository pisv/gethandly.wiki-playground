# Step Four: Working Copy

In [[Step Three]] we have built a Navigator view for our Handly-based model.
It works fine, except that it cannot display the outline of the editor's
current contents for a Foo file -- it only "sees" the saved contents of the
file. In this step we will employ the so-called working copy facility to make
that view even more alive. We will also use the resulting infrastructure
to build an Outline page for the Foo editor -- another view of our model.
The complete source code for this step of the running example is available
in the [Step Four repository](https://github.com/pisv/gethandly.4th).

In Handly, a source file (as a model element) can be switched into a
*working copy* mode. Switching to working copy means that the source file's
structure and properties shall no longer correspond to the file's saved contents.
Instead, those structure and properties will reflect the current contents
of the source file's editor. To enable this, the editor needs to be integrated
with Handly.

Fortunately, our Foo language is Xtext-based, and Handly provides an
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

    public Class<? extends IInputElementProvider> bindIInputElementProvider()
    {
        return FooInputElementProvider.class;
    }
```

The only missing piece is the `FooInputElementProvider`. This class
should implement the interface `IInputElementProvider` and provide `IElement`
corresponding to the given `IEditorInput`:

```java
// package org.eclipse.handly.internal.examples.basic.ui

/**
 * Implementation of {@link IInputElementProvider} to be bound
 * in the Xtext UI module for the language.
 */
@Singleton
public class FooInputElementProvider
    implements IInputElementProvider
{
    @Override
    public IElement getElement(IEditorInput input)
    {
        if (input == null)
            return null;
        IFile file = (IFile)input.getAdapter(IFile.class);
        return FooModelCore.create(file);
    }
}
```
For this code to compile, we have to add `org.eclipse.handly.ui` to the list
of required bundles of the `org.eclipse.handly.examples.basic.ui` plug-in.

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

To enable notifications about working copy changes, we just need to register
the notification manager in the *model context*:

```java
// FooModelManager.java

    private Context modelContext;

    public void startup() throws Exception
    {
        fooModel = new FooModel();
        elementManager = new ElementManager(new FooModelCache());
        notificationManager = new NotificationManager();
        // new code -->
        modelContext = new Context();
        modelContext.bind(INotificationManager.class).to(
            notificationManager);
        // <-- new code
        fooModel.getWorkspace().addResourceChangeListener(this,
            IResourceChangeEvent.POST_CHANGE);
    }

    public void shutdown() throws Exception
    {
        fooModel.getWorkspace().removeResourceChangeListener(this);
        // new code -->
        modelContext = null;
        // <-- new code
        notificationManager = null;
        elementManager = null;
        fooModel = null;
    }

    public IContext getModelContext()
    {
        if (modelContext == null)
            throw new IllegalStateException();
        return modelContext;
    }
```

and expose the model context in `FooModel`:

```java
// FooModel.java

    @Override
    public IContext getModelContext()
    {
        return FooModelManager.INSTANCE.getModelContext();
    }
```

That will make the notification manager accessible to the generic working copy
change notification facility in the class `SourceFile` (for details, see the
`hWorkingCopyModeChanged` method and the nested class `NotifyingReconcileOperation`).

And while we are here, let's also handle the working copy as a special case
in the `FooDeltaProcessor`:

```java
// FooDeltaProcessor.java

    private void contentChanged(IFooFile fooFile)
    {
        // new code -->
        if (fooFile.isWorkingCopy())
        {
            builder.changed(fooFile, IElementDeltaConstants.F_CONTENT
                | IElementDeltaConstants.F_UNDERLYING_RESOURCE);
            return;
        }
        // <-- new code

        close(fooFile);
        builder.changed(fooFile, IElementDeltaConstants.F_CONTENT);
    }
```

Everything should work fine now.

## Other Editors

Besides providing out-of-the-box integration with the Xtext editor,
Handly can also support other editors. In particular, it allows for
integration with TextFileBuffer-based editors. The following notes are
intended as forward pointers only; a full-blown editor implementation
is beyond the scope of this tutorial.

To integrate a TextFileBuffer-based editor with the working copy facility,
you'll need to subclass the editor's document provider from the class
`SourceFileDocumentProvider`, which extends the `TextFileDocumentProvider`.

For example:

```java
public class FooFileDocumentProvider
    extends SourceFileDocumentProvider
{
    @Override
    protected ISourceFile getSourceFile(Object element)
    {
        if (!(element instanceof IEditorInput))
            return null;
        IInputElementProvider provider =
            Activator.getDefault().getFooInputElementProvider();
        element = provider.getElement((IEditorInput)element);
        if (!(element instanceof ISourceFile))
            return null;
        return (ISourceFile)element;
    }
}
```

You'll also need to implement a Handly-based reconciler for the editor:

```java
public class FooReconciler
    extends EditorWorkingCopyReconciler
{
    public FooReconciler(IEditorPart editor, IWorkingCopyManager manager)
    {
        super(editor, manager);
    }

    @Override
    protected void addElementChangeListener(IElementChangeListener listener)
    {
        FooModelCore.getFooModel().addElementChangeListener(listener);
    }

    @Override
    protected void removeElementChangeListener(IElementChangeListener listener)
    {
        FooModelCore.getFooModel().removeElementChangeListener(listener);
    }
}
````

The reconciler is connected to the editor via a source viewer configuration:

```java
public class FooSourceViewerConfiguration
    extends TextSourceViewerConfiguration
{
    private final ITextEditor editor;
    private final IWorkingCopyManager manager;

    public FooSourceViewerConfiguration(IPreferenceStore preferenceStore,
        ITextEditor editor, IWorkingCopyManager manager)
    {
        super(preferenceStore);
        this.editor = editor;
        this.manager = manager;
    }

    @Override
    public IReconciler getReconciler(ISourceViewer sourceViewer)
    {
        if (editor != null && editor.isEditable())
        {
            return new FooReconciler(editor, manager);
        }
        return null;
    }
}
```

Then, you can wire it all together in the editor:

```java
public class FooEditor
    extends AbstractDecoratedTextEditor
{
    @Override
    protected void initializeEditor()
    {
        super.initializeEditor();
        FooFileDocumentProvider provider =
            Activator.getDefault().getFooFileDocumentProvider();
        setDocumentProvider(provider);
        setSourceViewerConfiguration(new FooSourceViewerConfiguration(
            getPreferenceStore(), this, provider));
    }
}
```

## Outline View

Now that our model has truly come to life with all the infrastructure
in place, it should be very easy to create a Foo Outline page using
the Common Outline framework provided by Handly. The outline will display
the same kind of elements as the **Foo Navigator** and can reuse existing
content and label providers. Basically, we've got a model and would like
to have a number of views on it.

```java
public class FooOutlinePage
    extends HandlyXtextOutlinePage
{
    @Inject
    private FooContentProvider contentProvider;
    @Inject
    private FooLabelProvider labelProvider;

    @Override
    protected ITreeContentProvider getContentProvider()
    {
        return contentProvider;
    }

    @Override
    protected IBaseLabelProvider getLabelProvider()
    {
        return labelProvider;
    }

    @Override
    protected void addElementChangeListener(IElementChangeListener listener)
    {
        FooModelCore.getFooModel().addElementChangeListener(listener);
    }

    @Override
    protected void removeElementChangeListener(IElementChangeListener listener)
    {
        FooModelCore.getFooModel().removeElementChangeListener(listener);
    }
}
```

The code should be self-explanatory. The superclass `HandlyXtextOutlinePage`
provides rich outline functionality out-of-the-box, including 'Link with Editor'
and 'Sort'. The outline will refresh itself when its contents is affected
by a change in the Foo model. It will also reset its input when the
corresponding editor input changes.

The Outline page needs to be bound in the Xtext UI module for the language:

```java
// FooUiModule.java

    @Override
    public Class<? extends IContentOutlinePage> bindIContentOutlinePage()
    {
        return FooOutlinePage.class;
    }
```

Launch the runtime workbench and test the newly added functionality.

## Closing Words

We would like to close this guide with a quote from the classic book
*Contributing to Eclipse* by Erich Gamma and Kent Beck. It deeply
resonates with our intent:

> Overcoming this feeling of dislocation is the primary goal...
> When you are finished, you won't have a complete map, but you'll know
> at least one place to get each of your basic needs met. It's as if we draw
> you a map ... marked with six streets, a restaurant, and a hotel. You won't
> know everything, but you'll know enough to survive, and enough to learn more.

We hope this tutorial will help you get started in a similar way.

You should know enough now to explore Handly and venture out on your own.
To dive deeper, let us point you to the project's [web site](https://eclipse.org/handly/)
and, of course, [source code](https://projects.eclipse.org/projects/technology.handly/developer).
In particular, you might be interested in a more advanced exemplary
implementation that Handly provides: example model for Java
(`org.eclipse.handly.examples.javamodel`). There is also a concise
[architectural overview](http://www.eclipse.org/downloads/download.php?file=/handly/docs/handly-overview.pdf&r=1)
of the core framework that can be used as a quick refresher of the concepts
learnt from this guide. Last but not least we have created an
[experimental fork of Eclipse Java development tools](https://github.com/pisv/jdt.core-handly),
which can serve as an example of Handly usage within the context of a
non-trivial existing model implementation.

If you have found an error in the text or in a code example,
or would like to suggest an enhancement, please
[raise an issue](https://github.com/pisv/gethandly/issues) against the
[guide's repository](https://github.com/pisv/gethandly).
We also accept pull requests. It is open source, after all! ;-)

If you could not find an answer to your question or would like to provide
feedback about this tutorial or about Handly in general, consider using the
project's [forum](http://eclipse.org/forums/eclipse.handly) or
[mailing list](https://dev.eclipse.org/mailman/listinfo/handly-dev).

*Put Handly to work for you!*