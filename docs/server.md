# Deep Model Server

The primary feature of DMS is the model server. It can be used for many types of inference in production settings. It provides an easy-to-use command line interface and utilizes an industry standard OpenAPI interface. It has features for pre-processing and post-processing inputs and outputs for customized applications.

For example, you want to make an app that lets your users snap a picture, and it'll tell them what what objects were detected in the scene and predictions on what the objects might be. You can use DMS to serve a prediction endpoint for a object detection and identification model that intakes images, then returns predictions. You can also modify DMS behavior with custom services and run multiple models. There are examples of custom services, pre-processing, post-processing in the [examples](../examples) folder. The object detection example is in [examples/ssd](../examples/ssd/README.md).

While current models and support is limited to MXNet as the underlying framework, other framework support and serving of models originating from other frameworks will be supported in the future.

## Technical Details

Now that you have a high level view of DMS, let's get a little into the weeds. DMS takes a deep learning model and it wraps it in a REST API. Currently it is bundled with the MXNet framework, and it comes with a built-in web server that you run from command line. This command line call takes in the single or multiple models you want to serve, along with additional optional parameters controlling the port, host, and logging. Additionally, you can point it to service extensions which define pre-processing and post-processing steps. DMS also comes with a default vision service that makes it easy to serve an image classification model. If you're looking to build chat bots or video understanding then you'll have some additional leg work to do with the pre-processing and post-processing steps. These are covered in more detail in the [custom service](custom_service.md) documentation.

To try out DMS serving now, you can load squeezenet, which is under 5 MB, with this example:

