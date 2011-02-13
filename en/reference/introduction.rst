Introduction
============

Doctrine CouchDB comes with two components that enable you to communicate to a CouchDB database using PHP:

-   **CouchDB Client** - A thin layer that comes with HTTP clients optimized for speaking to CouchDB and a convenience wrapper around the CouchDB API.
-   **Object-Document-Mapper (ODM)** - A mapping layer that maps CouchDB documents to PHP objects following the Doctrine persistence semantics also employed by the Doctrine 2.0 ORM and MongoDB ODM.

You can use the CouchDB Client without the Document-Mapper if you just want to talk to a CouchDB database without the object mapping. The Mapper however depends on the CouchDB Client.

Features
--------

-   Convenience layer to access CouchDB HTTP Document, Bulk and View API
-   Mapping layer for CouchDB Documents to PHP Objects
-   Write-behind changes from PHP Memory to CouchDB for high-performance
-   Map CouchDB views to PHP objects
-   Integration into CouchDB Lucene to query Lucene and retrieve your mapped PHP objects.
-   Explicit support for document conflict resolution

Architecture
------------

The Doctrine persistence semantics demand a strict separation of persistence and business logic. This means the PHP Objects that you can map to CouchDB documents do not extend any base class or implement any interface. These objects and their associations between each other don't even know that they are saved inside the CouchDB, fow all they know it could also be the ORM or MongoDB solving their persistence needs. Instead of having ``save()``, ``delete()`` and finder methods on your PHP objects Doctrine CouchDB ODM provides you with a management/query layer that translates between CouchDB documents and PHP objects.

For example take this BlogPost entity:

.. code-block::

    <?php
    namespace MyApp\Document;

    class BlogPost
    {
        private $id;
        private $headline;
        private $text;
        private $publishDate;
        
        // getter/setter here
    }

No abstract/base-class nor interface was implemented, yet you can save an object of ``BlogPost`` into the CouchDB. You have to provide some metadata mapping information though, which is a neccessary evil of any generic solution mapping between two different systems (CouchDB and PHP Objects). Doctrine CouchDB allows you to specifiy metadata using four different means: PHP Docblock Annotations, YAML, XML and PHP. Have a look at the following code examples:

.. configuration-block::

    .. code-block:: php

        <?php
        namespace MyApp\Document;
        
        /** @Document */
        class BlogPost
        {
            /** @Id */
            private $id;
            /** @Field(type="string") */
            private $headline;
            /** @Field(type="string") */
            private $text;
            /** @Field(type="datetime) */
            private $publishDate;
            
            // getter/setter here
        }

This simple definitions describe to Doctrine CouchDB ODM what parts of your object should be mapped. Now your application code saving an instance of a blog post will look like the following lines:

.. code-block::

    <?php
    // $dm is an instanceof Doctrine\ODM\CouchDB\DocumentManager

    $blogPost = new BlogPost();
    $blogPost->setHeadline("Hello World!");
    $blogPost->setText("This is a blog post going to be saved into CouchDB");
    $blogPost->setPublishDate(new DateTime("now"));

    $dm->persist($blogPost);
    $dm->flush();

Using an instance of a ``DocumentManager`` you can save the blog post into your CouchDB instance. Calling ``persist`` tells the DocumentManager that this blog post is new and should be managed by Doctrine CouchDB ODM. It does not actually perform a POST request to create a document yet. The only thing that is being done is the generation of a UUID, which is automatically injected into the ``$id`` variable of your BlogPost.

The call to ``flush()`` triggers the synchronization of the in-memory state of all current objects with the CouchDB database. In this particular case only one BlogPost is managed by the DocumentManager and does not yet have a corresponding document in the CouchDB, hence a POST request to create the document is executed. 

Write-Behind
~~~~~~~~~~~~

You may ask yourself why ``persist`` and ``flush`` are two separate functions in the previous example. Doctrine persistence semantics apply a drastic performance optimization by aggregating all the required changes and synchronizing them back to the database at once. In the case of CouchDB ODM this means that all changes on objects (managed by CouchDB) in memory of the current PHP request are synchronized to CouchDB in a single POST request using the HTTP Bulk Document API. Compared to making an update request per document this leads to a considerable difference in performance.

