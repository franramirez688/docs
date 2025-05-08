.. _incubating_workspaces:

Workspaces
==========

The workspaces feature can be enabled defining the environment variable ``CONAN_WORKSPACE_ENABLE=will_break_next``.
The value ``will_break_next`` is used to emphasize that it will change in next releases, and this feature is for testing only, it cannot be used in production.

Once the feature is enabled, workspaces are defined by the ``conanws.yml`` and/or ``conanws.py`` files.
By default, any Conan command will traverse up the file system from the current working directory to the filesystem root, until it finds one of those files. That will define the "root" workspace folder.

The ``conan workspace`` command allows to open, add, remove packages from the current workspace. Check the ``conan workspace -h`` help and the help of the subcommands to check their usage.

Dependencies added to a workspace work as local ``editable`` dependencies. They are only resolved as ``editable`` under the current workspace, if the current directory is moved outside of it, those ``editable`` dependencies won't be used anymore.

The paths in the ``conanws`` files are intended to be relative to be relocatable if necessary, or could be committed to Git in monorepo-like projects.


Workspace files syntax
++++++++++++++++++++++

The most basic implementation of a workspace is a ``conanws.yml`` file with just the definition of properties.
For example, a very basic workspace file that just defines the current CONAN_HOME to be a local folder would be:

.. code-block:: yaml
   :caption: conanws.yml

   home_folder: myhome


But a ``conanws.yml`` can be extended with a way more powerful ``conanws.py`` that follows the same relationship as a ``ConanFile`` does with its ``conandata.yml``, for example, it can dynamically
define the workspace home with:

.. code-block:: python
   :caption: conanws.py

   from conan import Workspace

   class MyWs(Workspace):

      def home_folder(self):
         # This reads the "conanws.yml" file, and returns "new_myhome"
         # as the current CONAN_HOME for this workspace
         return "new_" + self.conan_data["home_folder"]


So the command ``conan config home``:

.. code-block:: bash

   $ conan config home
   /path/to/ws/new_myhome

Will display as the current CONAN_HOME the ``new_myhome`` folder (by default it is relative
to the folder containing the ``conanws`` file)

Likewise, a workspace ``conanws.yml`` defining 2 editables could be:

.. code-block:: yaml
   :caption: conanws.yml

   editables:
      dep1/0.1:
         path: dep1
      dep2/0.1:
         path: dep2


But if we wanted to dynamically define the ``editables``, for example based on the
existence of some ``name.txt`` and ``version.txt`` files in folders, the editables
could be defined in ``conanws.py`` as:

.. code-block:: python
   :caption: conanws.py

   import os
   from conan import Workspace

   class MyWorkspace(Workspace):

      def editables(self):
         result = {}
         for f in os.listdir(self.folder):
            if os.path.isdir(os.path.join(self.folder, f)):
               with open(os.path.join(self.folder, f, "name.txt")) as fname:
                  name = fname.read().strip()
               with open(os.path.join(self.folder, f, "version.txt")) as fversion:
                  version = fversion.read().strip()
               result[f"{name}/{version}"] = {"path": f}
         return result


It is also possible to re-use the ``conanfile.py`` logic in ``set_name()`` and ``set_version()``
methods, using the ``Workspace.load_conanfile()`` helper:

.. code-block:: python
   :caption: conanws.py

   import os
   from conan import Workspace

   class MyWorkspace(Workspace):
      def editables(self):
         result = {}
         for f in os.listdir(self.folder):
            if os.path.isdir(os.path.join(self.folder, f)):
               conanfile = self.load_conanfile(f)
               result[f"{conanfile.name}/{conanfile.version}"] = {"path": f}
         return result


Workspace commands
++++++++++++++++++

conan workspace add/remove
**************************

Use these commands to add or remove editable packages to the current workspace. The ``conan workspace add <path>`` folder must contain a ``conanfile.py``.

The ``conanws.py`` has a default implementation, but it is possible to override the default behavior:

.. code-block:: python
   :caption: conanws.py

   import os
   from conan import Workspace

   class MyWorkspace(Workspace):
      def name(self):
         return "myws"

      def add(self, ref, path, *args, **kwargs):
         self.output.info(f"Adding {ref} at {path}")
         super().add(ref, path, *args, **kwargs)

      def remove(self, path, *args, **kwargs):
         self.output.info(f"Removing {path}")
         return super().remove(path, *args, **kwargs)


conan workspace info
********************

Use this command to show information about the current workspace

.. code-block:: bash

   $ cd myfolder
   $ conan new workspace
   $ conan workspace info
   WARN: Workspace found
   WARN: Workspace is a dev-only feature, exclusively for testing
   name: myfolder
   folder: /path/to/myfolder
   products
      app1
   editables
      liba/0.1
         path: liba
      libb/0.1
         path: libb
      app1/0.1
         path: app1


conan workspace open
********************

The new ``conan workspace open`` command implements a new concept. Those packages containing an ``scm`` information in the ``conandata.yml`` (with ``git.coordinates_to_conandata()``) can be automatically cloned and checkout inside the current workspace from their Conan recipe reference (including recipe revision).


