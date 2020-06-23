# Step Zero: Setting Up

The next step will take you through the entire development process
of a Handly-based model. First, though, we need to set up the development
environment for our running example.

## Setting Up Eclipse

In our case, **Eclipse IDE for Java and DSL Developers** would be the best choice.
This tutorial assumes 2020-06 release. You can find a build for
your platform by visiting <https://www.eclipse.org/downloads/packages/>.

## Setting Up a Workspace

You may want to checkout example plug-in projects [*] from the
[Step Zero repository](https://github.com/pisv/gethandly.0) into your workspace
to use them as a basis for further development while following along
with instructions of the next step: [[Basic Model|Step One]]. Use the provided
`example.target` file in the `org.eclipse.handly.examples.basic` project
to set the target platform for the example.

**Note:** In general, there is a separate sub-module repository hosting
source code for a particular step of our running example. Each subsequent step
takes the source code of the preceding step and adds some more functionality.
In that way, you can always checkout the source code of the previous step
and use it as a starting point for development of the current step's
functionality.

<sub>[*] When importing example projects with the import wizard,
don't select the option __Search for nested projects__</sub>