How To
======

This section contains examples of common operations in RxSci and Maki Nage. You
can find the associated code for most of them `here
<https://github.com/maki-nage/makinage-examples/tree/master/notebook>`_.


Import and Export Data
------------------------

Read a csv file
~~~~~~~~~~~~~~~~

You can work directly on CSV files (vs using Kafka as a source and sink), via
the *load_from_csv* factory operator.

First import the *csv* module that contains all csv operators:

.. code:: python

    import rxsci.container.csv as csv


Then you must declare the schema of the CSV file. Below is the schema of the
iris dataset that you can retrieve from `kaggle
<https://www.kaggle.com/uciml/iris>`_.

.. code:: python

    iris_parser = csv.create_line_parser(
        dtype=[
            ('id', 'int'),
            ('sepal_length_cm', 'float'),
            ('sepal_width_cm', 'float'),
            ('petal_length_cm', 'float'),
            ('petal_width_cm', 'float'),
            ('species', 'str'),
        ]
    )

The *create_line_parser* operator supports options to customize the parser, such
as the text encoding, column separator, and default values. See the
`documentation
<https://www.makinage.org/doc/rxsci/latest/operators_container.html#rxsci.container.csv.create_line_parser>`_
for more information.

.. code:: python

    iris_data = csv.load_from_file('./Iris.csv', iris_parser)



Write a csv file
~~~~~~~~~~~~~~~~~~

The *dump_to_file* operator writes each input item to a row of a CSV file. The
input items must be namedtuples. So first ensure that your data is structured as
a namedtuple:

.. code:: python

    IrisFeature = namedtuple('IrisFeature', ['id', 'species', 'sepal_ratio', 'petal_ratio'])

    iris_features = iris_data.pipe(
        rs.ops.map(lambda i: IrisFeature(
            id=i.id, species=i.species,
            sepal_ratio=i.sepal_length_cm / i.sepal_width_cm,
            petal_ratio=i.petal_length_cm / i.petal_width_cm
        )),
    )

The dump_to_file operator uses the fields of the namedtuple to infer the columns
names of the CSV file:

.. code:: python

    iris_features.pipe(
        csv.dump_to_file('iris_features.csv', encoding='utf-8'),
    ).subscribe()


Extend Maki Nage
-----------------

Create an operator by composition
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~



Run a local Kafka server
-------------------------

