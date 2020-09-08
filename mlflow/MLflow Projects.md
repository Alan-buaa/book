# MLflow Projects

An MLflow Project is a format for packaging data science code in a reusable and reproducible way, based primarily on conventions. In addition, the Projects component includes an API and command-line tools for running projects, making it possible to chain together projects into workflows.

Table of Contents

- [Overview](https://mlflow.org/docs/latest/projects.html#overview)
- [Specifying Projects](https://mlflow.org/docs/latest/projects.html#specifying-projects)
- [Running Projects](https://mlflow.org/docs/latest/projects.html#running-projects)
- [Iterating Quickly](https://mlflow.org/docs/latest/projects.html#iterating-quickly)
- [Building Multistep Workflows](https://mlflow.org/docs/latest/projects.html#building-multistep-workflows)

## [Overview](https://mlflow.org/docs/latest/projects.html#id8)

At the core, MLflow Projects are just a convention for organizing and describing your code to let other data scientists (or automated tools) run it. Each project is simply a directory of files, or a Git repository, containing your code. MLflow can run some projects based on a convention for placing files in this directory (for example, a `conda.yaml` file is treated as a [Conda](https://conda.io/docs) environment), but you can describe your project in more detail by adding a `MLproject` file, which is a [YAML](https://learnxinyminutes.com/docs/yaml/) formatted text file. Each project can specify several properties:

- Name

  A human-readable name for the project.

- Entry Points

  Commands that can be run within the project, and information about their parameters. Most projects contain at least one entry point that you want other users to call. Some projects can also contain more than one entry point: for example, you might have a single Git repository containing multiple featurization algorithms. You can also call any `.py` or `.sh` file in the project as an entry point. If you list your entry points in a `MLproject` file, however, you can also specify *parameters* for them, including data types and default values.

- Environment

  The software environment that should be used to execute project entry points. This includes all library dependencies required by the project code. See [Project Environments](https://mlflow.org/docs/latest/projects.html#project-environments) for more information about the software environments supported by MLflow Projects, including [Conda environments](https://mlflow.org/docs/latest/projects.html#project-conda-environments) and [Docker containers](https://mlflow.org/docs/latest/projects.html#project-docker-container-environments).

You can run any project from a Git URI or from a local directory using the `mlflow run` command-line tool, or the [`mlflow.projects.run()`](https://mlflow.org/docs/latest/python_api/mlflow.projects.html#mlflow.projects.run) Python API. These APIs also allow submitting the project for remote execution on [Databricks](https://mlflow.org/docs/latest/projects.html#databricks-execution) and [Kubernetes](https://mlflow.org/docs/latest/projects.html#kubernetes-execution).

> By default, MLflow uses a new, temporary working directory for Git projects. This means that you should generally pass any file arguments to MLflow project using absolute, not relative, paths. If your project declares its parameters, MLflow automatically makes paths absolute for parameters of type `path`.

## [Specifying Projects](https://mlflow.org/docs/latest/projects.html#id9)

By default, any Git repository or local directory can be treated as an MLflow project; you can invoke any bash or Python script contained in the directory as a project entry point. The [Project Directories](https://mlflow.org/docs/latest/projects.html#project-directories) section describes how MLflow interprets directories as projects.

To provide additional control over a project’s attributes, you can also include an [MLproject file](https://mlflow.org/docs/latest/projects.html#mlproject-file) in your project’s repository or directory.

Finally, MLflow projects allow you to specify the software [environment](https://mlflow.org/docs/latest/projects.html#project-environments) that is used to execute project entry points.

### Project Environments

MLflow currently supports the following project environments: Conda environment, Docker container environment, and system environment.

- Conda environment

  [Conda](https://conda.io/docs) environments support both Python packages and native libraries (e.g, CuDNN or Intel MKL). When an MLflow Project specifies a Conda environment, it is activated before project code is run.By default, MLflow uses the system path to find and run the `conda` binary. You can use a different Conda installation by setting the `MLFLOW_CONDA_HOME` environment variable; in this case, MLflow attempts to run the binary at `$MLFLOW_CONDA_HOME/bin/conda`.You can specify a Conda environment for your MLflow project by including a `conda.yaml` file in the root of the project directory or by including a `conda_env` entry in your `MLproject` file. For details, see the [Project Directories](https://mlflow.org/docs/latest/projects.html#project-directories) and [Specifying an Environment](https://mlflow.org/docs/latest/projects.html#mlproject-specify-environment) sections.

- Docker container environment

  [Docker containers](https://www.docker.com/resources/what-container) allow you to capture non-Python dependencies such as Java libraries.When you run an MLflow project that specifies a Docker image, MLflow adds a new Docker layer that copies the project’s contents into the `/mlflow/projects/code` directory. This step produces a new image. MLflow then runs the new image and invokes the project entrypoint in the resulting container.Environment variables, such as `MLFLOW_TRACKING_URI`, are propagated inside the Docker container during project execution. Additionally, [runs](https://mlflow.org/docs/latest/concepts.html#concepts) and [experiments](https://mlflow.org/docs/latest/tracking.html#organizing-runs-in-experiments) created by the project are saved to the tracking server specified by your [tracking URI](https://mlflow.org/docs/latest/tracking.html#where-runs-are-recorded). When running against a local tracking URI, MLflow mounts the host system’s tracking directory (e.g., a local `mlruns` directory) inside the container so that metrics, parameters, and artifacts logged during project execution are accessible afterwards.See [Dockerized Model Training with MLflow](https://github.com/mlflow/mlflow/tree/master/examples/docker) for an example of an MLflow project with a Docker environment.To specify a Docker container environment, you *must* add an [MLproject file](https://mlflow.org/docs/latest/projects.html#mlproject-file) to your project. For information about specifying a Docker container environment in an `MLproject` file, see [Specifying an Environment](https://mlflow.org/docs/latest/projects.html#mlproject-specify-environment).

- System environment

  You can also run MLflow Projects directly in your current system environment. All of the project’s dependencies must be installed on your system prior to project execution. The system environment is supplied at runtime. It is not part of the MLflow Project’s directory contents or `MLproject` file. For information about using the system environment when running a project, see the `Environment` parameter description in the [Running Projects](https://mlflow.org/docs/latest/projects.html#running-projects) section.

### Project Directories

When running an MLflow Project directory or repository that does *not* contain an `MLproject` file, MLflow uses the following conventions to determine the project’s attributes:

- The project’s name is the name of the directory.
- The [Conda environment](https://conda.io/docs/user-guide/tasks/manage-environments.html#create-env-file-manually) is specified in `conda.yaml`, if present. If no `conda.yaml` file is present, MLflow uses a Conda environment containing only Python (specifically, the latest Python available to Conda) when running the project.
- Any `.py` and `.sh` file in the project can be an entry point. MLflow uses Python to execute entry points with the `.py` extension, and it uses bash to execute entry points with the `.sh` extension. For more information about specifying project entrypoints at runtime, see [Running Projects](https://mlflow.org/docs/latest/projects.html#running-projects).
- By default, entry points do not have any parameters when an `MLproject` file is not included. Parameters can be supplied at runtime via the `mlflow run` CLI or the [`mlflow.projects.run()`](https://mlflow.org/docs/latest/python_api/mlflow.projects.html#mlflow.projects.run) Python API. Runtime parameters are passed to the entry point on the command line using `--key value` syntax. For more information about running projects and with runtime parameters, see [Running Projects](https://mlflow.org/docs/latest/projects.html#running-projects).

### MLproject File

You can get more control over an MLflow Project by adding an `MLproject` file, which is a text file in YAML syntax, to the project’s root directory. The following is an example of an `MLproject` file:

```
name: My Project

conda_env: my_env.yaml
# Can have a docker_env instead of a conda_env, e.g.
# docker_env:
#    image:  mlflow-docker-example

entry_points:
  main:
    parameters:
      data_file: path
      regularization: {type: float, default: 0.1}
    command: "python train.py -r {regularization} {data_file}"
  validate:
    parameters:
      data_file: path
    command: "python validate.py {data_file}"
```

The file can specify a name and [a Conda or Docker environment](https://mlflow.org/docs/latest/projects.html#mlproject-specify-environment), as well as more detailed information about each entry point. Specifically, each entry point defines a [command to run](https://mlflow.org/docs/latest/projects.html#mlproject-command-syntax) and [parameters to pass to the command](https://mlflow.org/docs/latest/projects.html#project-parameters) (including data types).

#### Specifying an Environment

This section describes how to specify Conda and Docker container environments in an `MLproject` file. `MLproject` files cannot specify *both* a Conda environment and a Docker environment.

* Conda environment

Include a top-level `conda_env` entry in the `MLproject` file. The value of this entry must be a *relative* path to a [Conda environment YAML file](https://conda.io/docs/user-guide/tasks/manage-environments.html#create-env-file-manually) within the MLflow project’s directory. In the following example:

```
conda_env: files/config/conda_environment.yaml
```

`conda_env` refers to an environment file located at `<MLFLOW_PROJECT_DIRECTORY>/files/config/conda_environment.yaml`, where `<MLFLOW_PROJECT_DIRECTORY>` is the path to the MLflow project’s root directory.

* Docker container environment

Include a top-level `docker_env` entry in the `MLproject` file. The value of this entry must be the name of a Docker image that is accessible on the system executing the project; this image name may include a registry path and tags. Here are a couple of examples.

**Example 1: Image without a registry path**

```
docker_env:
  image: mlflow-docker-example-environment
```

In this example, `docker_env` refers to the Docker image with name `mlflow-docker-example-environment` and default tag `latest`. Because no registry path is specified, Docker searches for this image on the system that runs the MLflow project. If the image is not found, Docker attempts to pull it from [DockerHub](https://hub.docker.com/).

**Example 2: Mounting volumes and specifying environment variables**

You can also specify local volumes to mount in the docker image (as you normally would with Docker’s -v option), and additional environment variables (as per Docker’s -e option). Environment variables can either be copied from the host system’s environment variables, or specified as new variables for the Docker environment. The environment field should be a list. Elements in this list can either be lists of two strings (for defining a new variable) or single strings (for copying variables from the host system). For example:

```
docker_env:
  image: mlflow-docker-example-environment
  volumes: ["/local/path:/container/mount/path"]
  environment: [["NEW_ENV_VAR", "new_var_value"], "VAR_TO_COPY_FROM_HOST_ENVIRONMENT"]
```

In this example our docker container will have one additional local volume mounted, and two additional environment variables: one newly-defined, and one copied from the host system.

**Example 3: Image in a remote registry**

```
docker_env:
  image: 012345678910.dkr.ecr.us-west-2.amazonaws.com/mlflow-docker-example-environment:7.0
```

In this example, docker_env refers to the Docker image with name mlflow-docker-example-environment and tag 7.0 in the Docker registry with path 012345678910.dkr.ecr.us-west-2.amazonaws.com, which corresponds to an Amazon ECR registry. When the MLflow project is run, Docker attempts to pull the image from the specified registry. The system executing the MLflow project must have credentials to pull this image from the specified registry.

#### Command Syntax

When specifying an entry point in an `MLproject` file, the command can be any string in Python [format string syntax](https://docs.python.org/2/library/string.html#formatstrings). All of the parameters declared in the entry point’s `parameters` field are passed into this string for substitution. If you call the project with additional parameters *not* listed in the `parameters` field, MLflow passes them using `--key value` syntax, so you can use the `MLproject` file to declare types and defaults for just a subset of your parameters.

Before substituting parameters in the command, MLflow escapes them using the Python [shlex.quote](https://docs.python.org/3/library/shlex.html#shlex.quote) function, so you don’t need to worry about adding quotes inside your command field.

#### Specifying Parameters

MLflow allows specifying a data type and default value for each parameter. You can specify just the data type by writing:

```
parameter_name: data_type
```

in your YAML file, or add a default value as well using one of the following syntaxes (which are equivalent in YAML):

```
parameter_name: {type: data_type, default: value}  # Short syntax

parameter_name:     # Long syntax
  type: data_type
  default: value
```

MLflow supports four parameter types, some of which it treats specially (for example, downloading data to local files). Any undeclared parameters are treated as `string`. The parameter types are:

- string

  A text string.

- float

  A real number. MLflow validates that the parameter is a number.

- path

  A path on the local file system. MLflow converts any relative `path` parameters to absolute paths. MLflow also downloads any paths passed as distributed storage URIs (`s3://`, `dbfs://`, gs://, etc.) to local files. Use this type for programs that can only read local files.

- uri

  A URI for data either in a local or distributed storage system. MLflow converts relative paths to absolute paths, as in the `path` type. Use this type for programs that know how to read from distributed storage (e.g., programs that use Spark).

## [Running Projects](https://mlflow.org/docs/latest/projects.html#id10)

MLflow provides two ways to run projects: the `mlflow run` [command-line tool](https://mlflow.org/docs/latest/cli.html#cli), or the [`mlflow.projects.run()`](https://mlflow.org/docs/latest/python_api/mlflow.projects.html#mlflow.projects.run) Python API. Both tools take the following parameters:

- Project URI

  A directory on the local file system or a Git repository path, specified as a URI of the form `https://<repo>` (to use HTTPS) or `user@host:path` (to use Git over SSH). To run against an MLproject file located in a subdirectory of the project, add a ‘#’ to the end of the URI argument, followed by the relative path from the project’s root directory to the subdirectory containing the desired project.

- Project Version

  For Git-based projects, the commit hash or branch name in the Git repository.

- Entry Point

  The name of the entry point, which defaults to `main`. You can use any entry point named in the `MLproject` file, or any `.py` or `.sh` file in the project, given as a path from the project root (for example, `src/test.py`).

- Parameters

  Key-value parameters. Any parameters with [declared types](https://mlflow.org/docs/latest/projects.html#project-parameters) are validated and transformed if needed.

- Deployment Mode

  Both the command-line and API let you [launch projects remotely](https://mlflow.org/docs/latest/projects.html#databricks-execution) in a [Databricks](https://databricks.com/) environment. This includes setting cluster parameters such as a VM type. Of course, you can also run projects on any other computing infrastructure of your choice using the local version of the `mlflow run` command (for example, submit a script that does `mlflow run` to a standard job queueing system).You can also launch projects remotely on [Kubernetes](https://kubernetes.io/) clusters using the `mlflow run` CLI (see [Run an MLflow Project on Kubernetes (experimental)](https://mlflow.org/docs/latest/projects.html#kubernetes-execution)).

- Environment

  By default, MLflow Projects are run in the environment specified by the project directory or the `MLproject` file (see [Specifying Project Environments](https://mlflow.org/docs/latest/projects.html#project-environments)). You can ignore a project’s specified environment and run the project in the current system environment by supplying the `--no-conda` flag.

For example, the tutorial creates and publishes an MLflow Project that trains a linear model. The project is also published on GitHub at https://github.com/mlflow/mlflow-example. To run this project:

```
mlflow run git@github.com:mlflow/mlflow-example.git -P alpha=0.5
```

There are also additional options for disabling the creation of a Conda environment, which can be useful if you quickly want to test a project in your existing shell environment.

### Run an MLflow Project on Databricks

You can run MLflow Projects remotely on Databricks. To use this feature, you must have an enterprise Databricks account (Community Edition is not supported) and you must have set up the [Databricks CLI](https://github.com/databricks/databricks-cli). Find detailed instructions in the Databricks docs ([Azure Databricks](https://docs.microsoft.com/en-us/azure/databricks/applications/mlflow/projects#run-an-mlflow-project), [Databricks on AWS](https://docs.databricks.com/applications/mlflow/projects.html)).

### Run an MLflow Project on Kubernetes (experimental)

> As an experimental feature, the API is subject to change.

You can run MLflow Projects with [Docker environments](https://mlflow.org/docs/latest/projects.html#project-docker-container-environments) on Kubernetes. The following sections provide an overview of the feature, including a simple Project execution guide with examples.

To see this feature in action, you can also refer to the [Docker example](https://github.com/mlflow/mlflow/tree/master/examples/docker), which includes the required Kubernetes backend configuration (`kubernetes_backend.json`) and [Kubernetes Job Spec](https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/#writing-a-job-spec) (`kubernetes_job_template.yaml`) files.

#### How it works

When you run an MLflow Project on Kubernetes, MLflow constructs a new Docker image containing the Project’s contents; this image inherits from the Project’s [Docker environment](https://mlflow.org/docs/latest/projects.html#project-docker-container-environments). MLflow then pushes the new Project image to your specified Docker registry and starts a [Kubernetes Job](https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/) on your specified Kubernetes cluster. This Kubernetes Job downloads the Project image and starts a corresponding Docker container. Finally, the container invokes your Project’s [entry point](https://mlflow.org/docs/latest/projects.html#running-projects), logging parameters, tags, metrics, and artifacts to your [MLflow tracking server](https://mlflow.org/docs/latest/tracking.html#tracking-server).

#### Execution guide

You can run your MLflow Project on Kubernetes by following these steps:

1. Add a Docker environment to your MLflow Project, if one does not already exist. For reference, see [Specifying an Environment](https://mlflow.org/docs/latest/projects.html#mlproject-specify-environment).

2. Create a backend configuration JSON file with the following entries:

   - `kube-context` The [Kubernetes context](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/#context) where MLflow will run the job. If not provided, MLflow will use the current context. If no context is available, MLflow will assume it is running in a Kubernetes cluster and it will use the Kubernetes service account running the current pod (‘in-cluster’ configuration).
   - `repository-uri` The URI of the docker repository where the Project execution Docker image will be uploaded (pushed). Your Kubernetes cluster must have access to this repository in order to run your MLflow Project.
   - `kube-job-template-path` The path to a YAML configuration file for your Kubernetes Job - a [Kubernetes Job Spec](https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/#writing-a-job-spec). MLflow reads the Job Spec and replaces certain fields to facilitate job execution and monitoring; MLflow does not modify the original template file. For more information about writing Kubernetes Job Spec templates for use with MLflow, see the [Job Templates](https://mlflow.org/docs/latest/projects.html#kubernetes-execution-job-templates) section.

   **Example Kubernetes backend configuration**

   ```
   {
     "kube-context": "docker-for-desktop",
     "repository-uri": "username/mlflow-kubernetes-example",
     "kube-job-template-path": "/Users/username/path/to/kubernetes_job_template.yaml"
   }
   ```

   

3. If necessary, obtain credentials to access your Project’s Docker and Kubernetes resources, including:

   - The [Docker environment image](https://mlflow.org/docs/latest/projects.html#mlproject-specify-environment) specified in the MLproject file.
   - The Docker repository referenced by `repository-uri` in your backend configuration file.
   - The [Kubernetes context](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/#context) referenced by `kube-context` in your backend configuration file.

   MLflow expects these resources to be accessible via the [docker](https://docs.docker.com/engine/reference/commandline/cli/) and [kubectl](https://kubernetes.io/docs/reference/kubectl/kubectl/) CLIs before running the Project.

4. Run the Project using the MLflow Projects CLI or [`Python API`](https://mlflow.org/docs/latest/python_api/mlflow.projects.html#mlflow.projects.run), specifying your Project URI and the path to your backend configuration file. For example:

   ```
   mlflow run <project_uri> --backend kubernetes --backend-config examples/docker/kubernetes_config.json
   ```

   where `<project_uri>` is a Git repository URI or a folder.

#### Job Templates

MLflow executes Projects on Kubernetes by creating [Kubernetes Job resources](https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/). MLflow creates a Kubernetes Job for an MLflow Project by reading a user-specified [Job Spec](https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/#writing-a-job-spec). When MLflow reads a Job Spec, it formats the following fields:

- `metadata.name` Replaced with a string containing the name of the MLflow Project and the time of Project execution
- `spec.template.spec.container[0].name` Replaced with the name of the MLflow Project
- `spec.template.spec.container[0].image` Replaced with the URI of the Docker image created during Project execution. This URI includes the Docker image’s digest hash.
- `spec.template.spec.container[0].command` Replaced with the Project entry point command specified when executing the MLflow Project.

The following example shows a simple Kubernetes Job Spec that is compatible with MLflow Project execution. Replaced fields are indicated using bracketed text.

**Example Kubernetes Job Spec**

```
apiVersion: batch/v1
kind: Job
metadata:
  name: "{replaced with MLflow Project name}"
  namespace: mlflow
spec:
  ttlSecondsAfterFinished: 100
  backoffLimit: 0
  template:
    spec:
      containers:
      - name: "{replaced with MLflow Project name}"
        image: "{replaced with URI of Docker image created during Project execution}"
        command: ["{replaced with MLflow Project entry point command}"]
        env: ["{appended with MLFLOW_TRACKING_URI, MLFLOW_RUN_ID and MLFLOW_EXPERIMENT_ID}"]
      resources:
        limits:
          memory: 512Mi
        requests:
          memory: 256Mi
      restartPolicy: Never
```

The `container.name`, `container.image`, and `container.command` fields are only replaced for the *first* container defined in the Job Spec. Further, the `MLFLOW_TRACKING_URI`, `MLFLOW_RUN_ID` and `MLFLOW_EXPERIMENT_ID` are appended to `container.env`. Use `KUBE_MLFLOW_TRACKING_URI` to pass a different tracking URI to the job container from the standard `MLFLOW_TRACKING_URI`. All subsequent container definitions are applied without modification.

## [Iterating Quickly](https://mlflow.org/docs/latest/projects.html#id11)

If you want to rapidly develop a project, we recommend creating an `MLproject` file with your main program specified as the `main` entry point, and running it with `mlflow run .`. To avoid having to write parameters repeatedly, you can add default parameters in your `MLproject` file.

## [Building Multistep Workflows](https://mlflow.org/docs/latest/projects.html#id12)

The [`mlflow.projects.run()`](https://mlflow.org/docs/latest/python_api/mlflow.projects.html#mlflow.projects.run) API, combined with [`mlflow.tracking`](https://mlflow.org/docs/latest/python_api/mlflow.tracking.html#module-mlflow.tracking), makes it possible to build multi-step workflows with separate projects (or entry points in the same project) as the individual steps. Each call to [`mlflow.projects.run()`](https://mlflow.org/docs/latest/python_api/mlflow.projects.html#mlflow.projects.run) returns a run object, that you can use with [`mlflow.tracking`](https://mlflow.org/docs/latest/python_api/mlflow.tracking.html#module-mlflow.tracking) to determine when the run has ended and get its output artifacts. These artifacts can then be passed into another step that takes `path` or `uri` parameters. You can coordinate all of the workflow in a single Python program that looks at the results of each step and decides what to submit next using custom code. Some example uses cases for multi-step workflows include:

- Modularizing Your Data Science Code

  Different users can publish reusable steps for data featurization, training, validation, and so on, that other users or team can run in their workflows. Because MLflow supports Git versioning, another team can lock their workflow to a specific version of a project, or upgrade to a new one on their own schedule.

- Hyperparameter Tuning

  Using [`mlflow.projects.run()`](https://mlflow.org/docs/latest/python_api/mlflow.projects.html#mlflow.projects.run) you can launch multiple runs in parallel either on the local machine or on a cloud platform like Databricks. Your driver program can then inspect the metrics from each run in real time to cancel runs, launch new ones, or select the best performing run on a target metric.

- Cross-validation

  Sometimes you want to run the same training code on different random splits of training and validation data. With MLflow Projects, you can package the project in a way that allows this, for example, by taking a random seed for the train/validation split as a parameter, or by calling another project first that can split the input data.

For an example of how to construct such a multistep workflow, see the MLflow [Multistep Workflow Example project](https://github.com/mlflow/mlflow/tree/master/examples/multistep_workflow).