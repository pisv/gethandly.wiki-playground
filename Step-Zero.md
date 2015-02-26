# Step Zero: Setting Up

The next step will take you through the entire development process
of a Handly-based model. First, though, we need to set up the development
environment for our running example.

## Setting Up Eclipse

To use the examples, **Eclipse IDE for Java and DSL Developers** is required.
You can find a build for your platform by visiting
https://www.eclipse.org/downloads/.

You also need to add **Handly** into the Eclipse installation. This can be
done by installing all features from a Handly update site. Please visit
the project's [Downloads page](https://projects.eclipse.org/projects/technology.handly/downloads)
to find the URL of the update site for the latest release of Handly.

**_ATTENTION_**: *Not every version of Xtext is compatible with a particular
Handly release. Please ensure that you have a right version of Eclipse IDE
for Java and DSL Developers according to the following table:*

| Handly | Xtext | Eclipse IDE for Java and DSL Developers |
| ------ | ----- | --------------------------------------- |
| 0.2.x  | 2.7.x | Luna SR1 (4.4.1) or Luna SR2 (4.4.2)    |
| 0.1.x  | 2.6.x | Luna (4.4.0) or Kepler SR2 (4.3.2)      |

## Setting Up a Workspace

You may want to checkout example plug-in projects [*] from the
[Step Zero repository](https://github.com/pisv/gethandly.0) into your workspace
to use them as a basis for further development while following along
with instructions of the next step: [[Basic Model|Step One]].

**Note:** In general, there is a separate sub-module repository hosting
source code for a particular step of our running example. Each subsequent step
takes the source code of the preceding step and adds some more functionality.
In that way, you can always checkout the source code of the previous step
and use it as a starting point for development of the current step's
functionality.

<sub>[*] When importing example projects with the import wizard,
don't select the option __Search for nested projects__</sub>