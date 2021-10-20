.. configuration:

Configuration
==============

The configuration of a Maki Mage application is done via a `YAML
<https://yaml.org/>`_ configuration file. This configuration is composed of
dedicated sections for each element of the application.

This configuration file is the only parameter used by the *makinage* CLI, as the
*config* parameter.

In this chapter, we associate each field with a type. The meaning of each type is:

* object: A YAML mapping
* list: A YAML collection
* string: A string
* module: A string describing a python module. The syntax follows the python "import" notation, i.e. modules separated by dots.
* function: A string describing a python function. The syntax is a module name followed by a function name. Both are separated by ":".


Here is an example configuration file:

.. code:: yaml

    application:
      name: push_house_data
    kafka:
      endpoint: "localhost"
    topics:
      - name: house_values
        encoder: makinage.encoding.string
    sources:
      house_data:
        factory: makinage_examples.house.push:read_house_data
        kwargs:
          filename: "/opt/dataset/HomeC.csv"
    sinks:
      console:
        factory: makinage_examples.house.pull:print_to_console
    regulators:
      - feedback: house_values
        control: house_data
    operators:
      forward_house_data:
        factory: makinage_examples.house.push:forward_house_data
        sources:
          - house_data
        sinks:
          - house_values    

application (object)
---------------------

This section contains information on the application.

* name: The name of the application. This name is used as the kafka consumer group.
* source_type: [string] How to work with incoming data. Default: stream. Possible values are [stream|batch]

kafka (object)
------------------

This section contains the kafka configuration. It contains the following fields:

* endpoint: [string] The kafka server endpoint. e.g. "localhost"

topics (list)
------------------

This sections describes the different topics that can be used by the operators.
Each entry contains the following fields:

* name: [string] values
* encoder: [module, optional] The encoder used for kafka records. default: "makinage.encoding.string"
* partition_selector: [function, optional]. default: int(random.random() * 1000)
* start_from: [string, optional]. Defines how records are consumed on service reload. possible values are [end|beginning|last]. default: "end"
* timestamp_mapper: [function] The mapper used to extract the timestamp value from incoming records.
* merge_lookup_depth: [integer] The depth used to order the reads on the topics partitions (in number of items). Default: 1.


sources (object)
------------------

This sections declares the different sources implemented in some connectors.
These sources available for the operators. The name of each entry is the name of
the source. Each entry contains the following fields:

* factory: [function] The function implementing the source connector.
* kwargs: [object, optional] The list of named arguments to provide to the factory.

Example:

.. code:: yaml

    sources:
      house_data:
        factory: makinage_examples.house.push:read_house_data
        kwargs:
          filename: "/opt/dataset/HomeC.csv"


sinks (object)
------------------

This sections declares the different sinks implemented in some connectors. These
sinks available for the operators. The name of each entry is the name of the
sink. Each entry contains the following fields:

* factory: [function] The function implementing the sink connector.
* kwargs: [object, optional] The list of named arguments to provide to the factory.

Example:

.. code:: yaml

    sinks:
      console:
        factory: makinage_examples.house.pull:print_to_console


regulators (list)
------------------

This section configures the regulation injection. Regulation injection limits
the rate of a source when a sink cannot process items fast enough.

Each entry contains the following fields:

* feedback: The name of the sink to monitor.
* control: The name of the source to regulate.

Any source and sink of the application can be used only once in regulators.

operators (object)
------------------

This section contains the list of operators in the application. Each operator
field is a named object of this section. Each operator object contains the
following fields:

* factory: [function] The name of the operator factory.
* sources: [list, optional] The list of source observables for this operator.
* sinks: [list, optional] The list of sink observables for this operator.

Example:

.. code:: yaml

    operators:
      forward_house_data:
        factory: makinage_examples.house.push:forward_house_data
        sources:
          - house_data
        sinks:
          - house_values  


config (object)
------------------

This section is a placeholder for application-specific configuration. This is
typically where one can put the feature engineering parameters or anything that
can be configured without code change.

This section contains some reserved sub-section keywords:

serve (object)
..............

This sub-section contains the configuration for model serving. The following
fields can be used to configure the inference behavior:

* pre_transform: [function] A factory function called with the configuration as a parameter. It must return a function that takes utterances as parameter and returns data in a format suitable for inference. 
* post_transform: [function] A factory function called with the configuration as a parameter. It must return a function that takes utterances and predicions as parameters and returns data in a format suitable for emission on the sink topic. 
* predict: [function] A factory function called with the model and configuration as a parameters. It must return a function that takes pre-transformed utterances as parameter and return the model prediction.

See the `Serving <serving.html>`_ chapter for more information on these fields.

Example:

.. code:: yaml

    config:
      serve:
        pre_transform: my_app:pre_transform
        post_transform: my_app:post_transform
        my_param1: 'foo'
        my_threshold: 0.82


redirect (object)
------------------

If the configuration file contains a *redirect* section, then the actual content
of the configuration is retrieved from the location described in this section.

It contains the following fields:

* connector: [string] The type of redirection to do. Only `consul <https://www.consul.io>`_ is supported for now.
* endpoint: [string] The consul endpoint url. e.g. "http://localhost:8500"
* key: [string] The name of the KV Store key to read. e.g. "myservice"

With Consul, the configuration changes are monitored and exposed in real-time on
the config argument of the operators. This allows to dynamically change the
configuration of a running service.


Example:

.. code:: yaml

    redirect:
      connector: "consul"
      endpoint: "http://consul:8500"
      key: "path.to.param"

