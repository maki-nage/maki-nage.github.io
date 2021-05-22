Get Started
============

Evaluate and train a Hoeffding Tree Classifier from a stream of events:

.. code:: Python

    import rx
    import rx.operators as ops

    import rxsci as rs
    import rxsci_river as rsr
    from river import synth
    from river import tree
    from river import metrics


    source = synth.ConceptDriftStream(
        stream=synth.SEA(seed=42, variant=0),
        drift_stream=synth.SEA(seed=42, variant=1),
        seed=1, position=500, width=50,
    )


    rx.from_(source).pipe(
        ops.take(10000),
        #ops.do_action(print),
        ops.map(lambda i: rsr.Utterance(i[0], i[1])),
        rsr.evaluate.prequential(
            tree.HoeffdingAdaptiveTreeClassifier(
                grace_period=100,
                split_confidence=1e-5,
                leaf_prediction='nb',
                nb_threshold=10,
                seed=0,
            )
        ),
        rsr.compute_metric(metrics.Accuracy())
    ).subscribe(
        on_next=print,
        on_completed=lambda: print("Done!"),
    )