conan new workspace
*******************

The command ``conan new`` has learned a new built-in (experimental) template ``workspace`` that creates a local project with some editable packages
and a ``conanws.yml`` that represents it. It is useful for quick demos, proofs of concepts and experimentation.


conan workspace build
*********************

The command ``conan workspace build`` does the equivalent of ``conan build <product-path> --build=editable``, for every ``product`` defined
in the workspace.

Products are the "downstream" consumers, the "root" and starting node of dependency graphs. They can be defined with the ``conan workspace add <folder> --product``
new ``--product`` argument.

The ``conan workspace build`` command just iterates all products, so it might repeat the build of editables dependencies of the products. In most cases, it will be a no-op as the projects would be already built, but might still take some time. This is pending for optimization, but that will be done later, the important thing now is to focus on tools, UX, flows, and definitions (of things like the ``products``).


conan workspace install
***********************

The command ``conan workspace install`` is useful to install and build the current workspace
as a monolithic super-project of the editables. See next section.

By default it uses all the ``editable`` packages in the workspace. It is possible to select
only a subset of them with the ``conan workspace install <folder1> .. <folderN>`` optional
arguments. Only the subgraph of those packages, incluing their dependencies and transitive
dependencies will be installed.


Workspace monolithic builds
+++++++++++++++++++++++++++

Conan workspaces can be built as a single monolithic project (sometimes called super-project),
which can be very convenient. Let's see it with an example:

.. code-block:: bash

   $ conan new workspace
   $ conan workspace install
   $ cmake --preset conan-release # use conan-default in Win
   $ cmake --build --preset conan-release

Let's explain a bit what happened.
First the ``conan new workspace`` created a template project with some relevant files:

The ``CMakeLists.txt`` defines the super-project with:

.. code-block:: cmake
   :caption: CMakeLists.txt

   cmake_minimum_required(VERSION 3.25)
   project(monorepo CXX)

   include(FetchContent)

   function(add_project SUBFOLDER)
      FetchContent_Declare(
         ${SUBFOLDER}
         SOURCE_DIR ${CMAKE_CURRENT_LIST_DIR}/${SUBFOLDER}
         SYSTEM
         OVERRIDE_FIND_PACKAGE
      )
      FetchContent_MakeAvailable(${SUBFOLDER})
   endfunction()

   add_project(liba)
   # They should be defined in the liba/CMakeLists.txt, but we can fix it here
   add_library(liba::liba ALIAS liba)
   add_project(libb)
   add_library(libb::libb ALIAS libb)
   add_project(app1)

So basically, the super-project uses ``FetchContent`` to add the subfolders sub-projects.
For this to work correctly, the subprojects must be CMake based sub projects with
``CMakeLists.txt``. Also, the subprojects must define the correct targets as would be
defined by the ``find_package()`` scripts, like ``liba::liba``. If this is not the case,
it is always possible to define some local ``ALIAS`` targets.

The other important part is the ``conanws.py`` file:


.. code-block:: python
   :caption: conanws.py

   from conan import Workspace
   from conan import ConanFile
   from conan.tools.cmake import CMakeDeps, CMakeToolchain, cmake_layout

   class MyWs(ConanFile):
      """ This is a special conanfile, used only for workspace definition of layout
      and generators. It shouldn't have requirements, tool_requirements. It shouldn't have
      build() or package() methods
      """
      settings = "os", "compiler", "build_type", "arch"

      def generate(self):
         deps = CMakeDeps(self)
         deps.generate()
         tc = CMakeToolchain(self)
         tc.generate()

      def layout(self):
         cmake_layout(self)

   class Ws(Workspace):
      def root_conanfile(self):
         return MyWs  # Note this is the class name


The role of the ``class MyWs(ConanFile)`` embedded conanfile is important, it defines
the super-project necessary generators and layout.

The ``conan workspace install`` does not install the different editables separately, for
this command, the editables do not exist, they are just treated as a single "node" in
the dependency graph, as they will be part of the super-project build. So there is only
a single generated ``conan_toolchain.cmake`` and a single common set of dependencies
``xxx-config.cmake`` files for all super-project external dependencies.


The template above worked without external dependencies, but everything would work
the same when there are external dependencies. This can be tested with:

.. code-block:: bash

   $ conan new cmake_lib -d name=mymath
   $ conan create .
   $ conan new workspace -d requires=mymath/0.1
   $ conan workspace install
   $ cmake ...


.. note::

   The current ``conan new workspace`` generates a CMake based super project.
   But it is possible to define a super-project using other build systems, like a
   MSBuild solution file that adds the different ``.vcxproj`` subprojects. As long as
   the super-project knows how to aggregate and manage the sub-projects, this is possible.

   It might also be possible for the ``add()`` method in the ``conanws.py`` to manage the
   addition of the subprojects to the super-project, if there is some structure.


For any feedback, please open new tickets in https://github.com/conan-io/conan/issues.