```bash
portico-server --models squeezenet=https://s3.amazonaws.com/model-server/models/squeezenet_v1.1/squeezenet_v1.1.model
```
Then check the [API description](http://127.0.0.1:8080/api-description). For info on other endpoints check out the [REST API documentation](rest_api.md).

For more models, check out the [model zoo](model_zoo.md).

## Model Files

The rest of this topic focus on serving of model files without much discussion on the model files themselves, where they come from, and how they're made. Long story short: it's a zip archive with the parameters, weights, and metadata that define a model that has been trained already. If you want to know more about the model files, take a look at the [export tool documentation](export.md).

## Command Line Interface

```bash
$ deep-model-server -h
usage: deep-model-server [-h] --models KEY1=VAL1,KEY2=VAL2...
                         [KEY1=VAL1,KEY2=VAL2... ...] [--service SERVICE]
                         [--gen-api GEN_API] [--port PORT] [--host HOST]
                         [--gpu]
                         [--log-file LOG_FILE]
                         [--log-rotation-time LOG_ROTATION_TIME]
                         [--log-level LOG_LEVEL]

Deep Model Serving

optional arguments:
  -h, --help            show this help message and exit
  --models KEY1=VAL1,KEY2=VAL2... [KEY1=VAL1,KEY2=VAL2... ...]
                        Models to be deployed
  --service SERVICE     Using user defined model service. By default it uses
                        mxnet_vision_service.
  --gen-api GEN_API     Generates API client for the supplied language.
                        Options include Java, C#, JavaScript and Go. For
                        complete list check out https://github.com/swagger-api
                        /swagger-codegen.
  --port PORT           Port number. By default it is 8080.
  --host HOST           Host. By default it is localhost.
  --log-file LOG_FILE   Log file name. By default it is "dms_app.log".
  --log-rotation-time LOG_ROTATION_TIME
                        Log rotation time. By default it is "1 H", which means
                        one hour. Valid format is "interval when". For weekday
                        and midnight, only "when" is required. Check https://d
                        ocs.python.org/2/library/logging.handlers.html#logging
                        .handlers.TimedRotatingFileHandler for detail values.
  --log-level LOG_LEVEL
                        Log level. By default it is INFO. Possible values are
                        NOTEST, DEBUG, INFO, ERROR AND CRITICAL.Check
                        https://docs.python.org/2/library/logging.html
                        #logging-levels
```

### Required Arguments & Defaults

Example single model usage:

```bash
deep-model-server --models name=model_location
```

`--models` is the only required argument. You can pass one or more models in a key value pair format: `name` you want to call the model and `model_location` for the local file path or URI to the model. The name is what appears in your REST API's endpoints. In the first example we used `squeezenet_v1.1` for the name, e.g. `deep-model-server --models squeezenet_v1.1=...`, and accordingly the predict endpoint was called by `http://127.0.0.1:8080/squeezenet_v1.1/predict`. In the first example this was `squeezenet=https://s3.amazonaws.com/mms-models/squeezenet_v1.1.model`. Alternatively, we could have downloaded the file and used a local file path like `squeezenet=dms_models/squeezenet_v1.1.model`.

The rest of these arguments are optional and will have the following defaults:
* [--service mxnet_vision_service]
* [--port 8080]
* [--host 127.0.0.1]

gpu argument is to specifiy whether to use gpu for inference.

Logging and exporting an SDK can also be triggered with additional arguments. Details are in the following Arguments section.

#### Arguments:
1. **models**: required, <model_name>=<model_path> pairs.

    (a) Model path can be a local file path or URI (s3 link, or http link).
        local file path: path/to/local/model/file or file://root/path/to/model/file
        s3 link: s3://S3_endpoint[:port]/...
        http link: http://hostname/path/to/resource

    (b) Currently, the model file has .model extension, it is actually a zip file with a .model extension packing trained MXNet models and model signature files. The details will be explained in **Export existing model** section.

    (c) Multiple models loading are also supported by specifying multiple name path pairs.
2. **service**: optional, the system will load input service module and will initialize MXNet models with the service defined in the module. The module should contain a valid class which extends the base model service with customized `_preprocess` and `_postprocess` functions.
3. **port**: optional, default is 8080
4. **host**: optional, default is 127.0.0.1
5. **gpu**: optional, gpu device id, such as 0 or 1. cpu will be used if this argument is not set.
5. **gen-api**: optional, this will generate an open-api formated client sdk in build folder.
6. **log-file**: optional, log file name. By default it is "dms_app.log".
7. **log-rotation-time**: optional, log rotation time. By default it is "1 H", which means one hour. Valid format is "interval when". For weekday and midnight, only "when" is required. Check https://docs.python.org/2/library/logging.handlers.html#logging.handlers.TimedRotatingFileHandler for detail values.
8. **log-level**: optional, log level. By default it is INFO. Possible values are NOTEST, DEBUG, INFO, ERROR AND CRITICAL. Check https://docs.python.org/2/library/logging.html#logging-levels

## Advanced Features

### Custom Names

You can change the name of the model prediction endpoint to be whatever you want. In our previous examples we used `squeezenet_v1.1=<url>`, but we can shorten that or use whatever name we want. For example, let's use `squeezenet` instead:

```bash
deep-model-server --models squeezenet=https://s3.amazonaws.com/model-server/models/squeezenet_v1.1/squeezenet_v1.1.model
```

### Custom Services

This topic is covered in much more detail on the [custom service documentation page](custom_service.md), but let's talk about how you start up your DMS server using a custom service and why you might want one.
Let's say you have a model named `super-fancy-net.model` that can detect a lot of things, but you want an API endpoint that detects only hotdogs. You would use a name that makes sense for it, such as the "not-hot-dog" API. In this case we might invoke DMS like this:

```bash
deep-model-server --models not-hot-dog=https://s3.amazonaws.com/model-server/models/squeezenet_v1.1/super-fancy-net.model
```

You would also want to customize and limit DMS's inference with a custom service, put that code into a Python file (e.g. nothotdog.py) along with the model file, and call that script with the `--service` argument as in this example:

```bash
deep-model-server --models not-hot-dog=https://s3.amazonaws.com/model-server/models/squeezenet_v1.1/super-fancy-net.model --service nothotdog.py
```

This would serve a prediction endpoint at `/not-hot-dog/predict` and run your custom service code that is located in `nothotdog.py`. For more info on custom services, check out the [object detection example](../examples/ssd/README.md) and the [custom service documentation](custom_service.md).

### Serving Multiple Models with DMS

Example multiple model usage:

```bash
deep-model-server --models name=model_location name2=model_location2
```

Here's an example for running the resnet-18 and the vgg16 models using local model files.

```bash
deep-model-server --models resnet-18=file://models/resnet-18 squeezenet=file://models/squeezenet_v1.1
```

If you don't have the model files locally, then you can call DMS using URLs to the model files.

```bash
deep-model-server --models resnet=https://s3.amazonaws.com/model-server/models/resnet-18/resnet-18.model squeezenet=https://s3.amazonaws.com/model-server/models/squeezenet_v1.1/squeezenet_v1.1.model
```

This will setup a local host serving resnet-18 model and squeezenet model on the same port, using the default 8080. Check http://127.0.0.1:8080/api-description to see that each model has an endpoint for prediction. In this case you would see `resnet/predict` and `squeezenet/predict`/

Note that if you supply a [custom service](custom_service.md) for pre or post-processing, both models will use that same pipeline. There is currently no support for using different pipelines per-model.