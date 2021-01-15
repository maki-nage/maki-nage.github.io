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

The simplest way to create a new operator is by composing other existing
operators. Let's consider these three operations done on some text input:

.. code:: python

    rs.ops.map(lambda i: i.replace("-", " "))
    rs.ops.filter(lambda i: 'bill' not in i)
    rs.ops.map(lambda i: i.capitalize())

The natural way to use them is by chaining them in a pipe:

.. code:: python

    data.pipe(
        rs.ops.map(lambda i: i.replace("-", " "))
        rs.ops.filter(lambda i: 'bill' not in i)
        rs.ops.map(lambda i: i.capitalize())
    )


But as more and more transforms are added, you can end up with a very long pipe.
You can easily improve the readability and reuse some operations by grouping
operators. For example, the previous three operators can be grouped as a custom
operator:

.. code:: python

    def cleanup_text():
        return rx.pipe(
            rs.ops.map(lambda i: i.replace("-", " ")),
            rs.ops.filter(lambda i: 'bill' not in i),
            rs.ops.map(lambda i: i.capitalize()),        
        )

The function *cleanup_text* is an operator that you can use in a pipe:

.. code:: python

    data = [
        'hello',
        'the-quick-brown-fox',
        'bill is fast',
        'lorem ipsum'
    ]

    rx.from_(data).pipe(
        cleanup_text()
    ).subscribe(on_next=print)


.. code:: console

    Hello
    The quick brown fox
    Lorem ipsum


Create a stateful operator by composition
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Stateful operators are more complex to implement because they need to update a
state. Hopefully, In many cases, you can create new operators by combining three
base operators: scan, filter, and map:

* The scan operator updates the state.
* The filter operator controls when items must be emitted.
* The map operator emits the items from the state.

Let's consider the following need: Sum all items up to some threshold. An item
must be emitted each time the sum would cross the threshold. Then the sum process
starts again:

.. marble::
    :alt: sum_split

    -1-2-3-4-1-2-6-1-|
    [  sum_split(5)  ]
    -----3-3---5-2-6-|


The state logic can be implemented with the following function:

.. code:: python

    def _sum_split(acc, i):
            if acc[0] + i > threshold:
                return (i, acc[0])
            return (acc[0]+i, None)

Here *acc* contains the state. It is a tuple where:

- The first field is the current sum
- The second field is the item to emit or None if nothing must be emitted.

The full implementation of the operator simply consists in combining scan,
filter, and map in a wrapper function:

.. code:: python

    def sum_split(threshold):
        def _sum_split(acc, i):
            if acc[0] + i > threshold:
                return (i, acc[0])
            return (acc[0]+i, None)
            
        return rx.pipe(
            rs.ops.scan(_sum_split, seed=(0, None)),
            rs.ops.filter(lambda i: i[1] is not None),
            rs.ops.map(lambda i: i[1]),
        )


You can now use *sum_split* just as any builtin operator:

.. code:: python

    data = [1, 2, 3, 4, 1, 2, 6, 1]

    rx.from_(data).pipe(
        sum_and_split(5)
    ).subscribe(print)

.. code:: console

    3 3 5 2 6


Run a local Kafka server
-------------------------

