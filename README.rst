=============================
Django Elastic App Search
=============================

.. image:: https://badge.fury.io/py/django-elastic-appsearch.svg
    :target: https://badge.fury.io/py/django-elastic-appsearch

.. image:: https://github.com/corrosivekid/django_elastic_appsearch/workflows/Tests/badge.svg
    :target: https://github.com/CorrosiveKid/django_elastic_appsearch/actions?query=workflow%3ATests

.. image:: https://github.com/corrosivekid/django_elastic_appsearch/workflows/Lint/badge.svg
    :target: https://github.com/CorrosiveKid/django_elastic_appsearch/actions?query=workflow%3ALint

.. image:: https://codecov.io/gh/CorrosiveKid/django_elastic_appsearch/branch/master/graph/badge.svg
    :target: https://codecov.io/gh/CorrosiveKid/django_elastic_appsearch

.. image:: https://readthedocs.org/projects/django-elastic-appsearch/badge/?version=latest
    :target: https://django-elastic-appsearch.readthedocs.io/en/latest/?badge=latest

.. image:: https://github.com/corrosivekid/django_elastic_appsearch/workflows/Dependencies/badge.svg
    :target: https://github.com/CorrosiveKid/django_elastic_appsearch/actions?query=workflow%3ADependencies

.. image:: https://badgen.net/dependabot/CorrosiveKid/django_elastic_appsearch?icon=dependabot
    :target: https://dependabot.com/

Integrate your Django Project with Elastic App Search with ease.

Documentation
-------------

The full documentation is at https://django_elastic_appsearch.readthedocs.io. Read our step-by-step guide on integrating App Search with your existing Django project over at Medium_.

.. _Medium: https://medium.com/@rasika.am/integrating-a-django-project-with-elastic-app-search-fb9f16726b5c

Dependencies
------------

* Python >= 3.6
* Django >= 2.2
* `elastic-app-search <https://pypi.org/project/elastic-app-search/>`_
* `serpy <https://pypi.org/project/serpy/>`_

Usage
-----
Installing
==========

Install Django Elastic App Search::

    pip install django_elastic_appsearch

Add it to your `INSTALLED_APPS`:

.. code-block:: python

    INSTALLED_APPS = (
        ...
        'django_elastic_appsearch',
        ...
    )

Add the Elastic App Search URL and Key to your settings module:

.. code-block:: python

    APPSEARCH_HOST = 'localhost:3002'
    APPSEARCH_API_KEY = 'some_appsearch_api_token'

Configuring app search indexable models
=======================================

Configure the Django models you want to index to Elastic App Search. You can do this by inheriting from the ``AppSearchModel``, and then setting some meta options.

``AppsearchMeta.appsearch_engine_name`` - Defines which engine in your app search instance your model will be indexed to.

``AppsearchMeta.appsearch_serialiser_class`` - Defines how your model object will be serialised when sent to your elastic app search instance. The serialiser and fields used here derives from `Serpy <https://serpy.readthedocs.io/>`__, and you can use any of the serpy features like method fields.

Example:

.. code-block:: python

    from django_elastic_appsearch.orm import AppSearchModel
    from django_elastic_appsearch import serialisers

    class CarSerialiser(serialisers.AppSearchSerialiser):
        full_name = serialisers.MethodField()
        make = serialisers.StrField()
        model = serialisers.StrField()
        manufactured_year = serialisers.Field()

        def get_full_name(self, instance):
            return '{} {}'.format(make, model)


    class Car(AppSearchModel):

        class AppsearchMeta:
            appsearch_engine_name = 'cars'
            appsearch_serialiser_class = CarSerialiser

        make = models.CharField(max_length=100)
        model = models.CharField(max_length=100)
        manufactured_year = models.CharField(max_length=4)

Using model and queryset methods to index and delete documents
==============================================================

Then you can call ``index_to_appsearch`` and ``delete_from_appsearch`` from your model objects.

Send the car with id 25 to app search.

.. code-block:: python

    from mymodels import Car

    car = Car.objects.get(id=25)
    car.index_to_appsearch()

