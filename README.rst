========================
Django Elasticsearch DSL
========================

.. image:: https://travis-ci.org/sabricot/django-elasticsearch-dsl.png?branch=master
    :target: https://travis-ci.org/sabricot/django-elasticsearch-dsl
.. image:: https://codecov.io/gh/sabricot/django-elasticsearch-dsl/coverage.svg?branch=master
    :target: https://codecov.io/gh/sabricot/django-elasticsearch-dsl
.. image:: https://badge.fury.io/py/django-elasticsearch-dsl.svg
    :target: https://pypi.python.org/pypi/django-elasticsearch-dsl

This is a package that allows indexing of django models in elasticsearch. It is
built as a thin wrapper around elasticsearch-dsl-py_ so you can use all the features developed
by the elasticsearch-dsl-py team.

.. _elasticsearch-dsl-py: https://github.com/elastic/elasticsearch-dsl-py

Features
--------

- Based on elasticsearch-dsl-py_ so you can make queries with the Search_ class.
- Django signal receivers on save and delete for keeping Elasticsearch in sync.
- Management commands for creating, deleting, rebuilding and populating indices.
- Elasticsearch auto mapping from django models fields.
- Complex field type support (ObjectField, NestedField).
- Requirements
   - Django >= 1.8
   - Python 2.7, 3.4, 3.5, 3.6
   - Elasticsearch >= 2.0 < 7.0

.. _Search: http://elasticsearch-dsl.readthedocs.io/en/stable/search_dsl.html

Quickstart
----------

Install Django Elasticsearch DSL::

    pip install django-elasticsearch-dsl

    # Elasticsearch 6.x
    pip install 'elasticsearch-dsl>=6.0,<6.2'

    # Elasticsearch 5.x
    pip install 'elasticsearch-dsl>=5.0,<6.0'

    # Elasticsearch 2.x
    pip install 'elasticsearch-dsl>=2.1,<3.0'

Then add ``django_elasticsearch_dsl`` to the INSTALLED_APPS

You must define ``ELASTICSEARCH_DSL`` in your django settings.

For example:

.. code-block:: python

    ELASTICSEARCH_DSL={
        'default': {
            'hosts': 'localhost:9200'
        },
    }

``ELASTICSEARCH_DSL`` is then passed to ``elasticsearch-dsl-py.connections.configure`` (see here_).

.. _here: http://elasticsearch-dsl.readthedocs.io/en/stable/configuration.html#multiple-clusters

Then for a model:

.. code-block:: python

    # models.py

    class Car(models.Model):
        name = models.CharField()
        color = models.CharField()
        description = models.TextField()
        type = models.IntegerField(choices=[
            (1, "Sedan"),
            (2, "Truck"),
            (4, "SUV"),
        ])

To make this model work with Elasticsearch, create a subclass of ``django_elasticsearch_dsl.DocType``
and create a ``django_elasticsearch_dsl.Index`` to define your Elasticsearch indices, names, and settings. This classes must be
defined in a ``documents.py`` file.

.. code-block:: python

    # documents.py

    from django_elasticsearch_dsl import DocType, Index
    from .models import Car

    # Name of the Elasticsearch index
    car = Index('cars')
    # See Elasticsearch Indices API reference for available settings
    car.settings(
        number_of_shards=1,
        number_of_replicas=0
    )


    @car.doc_type
    class CarDocument(DocType):
        class Meta:
            model = Car # The model associated with this DocType

            # The fields of the model you want to be indexed in Elasticsearch
            fields = [
                'name',
                'color',
                'description',
                'type',
            ]

            # Ignore auto updating of Elasticsearch when a model is saved
            # or deleted:
            # ignore_signals = True
            # Don't perform an index refresh after every update (overrides global setting):
            # auto_refresh = False
            # Paginate the django queryset used to populate the index with the specified size
            # (by default there is no pagination)
            # queryset_pagination = 5000


To create and populate the Elasticsearch index and mapping use the search_index command::

    $ ./manage.py search_index --rebuild

Now, when you do something like:

.. code-block:: python

    car = Car(
        name="Car one",
        color="red",
        type=1,
        description="A beautiful car"
    )
    car.save()

The object will be saved in Elasticsearch too (using a signal handler). To get an
elasticsearch-dsl-py Search_ instance, use:

.. code-block:: python

    s = CarDocument.search().filter("term", color="red")

    # or

    s = CarDocument.search().query("match", description="beautiful")

    for hit in s:
        print(
            "Car name : {}, description {}".format(hit.name, hit.description)
        )

The previous example returns a result specific to elasticsearch_dsl_, but it is also
possible to convert the elastisearch result into a real django queryset, just be aware
that this costs a sql request to retrieve the model instances with the ids returned by
the elastisearch query.

.. _elasticsearch_dsl: http://elasticsearch-dsl.readthedocs.io/en/latest/search_dsl.html#response

.. code-block:: python

    s = CarDocument.search().filter("term", color="blue")[:30]
    qs = s.to_queryset()
    # qs is just a django queryset and it is called with order_by to keep
    # the same order as the elasticsearch result.
    for car in qs:
        print(car.name)

