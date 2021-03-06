User Libraries
==============

*Integration Service* defines three types of libraries, *Transformation Library*, *Bridge Library*,
and *Types Library*. The users can implement all of them.
The same library can mix all these libraries, including adding several libraries of the same kind
(`FIROS2's TIS_NGSIv2 example <https://github.com/eProsima/FIROS2/tree/master/examples/TIS_NGSIv2>`__ mixes
several transformation libraries in the same file).

See :ref:`Configuration format` for more information about how to indicate *Integration Service* which libraries to use.

* :ref:`Transformation Library`: Allows creating custom data transformations.

* :ref:`Bridge Library`: Allows implementing new *Writers* and *Readers* to communicate with other protocols.

* :ref:`Types Library`: Allows defining new *TopicDataTypes*.


Transformation Library
----------------------

*Integration Service* allows defining custom *transformation functions* that a :ref:`connector` will apply.
*Transformation functions* are static functions that receive the input data,
apply some transformation and store the result in the output data.
The *connector* will be configured with the library and the function to call in each case.
There is a static data prototype in
`resource/templatelib.cpp <https://github.com/eProsima/Integration-Service/blob/master/resource/templatelib.cpp>`__:

These functions must have one of the following interfaces:

Static Data
^^^^^^^^^^^

.. literalinclude:: technical.cpp
    :language: cpp
    :start-after: // Static Data Start
    :end-before: // Static Data End


Dynamic Data
^^^^^^^^^^^^

.. literalinclude:: technical.cpp
    :language: cpp
    :start-after: // Dynamic Data Start
    :end-before: // Dynamic Data End

In both cases, the transformation function parses the :class:`inputData`,
modifies it at will and stores the result into :class:`outputData`.
See the :ref:`Dynamic Types example` for some already working implementations.

Bridge Library
--------------

*Bridge libraries* are used to integrate new communication protocols.
They must offer the following function declarations:

* **create_bridge**:

.. literalinclude:: technical.cpp
    :language: cpp
    :start-after: // Create Bridge Start
    :end-before: // Create Bridge End

As shown, the instantiated *bridge* must implement :ref:`isbridge`.
``ISBridges`` are in charge of communicating *readers* with *writers* and apply *transformation functions* as defined in
the :ref:`connector`.

* **create_reader**:

.. literalinclude:: technical.cpp
    :language: cpp
    :start-after: // Create Reader Start
    :end-before: // Create Reader End

The *reader* returned must implement :ref:`isreader`.
``ISReaders`` must be able to receive data from the input protocol.


* **create_writer**:

.. literalinclude:: technical.cpp
    :language: cpp
    :start-after: // Create Writer Start
    :end-before: // Create Writer End

The *writer* returned must implement :ref:`iswriter`.
``ISWriters`` must be able to send data to the destination protocol.


When the bridge stops, *Integration Service* will deallocate these objects from memory.
If a node in the XML configuration file has at least one property, a vector of pairs of strings will be provided
to the corresponding function (see :ref:`Bridge configuration` for more information).

If some functions want to use the default implementation (*RTPS-Bridge*), they must return :class:`nullptr`.
See :ref:`Integration Service architecture` section for more information about the interfaces that any *Bridge Library*
must implement.

The responsibility of how to instantiate the *bridge*, *writer*, and *reader* is on the *Bridge Library*,
but it's important to remark that "RTPS" *publishers* and *subscribers* will be filled automatically by ``ISManager``
with the configuration from the ``<participant>`` node of the :ref:`Fast-RTPS profiles`.
See the :ref:`Adding new Bridges` section for some already working implementations.

ISBridge
^^^^^^^^

This component must communicate ``ISReaders`` with ``ISWriters``, applying the
:ref:`transformation functions <Transformation Library>` if any and its default implementation must be enough for the majority of cases.
It can be seen as a **connector manager** as it is responsible for applying the data flow and the logic of each connector.
A bridge can manage several *connectors*, and it should reuse readers, transformation functions, and writers if it's
possible. In complex configurations, like in this :ref:`example`, several connectors can share the same readers,
transformation functions, and writers.