Delete the car with id 21 from app search.

.. code-block:: python

    from mymodels import Car

    car = Car.objects.get(id=21)
    car.delete_from_appsearch()

You can also call ``index_to_appsearch`` and ``delete_from_appsearch`` on QuerySets of ``AppSearchModel``

Send all cars where the make is 'Toyota' to app search.

.. code-block:: python

    cars = Car.objects.filter(make='Toyota')
    cars.index_to_appsearch()

Delete all cars where the make is 'Saab' from app search

.. code-block:: python

    cars = Car.objects.filter(make='Saab')
    cars.delete_from_appsearch()

``index_to_appsearch`` methods on the QuerySet and your model also supports an optional ``update_only`` parameter which takes in a boolean value. If ``update_only`` is set to ``True``, the operation on the app search instance will be carried out as a ``PATCH`` operation. This will be useful if your Django application is only doing partial updates to the documents.

This will also mean that your serialisers can contain a subset of the fields for a document. This will be useful when two Django models/applications are using the same app search engine to update different sets of fields on a single document type.

Example below (Continued from the above ``Car`` example):

.. code-block:: python

    from django.db import models
    from django_elastic_appsearch.orm import AppSearchModel
    from django_elastic_appsearch import serialisers

    class CarVINNumberSerialiser(serialisers.AppSearchSerialiser):
        vin_number = serialisers.StrField()

    class CarVINNumber(AppSearchModel):

        class AppsearchMeta:
            appsearch_engine_name = 'cars'
            appsearch_serialiser_class = CarVINNumberSerialiser

        car = models.OneToOneField()
        vin_number = models.CharField(max_length=100)

.. code-block:: python

    from mymodels import CarVINNumber

    car_vin = CarVINNumber.objects.filter('car__id'=25).first()
    car_vin.vin_number = '1M8GDM9A_KP042788'
    car_vin.save()
    car_vin.refresh_from_db()
    car_vin.index_to_appsearch(update_only=True)

The above example will update the car document with id 25 with the new VIN number and leave the data for the rest of the fields intact.

Important note: ``PATCH`` operations on Elastic App Search cannot create new schema fields if you submit schema fields currently unknown to your engine. So always make sure you're submitting values for existing schema fields on your engine.

Use with your own custom queryset managers
==========================================

If you want to specify custom managers which also has this functionality, you can inherit from ``django_elastic_appsearch.orm.AppSearchQuerySet``

.. code-block:: python

    from django_elastic_appsearch.orm import AppSearchModel, AppSearchQuerySet

    class MyCustomQuerySetManager(AppSearchQuerySet):
        def my_custom_queryset_feature(self):
            # Do Something cool
            pass

    class MyCustomModel(AppSearchModel):
        field_1 = models.CharField(max_length=100)

        # Set the custom manager
        objects = MyCustomQuerySetManager.as_manager()

Settings
========

This package provides various Django settings entries you can use to configure your connection to the Elastic App Search instance you're using.

APPSEARCH_HOST
^^^^^^^^^^^^^^

* Required: Yes
* Default: No default value

This is a **required** setting to tell your Django application which Elastic App Search instance to connect with.

.. code-block:: python

    APPSEARCH_HOST = 'localhost:3002'

APPSEARCH_API_KEY
^^^^^^^^^^^^^^^^^

* Required: Yes
* Default: No default value

This is a **required** setting to tell your Django application the private key to use to talk to your Elastic App Search instance.

.. code-block:: python

    APPSEARCH_API_KEY = 'private-key'

APPSEARCH_USE_HTTPS
^^^^^^^^^^^^^^^^^^^

* Required: No
* Default: ``True``

This is an **optional** setting to configure whether to use HTTPS or not when your Django application communicates with your Elastic App Search instances. It defaults to ``True`` if it's not set. This might be useful when you're running your Django project against a local Elastic App Search instance. It's insecure to have this as ``False`` in a production environment, so make sure to change to ``True`` in your production version.

.. code-block:: python

    APPSEARCH_USE_HTTPS = False

