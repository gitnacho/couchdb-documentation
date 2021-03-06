Configuration
=============

Bootstrapping
-------------

Bootstrapping Doctrine is a relatively simple procedure that
roughly exists of just 2 steps:


-  Making sure Doctrine class files can be loaded on demand.
-  Obtaining a DocumentManager instance.

Class Loading
-------------

Doctrine CouchDB has dependencies on two other libraries:

-  Doctrine\Common
-  Symfony\Component\Console

You have to make sure that both dependencies are installed and autoloadable.

The Github checkout of comes with a submodule of the Doctrine Common library. It contains
the ``Doctrine\Common\ClassLoader`` which should be used for autoloading all the necessary
Doctrine classes.

.. code-block:: php

    <?php
    $couchPath = "path/lib";
    require_once $couchPath . "vendor/doctrine-common/lib/Doctrine/Common/ClassLoader.php";

    $loader = new \Doctrine\Common\ClassLoader("Doctrine\Common", $couchPath . "vendor/doctrine-common/lib");
    $loader->register();

    $loader = new \Doctrine\Common\ClassLoader("Doctrine\ODM\CouchDB", $couchPath);
    $loader->register();

    $loader = new \Doctrine\Common\ClassLoader("Symfony", $couchPath."/vendor");
    $loader->register();

If Doctrine Common is installed via PEAR the ClassLoader can be loaded
from the include path:

.. code-block:: php

    <?php
    require_once "Doctrine/Common/ClassLoader.php";

    $loader = new \Doctrine\Common\ClassLoader("Doctrine\Common");
    $loader->register();

Obtaining the DocumentManager
-----------------------------

To obtain the DocumentManager you have to start setting up a CouchDB configuration object.
See this example:

.. code-block:: php

    <?php
    $database = "project_database_name";
    $documentPaths = array("MyProject\Documents");
    $couchClient = \Doctrine\CouchDB\CouchDBClient::create(array('dbname' => $database));

    $config = new \Doctrine\ODM\CouchDB\Configuration();
    $metadataDriver = $config->newDefaultAnnotationDriver($documentPaths);

    $config->setProxyDir(__DIR__ . "/proxies");
    $config->setMetadataDriverImpl($metadataDriver);
    $config->setLuceneHandlerName('_fti');

    $dm = \Doctrine\ODM\CouchDB\DocumentManager::create($couchClient, $config);

CouchDBClient
-------------

You can create a CouchDBClient with a factory method or just by
constructing a new instance. The factory method accepts an array of configuration
parameters and applies a set of defaults. The constructor requires
an instantiated HTTP Client and a database name.

Database Name (***REQUIRED***)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

You have to specify the name of the CouchDB database to use
with Doctrine CouchDB.

.. code-block:: php

    <?php
    $couchClient = CouchClient::create(array('dbname' => 'test_database'));
    echo $couchClient->getDatabase();

HTTP Client (***OPTIONAL***)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: php

    <?php
    $couchClient = CouchClient::create(array('dbname' => 'test_database'));
    $client = $couchClient->getHttpClient();

There are two different HTTP Clients shipped with Doctrine CouchDB:

-   ``Doctrine\ODM\CouchDB\HTTP\SocketClient`` The default client uses fsocketopen and
    has very good performance using keep alive connections.
-   ``Doctrine\ODM\CouchDB\HTTP\StreamClient`` Uses fopen and is therefore simpler than the SocketClient,
    however cannot use keep alive. In some PHP setups the SocketClient doesn't work and the StreamClient
    is a fallback for these situations.

You can pass the following options to configure the HTTP Client:

-   host (default localhost)
-   port (default 5984)
-   user (default null)
-   password (default null)
-   ip (default null)
-   logging (default false)

Configuration Options
---------------------

The following sections describe all the configuration options
available on a ``Doctrine\ODM\CouchDB\Configuration`` instance.

Proxy Directory (***REQUIRED***)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: php

    <?php
    $config->setProxyDir($dir);
    $config->getProxyDir();

Gets or sets the directory where Doctrine generates any proxy
classes. For a detailed explanation on proxy classes and how they
are used in Doctrine, refer to the "Proxy Objects" section further
down.

Proxy Namespace (***OPTIONAL***)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: php

    <?php
    $config->setProxyNamespace($namespace);
    $config->getProxyNamespace();

