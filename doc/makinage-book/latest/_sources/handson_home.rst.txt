Home Energy Consumption
========================

The dataset
------------

On this example we will work on the *Smart Home Dataset*. It can be downloaded from
kaggle `here <https://www.kaggle.com/taranvee/smart-home-dataset-with-weather-information>`_.

The dataset is a csv file, its content looks like this:

.. code:: console

    $ head HomeC.csv
    time,use [kW],gen [kW],House overall [kW],Dishwasher [kW],Furnace 1 [kW],Furnace 2 [kW],Home office [kW],Fridge [kW],Wine cellar [kW],Garage door [kW],Kitchen 12 [kW],Kitchen 14 [kW],Kitchen 38 [kW],Barn [kW],Well [kW],Microwave [kW],Living room [kW],Solar [kW],temperature,icon,humidity,visibility,summary,apparentTemperature,pressure,windSpeed,cloudCover,windBearing,precipIntensity,dewPoint,precipProbability
    1451624400,0.932833333,0.003483333,0.932833333,3.33E-05,0.0207,0.061916667,0.442633333,0.12415,0.006983333,0.013083333,0.000416667,0.00015,0,0.03135,0.001016667,0.004066667,0.001516667,0.003483333,36.14,clear-night,0.62,10,Clear,29.26,1016.91,9.18,cloudCover,282,0,24.4,0
    1451624401,0.934333333,0.003466667,0.934333333,0,0.020716667,0.063816667,0.444066667,0.124,0.006983333,0.013116667,0.000416667,0.00015,0,0.0315,0.001016667,0.004066667,0.00165,0.003466667,36.14,clear-night,0.62,10,Clear,29.26,1016.91,9.18,cloudCover,282,0,24.4,0
    1451624402,0.931816667,0.003466667,0.931816667,1.67E-05,0.0207,0.062316667,0.446066667,0.123533333,0.006983333,0.013083333,0.000433333,0.000166667,1.67E-05,0.031516667,0.001,0.004066667,0.00165,0.003466667,36.14,clear-night,0.62,10,Clear,29.26,1016.91,9.18,cloudCover,282,0,24.4,0


.. code:: python

    from collections import namedtuple
    import rx
    import rx.operators as ops
    import rxsci as rs
    import rxsci.container.csv as csv
    import distogram


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


.. code:: python

    csv.load_from_file(dataset_path, parser).pipe().run()


Data Exploration
------------------

.. code:: python

    dist = csv.load_from_file(dataset_path, parser).pipe(
        ops.map(lambda i: i.house_overall),
        rs.math.dist.update(reduce=True),
    ).run()


Overall Consumption
....................


.. code:: python

    print("house overall consumption statistics:")
    print(rx.just(dist).pipe(
        rs.math.dist.describe(),
    ).run())


==========  =====================
min         0.0
max         14.71456667
mean        0.8589623950050789
stddev      1.0574552803110226
p25         0.36709267099355347
p50         0.568152839202991
p75         0.9730122941873172
==========  =====================


.. code:: python

    hist = distogram.histogram(dist)
    df_hist = pd.DataFrame(np.array(hist), columns=["bin", "count"])
    fig = px.bar(df_hist, x="bin", y="count", title="house consumption")
    fig.update_layout(height=300)
    fig.show()


.. image:: images/home_house_consumption.png
    :align: center
    :scale: 80%


Multiple Variable Exploration
..............................