APPSEARCH_CHUNK_SIZE
^^^^^^^^^^^^^^^^^^^^

* Required: No
* Default: ``100``

This is an **optional** setting to configure the chunk size when doing queryset indexing/deleting. Elastic App Search supports upto a 100 documents in one index/destroy request. With this setting, you can change it to your liking. It defaults to the maximum of ``100`` when this is not set. This might be useful when you want to reduce the size of a request to your Elastic App Search instance when your documents have a lot of fields/data.

.. code-block:: python

    APPSEARCH_CHUNK_SIZE = 50

APPSEARCH_INDEXING_ENABLED
^^^^^^^^^^^^^^^^^^^^^^^^^^

* Required: No
* Default: ``True``

This is an **optional** setting to configure if you want to disable indexing to your Elastic App Search instance. This is useful when you want to disable indexing without changing any code. When it's set to ``False``, any code where you use ``index_to_appsearch()`` or ``delete_from_appsearch()`` will not do anything. It's set to ``True`` by default when it's not set.

.. code-block:: python

    APPSEARCH_INDEXING_ENABLED = True

Example with all settings entries
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: python

    APPSEARCH_HOST = 'localhost:3002'
    APPSEARCH_API_KEY = 'private-key'
    APPSEARCH_USE_HTTPS = False
    APPSEARCH_CHUNK_SIZE = 50
    APPSEARCH_INDEXING_ENABLED = True

Writing Tests
=============

This package provides a test case mixin called ``MockedAppSearchTestCase`` which makes it easier for you to write test cases against ``AppSearchModel``'s without actually having to run an Elastic App Search instance during tests.

All you have to do is inherit the mixin, and all the calls to Elastic App Search will be mocked. Example below.

.. code-block:: python

    from django.test import TestCase
    from django_elastic_appsearch.test import MockedAppSearchTestCase
    from myapp.test.factories import CarFactory

    class BookTestCase(MockedAppSearchTestCase, TestCase):
        def test_indexing_book(self):
            car = CarFactory()
            car.save()
            car.index_to_appsearch()

            self.assertAppSearchModelIndexCallCount(1)

You will have access to the following methods to check call counts to different mocked app search methods.

``self.assertAppSearchQuerySetIndexCallCount`` — Check the number of times index_to_appsearch was called on a appsearch model querysets.

``self.assertAppSearchQuerySetDeleteCallCount`` — Check the number of times delete_from_appsearch was called on an appsearch model querysets.

``self.assertAppSearchModelIndexCallCount`` — Check the number of times index_to_appsearch was called on an appsearch model objects.

``self.assertAppSearchModelDeleteCallCount`` — Check the number of times delete_from_appsearch was called on an appsearch model objects.

Using the elastic app search python client
==========================================

We use the official `elastic app search python client <https://github.com/elastic/app-search-python>`_ under the hood to communicate with the app search instance. So if needed, you can access the app search instance directly and use the functionality of the official elastic app search `client <https://github.com/elastic/app-search-python#usage>`_. Example below.

.. code-block:: python

    from django_elastic_appsearch.clients import get_api_v1_client

    client = get_api_v1_client()
    client.search('cars', 'Toyota Corolla', {})

Contributing
------------

Contributors are welcome!

* Prior to opening a pull request, please create an issue to discuss the change/feature you've written/thinking of writing if it doesn't already exist.

* Please write simple code and concise documentation, when appropriate.

* Please write test cases to cover the code you've written, where possible.

* Read the `Contributing <https://django-elastic-appsearch.readthedocs.io/en/latest/contributing.html#>`_ section of our documentation for more information around contributing to this project.

Running Tests
-------------

Does the code actually work?

::

    $ pipenv install --dev
    $ pipenv shell
    (django_elastic_appsearch) $ tox

Credits
-------

Tools used in rendering this package:

*  Cookiecutter_
*  `cookiecutter-djangopackage`_

.. _Cookiecutter: https://github.com/audreyr/cookiecutter
.. _`cookiecutter-djangopackage`: https://github.com/pydanny/cookiecutter-djangopackage