Fields
------

Once again the ``django_elasticsearch_dsl.fields`` are subclasses of elasticsearch-dsl-py
fields_. They just add support for retrieving data from django models.


.. _fields: http://elasticsearch-dsl.readthedocs.io/en/stable/persistence.html#mappings

Using Different Attributes for Model Fields
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Let's say you don't want to store the type of the car as an integer, but as the
corresponding string instead. You need some way to convert the type field on
the model to a string, so we'll just add a method for it:

.. code-block:: python

    # models.py

    class Car(models.Model):
        # ... #
        def type_to_string(self):
            """Convert the type field to its string representation
            (the boneheaded way).
            """
            if self.type == 1:
                return "Sedan"
            elif self.type == 2:
                return "Truck"
            else:
                return "SUV"

Now we need to tell our ``DocType`` subclass to use that method instead of just
accessing the ``type`` field on the model directly. Change the CarDocument to look
like this:

.. code-block:: python

    # documents.py

    from django_elasticsearch_dsl import DocType, fields

    # ... #

    @car.doc_type
    class CarDocument(DocType):
        # add a string field to the Elasticsearch mapping called type, the
        # value of which is derived from the model's type_to_string attribute
        type = fields.TextField(attr="type_to_string")

        class Meta:
            model = Car
            # we removed the type field from here
            fields = [
                'name',
                'color',
                'description',
            ]

After a change like this we need to rebuild the index with::

    $ ./manage.py search_index --rebuild

Using prepare_field
~~~~~~~~~~~~~~~~~~~

Sometimes, you need to do some extra prepping before a field should be saved to
Elasticsearch. You can add a ``prepare_foo(self, instance)`` method to a DocType
(where foo is the name of the field), and that will be called when the field
needs to be saved.

.. code-block:: python

    # documents.py

    # ... #

    class CarDocument(DocType):
        # ... #

        foo = TextField()

        def prepare_foo(self, instance):
            return " ".join(instance.foos)

Handle relationship with NestedField/ObjectField
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

For example for a model with ForeignKey relationships.

.. code-block:: python

    # models.py

    class Car(models.Model):
        name = models.CharField()
        color = models.CharField()
        manufacturer = models.ForeignKey('Manufacturer')

    class Manufacturer(models.Model):
        name = models.CharField()
        country_code = models.CharField(max_length=2)
        created = models.DateField()

    class Ad(models.Model):
        title = models.CharField()
        description = models.TextField()
        created = models.DateField(auto_now_add=True)
        modified = models.DateField(auto_now=True)
        url = models.URLField()
        car = models.ForeignKey('Car', related_name='ads')


You can use an ObjectField or a NestedField.

.. code-block:: python

    # documents.py

    from django_elasticsearch_dsl import DocType, Index, fields
    from .models import Car, Manufacturer, Ad

    car = Index('cars')
    car.settings(
        number_of_shards=1,
        number_of_replicas=0
    )


    @car.doc_type
    class CarDocument(DocType):
        manufacturer = fields.ObjectField(properties={
            'name': fields.TextField(),
            'country_code': fields.TextField(),
        })
        ads = fields.NestedField(properties={
            'description': fields.TextField(analyzer=html_strip),
            'title': fields.TextField(),
            'pk': fields.IntegerField(),
        })

        class Meta:
            model = Car
            fields = [
                'name',
                'color',
            ]
            related_models = [Manufacturer, Ad]  # Optional: to ensure the Car will be re-saved when Manufacturer or Ad is updated

        def get_queryset(self):
            """Not mandatory but to improve performance we can select related in one sql request"""
            return super(CarDocument, self).get_queryset().select_related(
                'manufacturer'
            )

        def get_instances_from_related(self, related_instance):
            """If related_models is set, define how to retrieve the Car instance(s) from the related model.
            The related_models option should be used with caution because it can lead in the index
            to the updating of a lot of items.
            """
            if isinstance(related_instance, Manufacturer):
                return related_instance.car_set.all()
            elif isinstance(related_instance, Ad):
                return related_instance.car


Field Classes
~~~~~~~~~~~~~
Most Elasticsearch field types_ are supported. The ``attr`` argument is a dotted
"attribute path" which will be looked up on the model using Django template
semantics (dict lookup, attribute lookup, list index lookup). By default the attr
argument is set to the field name.

For the rest, the field properties are the same as elasticsearch-dsl
fields_.

So for example you can use a custom analyzer_:

.. _analyzer: http://elasticsearch-dsl.readthedocs.io/en/stable/persistence.html#analysis
.. _types: https://www.elastic.co/guide/en/elasticsearch/reference/5.4/mapping-types.html

.. code-block:: python

    # documents.py

    # ... #

    html_strip = analyzer(
        'html_strip',
        tokenizer="standard",
        filter=["standard", "lowercase", "stop", "snowball"],
        char_filter=["html_strip"]
    )

    @car.doc_type
    class CarDocument(DocType):
        description = fields.TextField(
            analyzer=html_strip,
            fields={'raw': fields.KeywordField()}
        )

        class Meta:
            model = Car
            fields = [
                'name',
                'color',
            ]


