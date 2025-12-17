GoLand Plugin
=============

Introduction
------------

The Juice GoLand Plugin is an IDE tool specifically designed for the Juice framework. It significantly improves development efficiency and user experience by providing features like intelligent navigation and auto-completion, allowing developers to switch easily between XML configuration and Go code.

Preview
-------

.. figure:: media/output.gif
   :alt: Preview
   :width: 800px
   :align: center

Installation
------------

There are two ways to install the Juice GoLand Plugin:

1. Install via IDE Plugin Marketplace
   - Open GoLand IDE
   - Go to Settings/Preferences -> Plugins
   - Switch to the Marketplace tab
   - Search for "Juice"
   - Click the Install button

2. Manual Installation
   - Download the latest version of the plugin package (.zip file) from the GitHub Release page
   - Open GoLand IDE
   - Go to Settings/Preferences -> Plugins
   - Click the gear icon and select "Install Plugin from Disk"
   - Select the downloaded plugin package to install

Main Features
-------------

1. Intelligent Navigation
   - XML to Go Interface Jump: Ctrl/Cmd + Click on the interface name in the XML file to jump to the corresponding Go interface definition.
   - Go Interface to XML Reverse Jump: Use Alt + B or the right-click menu at the Go interface definition to jump to the corresponding XML configuration.

2. Auto-Completion
   - XML Namespace Auto-Completion: Automatically hints and completes namespaces while typing.
   - Interface Method Completion: Provides intelligent hints and completion when writing methods in XML.
   - Parameter Name Completion: Automatically completes method parameter names.

3. Syntax Highlighting
   - SQL syntax highlighting in XML files.
   - Syntax highlighting for configuration tags and attributes.
   - Error syntax hints.

4. Code Inspection
   - XML configuration file syntax check.
   - Interface method signature matching check.
   - Namespace correctness verification.

Usage Examples
--------------

1. XML to Go Interface Jump Example:

.. code-block:: xml

    <mapper namespace="github.com.example.repo.UserMapper">
        <!-- Ctrl/Cmd + Click on the namespace value to jump to the UserMapper interface definition -->
    </mapper>

2. Namespace Auto-Completion:

.. code-block:: xml

    <mapper namespace="github.com.example.repo.UserM"> <!-- Automatically suggests available namespaces while typing -->

3. SQL Syntax Highlighting:

.. code-block:: xml

    <select id="getUserById">
        SELECT * FROM users WHERE id = #{id} <!-- SQL statement syntax highlighting -->
    </select>

Shortcuts
---------

- Ctrl/Cmd + Click: Jump to definition
- Alt + B: Find usages / jump to XML
- Ctrl/Cmd + Space: Trigger auto-completion
- Alt + Enter: Show intention actions and quick fixes

FAQ
---

1. Plugin installation failed
   - Ensure GoLand version compatibility (supports 2023.1 and above).
   - Check if network connection is normal.
   - Try the manual installation method.

2. Navigation function not working
   - Ensure the project has GOPATH configured correctly.
   - Check if the namespace in the XML file is correct.
   - Ensure the Go interface file has been imported correctly.

3. Auto-completion not working
   - Check if auto-completion functionality is enabled.
   - Ensure project indexing is complete.
   - Try rebuilding the project index.

Feedback & Support
------------------

If you encounter any issues or have suggestions for improvement during use, please feel free to provide feedback via:

- Submit an Issue on the GitHub project
- Feedback through the official documentation comments section
- Send an email to the support mailbox
