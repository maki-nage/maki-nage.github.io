Home Energy Consumption
========================

In this first tutorial, we see how to use maki-nage for basic feature
engineering, and how to deploy a model operating in streaming mode. The task of
this tutorial is a regression: We will predict the power consumption of a house
, based on weather information.

.. important::
    Maki Nage is designed to work efficiently with `pypy
    <https://www.pypy.org/>`_. Please ensure that you use is as your interpreter
    for better performances.


The dataset
------------

On this example we will work on the *Smart Home Dataset*. It can be downloaded from
kaggle `here <https://www.kaggle.com/taranvee/smart-home-dataset-with-weather-information>`_.

The dataset is a csv file of 130MB, its content looks like this:

.. code:: console

    $ head HomeC.csv
    time,use [kW],gen [kW],House overall [kW],Dishwasher [kW],Furnace 1 [kW],Furnace 2 [kW],Home office [kW],Fridge [kW],Wine cellar [kW],Garage door [kW],Kitchen 12 [kW],Kitchen 14 [kW],Kitchen 38 [kW],Barn [kW],Well [kW],Microwave [kW],Living room [kW],Solar [kW],temperature,icon,humidity,visibility,summary,apparentTemperature,pressure,windSpeed,cloudCover,windBearing,precipIntensity,dewPoint,precipProbability
    1451624400,0.932833333,0.003483333,0.932833333,3.33E-05,0.0207,0.061916667,0.442633333,0.12415,0.006983333,0.013083333,0.000416667,0.00015,0,0.03135,0.001016667,0.004066667,0.001516667,0.003483333,36.14,clear-night,0.62,10,Clear,29.26,1016.91,9.18,cloudCover,282,0,24.4,0
    1451624401,0.934333333,0.003466667,0.934333333,0,0.020716667,0.063816667,0.444066667,0.124,0.006983333,0.013116667,0.000416667,0.00015,0,0.0315,0.001016667,0.004066667,0.00165,0.003466667,36.14,clear-night,0.62,10,Clear,29.26,1016.91,9.18,cloudCover,282,0,24.4,0
    1451624402,0.931816667,0.003466667,0.931816667,1.67E-05,0.0207,0.062316667,0.446066667,0.123533333,0.006983333,0.013083333,0.000433333,0.000166667,1.67E-05,0.031516667,0.001,0.004066667,0.00165,0.003466667,36.14,clear-night,0.62,10,Clear,29.26,1016.91,9.18,cloudCover,282,0,24.4,0

For simplicity, We will only few columns:

* House overall: This is the label we want to predict, the total energy consumption of the house.
* temperature: The weather temperature in °F.
* pressure: The atmospheric pressure in hPA.
* windSpeed: The wind speed.


We will use the following imports:

.. code:: python

    from collections import namedtuple
    import rx
    import rx.operators as ops
    import rxsci as rs
    import rxsci.container.csv as csv
    import distogram


The recommended data type when working with Maki Nage is namedtuple, hence the
first import. Then there are two RxPy imports. Since Maki Nage is based on
ReactiveX, we will use some ReactiveX operators. Then is the main RxSci import,
and another one to work with csv files. RxSci is the Maki Nage library that
implements all transformation functions. It is the library that you will mostly
use in a Maki Nage application. The final import is used for the data
exploration phase, when computing statisctis on the data distributions. 

With these import, we can start a template code that will read the csv file. The
first step consists in defining the schema of the csv file:

.. code:: python

    dataset_path = '/opt/dataset/HomeC.csv'
    parser = csv.create_line_parser(
        dtype=[
            ('time', 'int'),
            ('use', 'float'),
            ('gen', 'float'),
            ('house_overall', 'float'),
            ('dishwasher', 'float'),
            ('furnace1', 'float'),
            ('furnace2', 'float'),
            ('home_office', 'float'),
            ('fridge', 'float'),
            ('wine_cellar', 'float'),
            ('garage_door', 'float'),
            ('kitchen_12', 'float'),
            ('kitchen_14', 'float'),
            ('kitchen_38', 'float'),
            ('barn', 'float'),
            ('well', 'float'),
            ('microwave', 'float'),
            ('living_room', 'float'),
            ('solar', 'float'),
            ('temperature', 'float'),
            ('icon', 'str'),
            ('humidity', 'float'),
            ('visibility', 'float'),
            ('summary', 'str'),
            ('apparent_temperature', 'float'),
            ('pressure', 'float'),
            ('wind_speed', 'float'),
            ('cloud_cover', 'str'),
            ('wind_bearing', 'float'),
            ('precip_intensity', 'float'),
            ('dew_point', 'float'),
            ('precip_probability', 'float'),
        ]
    )


Then, loading and processing its content is done the following way:

.. code:: python

    csv.load_from_file(dataset_path, parser).pipe().run()

Many things happen on this single line: The *load_from_file* function is a
factory operator that creates an Observable from the content of the csv file. An
Observable is the term used for a stream of events. The Observable will emit one
item - one event - for each line of the csv file. The *load_from_file* operator
takes two arguments: the path of the csv file, and its schema.

The pipe operator one of the two operators that you will use most frequently: It
allows to create chaines of operators, so that data transformations can be done
in a sequential way. For now, no sequence is provided, so no transformation is
done.

Finally the run operator subscribes to the Observable. Calling it triggers that
actual execution of the whole computation graph.

No that we have our csv dataset available as an Observable, let's work on it!


Data Exploration
------------------


Overall Consumption
....................


As a first step, let's compute some statistics on the *house overall* column,
and plot its histogram. For these two steps, we need to compute a compressed
representation of the data distribution. This is done by creating a graph that
consists in two operators:

.. code:: python

    dist = csv.load_from_file(dataset_path, parser).pipe(
        ops.map(lambda i: i.house_overall),
        rs.math.dist.update(reduce=True),
    ).run()

The map operator applies a transformation function for each item emitted by the
source Observable. Note that all operators take observables as input and return
Observables as output. This allows to chain operators very easily.

The second operator is *math.dist.update*. It computes a compressed
representation of the data distribution that can be used to compute statistics
on it, such as mean, variance, and quantiles. The reduce parameter indicates
that we want the operator to emit an item only when the source Observable
completes, i.e. when all items have been processed. The default behavior is to
emit a distribution object for each item received as input.

As said previously, the run operator subscribes to the resulting Observable.
Moreover, it return that last item emitted. On our graph, this is the
distribution object.


From this object, we can do several statistic analysis. Computing classical
statistical information is done with the *describe* operator:

.. code:: python

    print("house overall consumption statistics:")
    print(rx.just(dist).pipe(
        rs.math.dist.describe(),
    ).run())

Note that we still have the same structure here, always working with
Observables: First an observable is created from the dist object, with the
*just* operator. This observable will emit a single item. This may seem
unnecessarily complex at first, but when working in a streaming way, this is
just a particular case of stream. The important point is that all the operators
that we use work on Observables, weather they emit no item, some items, or an
inifinite number of items.

The *math.dist.describe* operator emits a namedtuple with the following
statistics for each item it receives:


==========  =====================
min         0.0
max         14.71456667
mean        0.8589623950050789
stddev      1.0574552803110226
p25         0.36709267099355347
p50         0.568152839202991
p75         0.9730122941873172
==========  =====================

Another kind of information that can be computed from the dist object is the
histogram of the distribution. For this, we use once again the same pattern:

.. code:: python

    df_hist = rx.just(dist).pipe(
        rs.math.dist.histogram(),
        ops.map(lambda i: pd.DataFrame(np.array(i), columns=["bin", "count"])),
    ).run()

    fig = px.bar(df_hist, x="bin", y="count", title="house consumption")
    fig.update_layout(height=300)
    fig.show()


The *math.dist.histogram* operator computes the histogram of the distribution.
Then this histogram object is mapped to a pandas dataframe so that it can be
used by plotly. The resulting plot is:

.. image:: images/home_house_consumption.png
    :align: center
    :scale: 80%


Multiple Variable Exploration
..............................