Available Fields
~~~~~~~~~~~~~~~~

- Simple Fields

    - BooleanField(attr=None, \*\*elasticsearch_properties)
    - ByteField(attr=None, \*\*elasticsearch_properties)
    - CompletionField(attr=None, \*\*elasticsearch_properties)
    - DateField(attr=None, \*\*elasticsearch_properties)
    - DoubleField(attr=None, \*\*elasticsearch_properties)
    - FileField(attr=None, \*\*elasticsearch_properties)
    - FloatField(attr=None, \*\*elasticsearch_properties)
    - IntegerField(attr=None, \*\*elasticsearch_properties)
    - IpField(attr=None, \*\*elasticsearch_properties)
    - GeoPointField(attr=None, \*\*elasticsearch_properties)
    - GeoShapeField(attr=None, \*\*elasticsearch_properties)
    - ShortField(attr=None, \*\*elasticsearch_properties)
    - StringField(attr=None, \*\*elasticsearch_properties)

- Complex Fields

    - ObjectField(properties, attr=None, \*\*elasticsearch_properties)
    - NestedField(properties, attr=None, \*\*elasticsearch_properties)

- Elasticsearch >=5 Fields

    - TextField(attr=None, \*\*elasticsearch_properties)
    - KeywordField(attr=None, \*\*elasticsearch_properties)

``properties`` is a dict where the key is a field name, and the value is a field
instance.


Index
-----

To define an Elasticsearch index you must instantiate a ``django_elasticsearch_dsl.Index`` class and set the name
and settings of the index. This class inherits from elasticsearch-dsl-py Index_.
After you instantiate your class, you need to associate it with the DocType you
want to put in this Elasticsearch index.


.. _Index: http://elasticsearch-dsl.readthedocs.io/en/stable/persistence.html#index

.. code-block:: python

    # documents.py

    from django_elasticsearch_dsl import DocType, Index
    from .models import Car, Manufacturer

    # The name of your index
    car = Index('cars')
    # See Elasticsearch Indices API reference for available settings
    car.settings(
        number_of_shards=1,
        number_of_replicas=0
    )


    @car.doc_type
    class CarDocument(DocType):
        class Meta:
            model = Car
            fields = [
                'name',
                'color',
            ]

    @car.doc_type
    class ManufacturerDocument(DocType):
        class Meta:
            model = Car
            fields = [
                'name', # If a field as the same name in multiple DocType of
                        # the same Index, the field type must be identical
                        # (here fields.TextField)
                'country_code',
            ]

When you execute the command::

    $ ./manage.py search_index --rebuild

This will create an index named ``cars`` in Elasticsearch with two mappings:
``manufacturer_document`` and ``car_document``.


Management Commands
-------------------

Delete all indices in Elasticsearch or only the indices associate with a model (--models):

::

    $ search_index --delete [-f] [--models [app[.model] app[.model] ...]]


Create the indices and their mapping in Elasticsearch:

::

    $ search_index --create [--models [app[.model] app[.model] ...]]

Populate the Elasticsearch mappings with the django models data (index need to be existing):

::

    $ search_index --populate [--models [app[.model] app[.model] ...]]

Recreate and repopulate the indices:

::

    $ search_index --rebuild [-f] [--models [app[.model] app[.model] ...]]


Settings
--------

ELASTICSEARCH_DSL_AUTOSYNC
~~~~~~~~~~~~~~~~~~~~~~~~~~

Default: ``True``

Set to ``False`` to globally disable auto-syncing.

ELASTICSEARCH_DSL_INDEX_SETTINGS
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Default: ``{}``

Additional options passed to the elasticsearch-dsl Index settings (like ``number_of_replicas`` or ``number_of_shards``).

ELASTICSEARCH_DSL_AUTO_REFRESH
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Default: ``True``

Set to ``False`` not force an [index refresh](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-refresh.html) with every save.

ELASTICSEARCH_DSL_SIGNAL_PROCESSOR
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This (optional) setting controls what SignalProcessor class is used to handle
Django's signals and keep the search index up-to-date.

An example:

.. code-block:: python

    ELASTICSEARCH_DSL_SIGNAL_PROCESSOR = 'django_elasticsearch_dsl.signals.RealTimeSignalProcessor'

Defaults to ``django_elasticsearch_dsl.signals.RealTimeSignalProcessor``.

You could, for instance, make a ``CelerySignalProcessor`` which would add
update jobs to the queue to for delayed processing.

Testing
-------

You can run the tests by creating a Python virtual environment, installing
the requirements from ``requirements_test.txt`` (``pip install -r requirements_test``)::

    $ python runtests.py

Or::

    $ make test

    $ make test-all # for tox testing

For integration testing with a running Elasticsearch server::

    $ python runtests.py --elasticsearch [localhost:9200]


TODO
----

- Add support for --using (use another Elasticsearch cluster) in management commands.
- Add management commands for mapping level operations (like update_mapping....).
- Dedicated documentation.
- Generate ObjectField/NestField properties from a DocType class.
- More examples.
- Better ``ESTestCase`` and documentation for testing