Custom ``ISBridges`` must inherit from it:

.. literalinclude:: technical.cpp
    :language: cpp
    :start-after: // Class Bridge Start
    :end-before: // Class Bridge End

:class:`ISBridge.h` and :class:`ISBridge.cpp` implement the default behavior.
There is no need to implement any function from any
subclass but, if needed all these methods can be implemented. In that case, the implementation should be done
carefully with the full functionality.
It's recommended to copy the standard implementation and modify it with your needs.
After that, remove the unmodified methods.
:class:`addFunction` and :class:`on_received_data` methods have two flavors, with static and dynamic data.

RTPS-Bridge
^^^^^^^^^^^

*Integration Service* has a default built-in *RTPS-Bridge*, but it allows creating custom bridges
to connect new protocols implementing *bridge libraries*.

It implements a full ``ISBridge`` using *Fast-RTPS* *publisher* and *subscriber*.
It allows communicating several *subscribers* with several *publishers*, establishing routes and applying
:ref:`transformation functions <Transformation Library>` depending on each *connector* configuration.

The *connector* :ref:`RTPS Connector` uses this kind of bridge.

ISWriter
^^^^^^^^

This component must be able to write data to the destination protocol. The default implementation uses a *Fast-RTPS
publisher*.

.. literalinclude:: technical.cpp
    :language: cpp
    :start-after: // Class Writer Start
    :end-before: // Class Writer End

``ISWriter`` doesn't have a default implementation, so the built-in *RTPS-Bridge* provides the default behavior.
Any custom *bridge* that needs to define its *writer* must implement at least both :class:`write` methods.
If one of them isn't needed, implement it as follows:

.. literalinclude:: technical.cpp
    :language: cpp
    :start-after: // Writer Write Start
    :end-before: // Writer Write End

The user must be sure that this method's version never will be called.

ISReader
^^^^^^^^

This component is in charge of receive data from the input protocol. Its default implementation uses a *Fast-RTPS
subscriber*.

.. literalinclude:: technical.cpp
    :language: cpp
    :start-after: // Class Reader Start
    :end-before: // Class Reader End

``ISReader`` doesn't have a default implementation, so the built-in *RTPS-Bridge* provides the default behavior.
Any custom *bridge* that needs to define its *reader* must implement their reading method that must call at
least one of the :class:`on_received_data` methods.

.. literalinclude:: technical.cpp
    :language: cpp
    :start-after: // Protocol reader start
    :end-before: // Protocol reader end

If one of them isn't needed, implement it as follows:

.. literalinclude:: technical.cpp
    :language: cpp
    :start-after: // Reader Read Start
    :end-before: // Reader Read End


Types Library
-------------

*Integration Service* allows defining types libraries to create custom data types.
These libraries must offer a function with the following declaration:

.. literalinclude:: technical.cpp
    :language: cpp
    :start-after: // Topic Type 1 Start
    :end-before: // Topic Type 1 End

It receives the ``name`` of the TopicType and must return an instance of it
(subclass of *Fast RTPS's* :class:`TopicDataType`).
If the provided type is unknown, the function must return :class:`nullptr`.

.. literalinclude:: technical.cpp
    :language: cpp
    :start-after: // Topic Type 2 Start
    :end-before: // Topic Type 2 End

The returned type can be built using `Fast-RTPS dynamic types <http://docs.eprosima.com/en/latest/dynamictypes.html>`__,
using an already generated IDL statically or implementing it directly as a :class:`TopicDataType` subclass.

.. literalinclude:: technical.cpp
    :language: cpp
    :start-after: // Topic Type 3 Start
    :end-before: // Topic Type 3 End

In section :ref:`Dynamic Data Integration` you can find an already working example.