Gets or sets the namespace to use for generated proxy classes. For
a detailed explanation on proxy classes and how they are used in
Doctrine, refer to the "Proxy Objects" section further down.

Metadata Driver (***REQUIRED***)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: php

    <?php
    $config->setMetadataDriverImpl($driver);
    $config->getMetadataDriverImpl();

Gets or sets the metadata driver implementation that is used by
Doctrine to acquire the object-relational metadata for your
classes.

There are currently one working available implementation:


-  ``Doctrine\ODM\CouchDB\Mapping\Driver\AnnotationDriver``

Throughout the most part of this manual the AnnotationDriver is
used in the examples. For information on the usage of the other drivers
please refer to the dedicated chapters.

The annotation driver can be configured with a factory method on
the ``Doctrine\ODM\CouchDB\Configuration``:

.. code-block:: php

    <?php
    $driverImpl = $config->newDefaultAnnotationDriver(array('/path/to/lib/MyProject/Documents'));
    $config->setMetadataDriverImpl($driverImpl);

The path information to the documents is required for the annotation
driver, because otherwise mass-operations on all entities through
the console could not work correctly. All of metadata drivers
accept either a single directory as a string or an array of
directories. With this feature a single driver can support multiple
directories of documents.

Metadata Cache (***RECOMMENDED***)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: php

    <?php
    $config->setMetadataCacheImpl($cache);
    $config->getMetadataCacheImpl();

Gets or sets the cache implementation to use for caching metadata
information, that is, all the information you supply via
annotations, xml or yaml, so that they do not need to be parsed and
loaded from scratch on every single request which is a waste of
resources. The cache implementation must implement the
``Doctrine\Common\Cache\Cache`` interface.

Usage of a metadata cache is highly recommended.

The recommended implementations for production are:


-  ``Doctrine\Common\Cache\ApcCache``
-  ``Doctrine\Common\Cache\MemcacheCache``
-  ``Doctrine\Common\Cache\XcacheCache``

For development you should use the
``Doctrine\Common\Cache\ArrayCache`` which only caches data on a
per-request basis.

Lucene Handler Name (***OPTIONAL***)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: php

    <?php
    $config->setLuceneHandlerName($handlerName);
    $config->getLuceneHandlerName();

The default CouchDB Lucene handler is named "_fti", but it might be named differently in your
setup. You can rename this handler name with this option. You have to set this option
to "_fti", without setting this option it is supposed that CouchDB Lucene is not installed.

Proxy Objects
-------------

A proxy object is an object that is put in place or used instead of
the "real" object. A proxy object can add behavior to the object
being proxied without that object being aware of it. In Doctrine CouchDB,
proxy objects are used to realize several features but mainly for
transparent lazy-loading.

Proxy objects with their lazy-loading facilities help to keep the
subset of objects that are already in memory connected to the rest
of the objects. This is an essential property as without it there
would always be fragile partial objects at the outer edges of your
object graph.

Doctrine CouchDB implements a variant of the proxy pattern where it
generates classes that extend your document classes and adds
lazy-loading capabilities to them. Doctrine can then give you an
instance of such a proxy class whenever you request an object of
the class being proxied. This happens in two situations:

Reference Proxies
~~~~~~~~~~~~~~~~~

The method ``DocumentManager#getReference($documentName, $identifier)``
lets you obtain a reference to a document for which the identifier
is known, without loading that document from the database. This is
useful, for example, as a performance enhancement, when you want to
establish an association to a document for which you have the
identifier. You could simply do this:

.. code-block:: php

    <?php
    // $dm instanceof DocumentManager, $cart instanceof MyProject\Model\Cart
    // $itemId comes from somewhere, probably a request parameter
    $item = $em->getReference('MyProject\Model\Item', $itemId);
    $cart->addItem($item);

Here, we added an Item to a Cart without loading the Item from the
database. If you invoke any method on the Item instance, it would
fully initialize its state transparently from the database. Here
$item is actually an instance of the proxy class that was generated
for the Item class but your code does not need to care. In fact it
**should not care**. Proxy objects should be transparent to your
code.

Association proxies
~~~~~~~~~~~~~~~~~~~

The second most important situation where Doctrine uses proxy
objects is when querying for objects. Whenever you query for an
object that has a single-valued association to another object that
is configured LAZY, without joining that association in the same
query, Doctrine puts proxy objects in place where normally the
associated object would be. Just like other proxies it will
transparently initialize itself on first access.
