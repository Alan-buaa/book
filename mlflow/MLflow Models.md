# MLflow Models

An MLflow Model is a standard format for packaging machine learning models that can be used in a variety of downstream tools—for example, real-time serving through a REST API or batch inference on Apache Spark. The format defines a convention that lets you save a model in different “flavors” that can be understood by different downstream tools.

Table of Contents

- [Storage Format](https://mlflow.org/docs/latest/models.html#storage-format)
- [Model Signature And Input Example](https://mlflow.org/docs/latest/models.html#model-signature-and-input-example)
- [Model API](https://mlflow.org/docs/latest/models.html#model-api)
- [Built-In Model Flavors](https://mlflow.org/docs/latest/models.html#built-in-model-flavors)
- [Model Customization](https://mlflow.org/docs/latest/models.html#model-customization)
- [Built-In Deployment Tools](https://mlflow.org/docs/latest/models.html#built-in-deployment-tools)
- [Deployment to Custom Targets](https://mlflow.org/docs/latest/models.html#deployment-to-custom-targets)



## [Storage Format](https://mlflow.org/docs/latest/models.html#id11)

Each MLflow Model is a directory containing arbitrary files, together with an `MLmodel` file in the root of the directory that can define multiple *flavors* that the model can be viewed in.

Flavors are the key concept that makes MLflow Models powerful: they are a convention that deployment tools can use to understand the model, which makes it possible to write tools that work with models from any ML library without having to integrate each tool with each library. MLflow defines several “standard” flavors that all of its built-in deployment tools support, such as a “Python function” flavor that describes how to run the model as a Python function. However, libraries can also define and use other flavors. For example, MLflow’s [`mlflow.sklearn`](https://mlflow.org/docs/latest/python_api/mlflow.sklearn.html#module-mlflow.sklearn) library allows loading models back as a scikit-learn `Pipeline` object for use in code that is aware of scikit-learn, or as a generic Python function for use in tools that just need to apply the model (for example, the `mlflow sagemaker` tool for deploying models to Amazon SageMaker).

All of the flavors that a particular model supports are defined in its `MLmodel` file in YAML format. For example, [`mlflow.sklearn`](https://mlflow.org/docs/latest/python_api/mlflow.sklearn.html#module-mlflow.sklearn) outputs models as follows:

```
# Directory written by mlflow.sklearn.save_model(model, "my_model")
my_model/
├── MLmodel
└── model.pkl
```

And its `MLmodel` file describes two flavors:

```
time_created: 2018-05-25T17:28:53.35

flavors:
  sklearn:
    sklearn_version: 0.19.1
    pickled_model: model.pkl
  python_function:
    loader_module: mlflow.sklearn
```

This model can then be used with any tool that supports either the `sklearn` or `python_function` model flavor. For example, the `mlflow models serve` command can serve a model with the `python_function` or the `crate` (R Function) flavor:

```
mlflow models serve -m my_model
```

In addition, the `mlflow sagemaker` command-line tool can package and deploy models to AWS SageMaker as long as they support the `python_function` flavor:

```
mlflow sagemaker deploy -m my_model [other options]
```

### Fields in the MLmodel Format

Apart from a **flavors** field listing the model flavors, the MLmodel YAML format can contain the following fields:

- time_created

  Date and time when the model was created, in UTC ISO 8601 format.

- run_id

  ID of the run that created the model, if the model was saved using [MLflow Tracking](https://mlflow.org/docs/latest/tracking.html#tracking).

- signature

  [model signature](https://mlflow.org/docs/latest/models.html#model-signature) in JSON format.

- input_example

  reference to an artifact with [input example](https://mlflow.org/docs/latest/models.html#input-example).

## [Model Signature And Input Example](https://mlflow.org/docs/latest/models.html#id12)

When working with ML models you often need to know some basic functional properties of the model at hand, such as “What inputs does it expect?” and “What output does it produce?”. MLflow models can include the following additional metadata about model inputs and outputs that can be used by downstream tooling:

- [Model Signature](https://mlflow.org/docs/latest/models.html#model-signature) - description of a model’s inputs and outputs.
- [Model Input Example](https://mlflow.org/docs/latest/models.html#input-example) - example of a valid model input.

### Model Signature

The Model signature defines the schema of a model’s inputs and outputs. Model inputs and outputs are described as a sequence of (optionally) named columns with type specified as one of the [`MLflow data types`](https://mlflow.org/docs/latest/python_api/mlflow.types.html#mlflow.types.DataType). The signature is stored in JSON format in the [MLmodel file](https://mlflow.org/docs/latest/python_api/mlflow.pyfunc.html#pyfunc-model-config), together with other model metadata. Model signatures are recognized and enforced by standard [MLflow model deployment tools](https://mlflow.org/docs/latest/models.html#built-in-deployment). For example, the [mlflow models serve](https://mlflow.org/docs/latest/models.html#local-model-deployment) tool, which deploys a model as a REST API, validates inputs based on the model’s signature.

The following example displays an MLmodel file excerpt containing the model signature for a classification model trained on the [Iris dataset](https://archive.ics.uci.edu/ml/datasets/iris). The input has 4 named, numeric columns. The output is an unnamed integer specifying the predicted class:

```
signature:
    inputs: '[{"name": "sepal length (cm)", "type": "double"}, {"name": "sepal width
      (cm)", "type": "double"}, {"name": "petal length (cm)", "type": "double"}, {"name":
      "petal width (cm)", "type": "double"}]'
    outputs: '[{"type": "integer"}]'
```