This approach has a drawback though with regards to the transactional semantics of CouchDB. By default the bulk update is forced using the ... parameter of the HTTP BUlk Document API, which means that in case of different versioning numbers it will produce document conflicts that you have to resolve later. Doctrine CouchDB ODM offers an event to resolve any document conflict and it is planned to offer automatic resolution strategies such as "First-One Wins" or "Last-One Wins". If you don't enable forcing changes to the CouchDB you can end up with inconsistent state, for example if one update of a document is accepted and another one is rejected.

We haven't actually figured out the best way of handling "object transactions" ourselfes, but are experimenting with it to find the best possible solution before releasing a stable Doctrine CouchDB version. Feedback in this area is highly appreciated.

Querying
~~~~~~~~

Coming back to our blog post example, in any next request you can grab the BlogPost by calling a simple finder method:

.. code-block::

    <?php
    // $dm is an instanceof Doctrine\ODM\CouchDB\DocumentManager
    $blogPost = $dm->find("MyApp\Document\BlogPost", $theUUID);

Here the variable ``$blogPost`` will be of the type ``MyApp\Document\BlogPost`` with no magic whatsoever being attached to that object. There will be some magic required later on described in the Object-Graph Traversal section, but its the most unspectacular magic we could come up with.

If you know how CouchDB works you are probably now asking yourself how the HTTP View API of CouchDB is integrated into Doctrine CouchDB ODM to offer convenience finder methods such as MongoDB or a relational database would easily allow. The answer may shock you: There is no magic query API provided except the previously shown query by ID (UUID or any assigned document ID) by default. You have to actively mark fields as "indexed" to be able to access them using equality constraints.

The reason for this approach is easily explained. While Doctrine CouchDB could potentially generate a bunch of powerful views for you that allow querying all fields by different means could potentially lead to a performance problem for your application down the road. Views in CouchDB come at a price: The more there are the slower your CouchDB gets. A view that indexes all fields of all documents would be much too large, so you have to construct the views yourself or use the indexing attributes to generate a simple query.

Object-Graph Traversal
~~~~~~~~~~~~~~~~~~~~~~

Besides the actual saving of objects the Doctrine persistence semantics dictate that you can always traverse from any part of the object graph to any other part as long as there are associations connecting the objects with each other. In a simple implementation this would mean you have to load all the objects into memory for every request. However Doctrine CouchDB is very efficient using the lazy-loading pattern.

Every single or multi-valued association such as Many-To-One or Many-To-Many are replaced with lazy loading proxies when created. This means the object graph is fully traversable, but only the parts you actually accessed get loaded into memory. For this feature to work there is some code-generation necessary, Doctrine creates proxy classes that extend your documents if necessary.

Configuration
=============

Installation
------------

Currently you have to install CouchDB ODM from Github at http://github.com/doctrine/couchdb-odm

Bootstrapping
-------------

To bootstrap your application with Doctrine CouchDB ODM you have to initialize the DocumentManager. The following code block shows a sample setup, rather complex at the moment but it will get simpler in the future:

.. code-block:: php

    <?php

    $httpClient = new \Doctrine\ODM\CouchDB\HTTP\SocketClient();
    $database = "myapp";
    $reader = new \Doctrine\Common\Annotations\AnnotationReader();
    $reader->setDefaultAnnotationNamespace('Doctrine\ODM\CouchDB\Mapping\\');
    $paths = __DIR__ . "/Documents";
    $metadriver = new \Doctrine\ODM\CouchDB\Mapping\Driver\AnnotationDriver($reader, $paths);

    $config = new \Doctrine\ODM\CouchDB\Configuration();
    $config->setDatabase($database);
    $config->setProxyDir(__DIR__ . "/cache");
    $config->setMetadataDriverImpl($metadriver);
    $config->setHttpClient($httpClient);

    $documentManager = \Doctrine\ODM\CouchDB\DocumentManager::create($config);

Explanation will follow.



