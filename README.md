# SDF Dagster Demo

This is a demo of how to use SDF with Dagster. The demo is based on the Moms Flower Shop SDF workspace, which is a simple workspace that models a flower shop's sales data, using the bronze, silver, and gold data modeling pattern.

**Warning**: For those arriving from the Dagster Deep Dive, some of the features demoed, such as caching and selective materialization, are not yet available in the latest release of Dagster. These will be made available in the next patch release.

## Preqrequisites
- Install [pyenv](https://github.com/pyenv/pyenv) or a python version manager of your choice
- Ensure python 3.11 is installed

## Tutorial

### Step 1: Setting up your Python environment

It's important to use a virtual environment to avoid conflicts with other projects. Here's how you can set up a virtual environment using pyenv:

```bash
pyenv virtualenv 3.11 dgdemo_env
pyenv activate dgdemo_env
```

Above, we created a virtual environment called `dgdemo_env` and activated it. You can deactivate the virtual environment by running `pyenv deactivate`.

### Step 2: Install Dagster and SDF

We will install Dagster and SDF using pip:

```bash
pip install dagster-sdf dagster-webserver
```

The `dagster-sdf` package will install the components necessary to integrate with your SDF workspace. It also includes the `sdf` CLI tool, which you can use to interact with your workspace. Run the following to validate the installation:

```bash
$ sdf --version

sdf 0.3.23

$ dagster-sdf --help

Usage: dagster-sdf [OPTIONS] COMMAND [ARGS]...
CLI tools for working with Dagster and sdf. 

--help  -h        Show this message and exit.
workspace   Commands for using an sdf workspace in Dagster. 
```

### Step 3: Create a new Moms Flower Shop sample workspace

If you're running this demo from this repository, you can skip the below steps and jump to Step 4.

We will create a new Moms Flower Shop workspace using the `sdf new` command. This command will create a new workspace in the current directory called `moms_flower_shop_completed`. Run the following command:

```bash
sdf new --sample moms_flower_shop_completed
```

You should see a new directory called `moms_flower_shop_completed` in your current directory. This directory contains the completed Moms Flower Shop workspace. 

### Step 4: Navigate to the New workspace and Run the SDF CLI

Navigate to the new workspace directory and run the following commands to compile, check, and run the workspace:

```bash
cd moms_flower_shop_completed
sdf compile
sdf check
sdf run
```

You should notice that the workspace compiles successfully, has a failing check, and runs successfully. The failing check is intentional and is used to demonstrate how the SDF CLI can be used to check the workspace for errors.

For a tutorial of using SDF with the Moms Flower Shop workspace, see our [tutorial documentation](https://docs.sdf.com/tutorials/tutorials-intro).

### Step 5: Scaffold an SDF Workspace

This will prompt you to enter a project name, like `moms_flower_shop_dagster`. The scaffolded project will contain the necessary files to integrate with Dagster.

```bash
dagster-sdf workspace scaffold --project-name moms_flower_shop_dagster
```

### Step 6: Explore the scaffolded project

The scaffolded project will contain the following files:

moms_flower_shop_dagster/
|-- moms_flower_shop_dagster/
|   |-- __init__.py
|   |-- assets.py
|   |-- constants.py
|   |-- definitions.py
|   |-- schedules.yaml
|-- pyproject.toml
|-- setup.py

Take a look at the `assets.py` file, which contains your workspaces `sdf_assets` definition, which will create you Dagster Asset Dag, from the outputs of an `sdf compile --save table-deps,info-schema` command.

```python
from dagster import AssetExecutionContext
from dagster_sdf import SdfCliResource, SdfWorkspace, sdf_assets
import polars as pl

from .constants import sdf_workspace_dir

target_dir = sdf_workspace_dir.joinpath("sdf_dagster_out")
environment = "dbg"
workspace = SdfWorkspace(workspace_dir=sdf_workspace_dir, target_dir=target_dir, environment=environment)


@sdf_assets(workspace=workspace)
def moms_flower_shop_sdf_assets(context: AssetExecutionContext, sdf: SdfCliResource):
    yield from sdf.cli(["run", "--save", "info-schema"], target_dir=target_dir, environment=environment, context=context).stream()
```

Notice that we define the `target_dir` and `environment` variables, which are used to specify the output directory for sdf operations and metadata and the environment to run the workspace with (default: dbg). We then create an `SdfWorkspace` object, which wraps the sdf workspace directory, providing a convenient way for dagster to interact with the workspace.

The `definitions.py` file tells dagster what context to pass into asset executions and which assets makeup your dag.

Check out the [dagster docs](https://docs.dagster.io/concepts) for more information on dagster native concepts.

### Step 7: Start the Dagster Development Server

Enter the project directory that was just scaffolded and start the Dagster development server:

```bash
cd moms_flower_shop_dagster
DAGSTER_SDF_COMPILE_ON_LOAD=1 dagster dev
```

Notice the `DAGSTER_SDF_COMPILE_ON_LOAD=1` environment variable, which tells the Dagster development server to compile the workspace on load. This is necessary to expose SDF's rich compile-time metadata to Dagster.

Navigate to the dagster development server UI in your browser (usually at `http://localhost:3000`) and explore the workspace. You should see the `moms_flower_shop_sdf_assets` asset, which represents your workspace assets. 

Click on view lineage to visualize your SDF Workspace asset dag, and explore the metadata, available at compile time, like `dagster/column_schema` in the Metadata section. Just click [Show Table Schema].

![gif](https://cdn.sdf.com/img/dagster-sdf-gif-1.gif)

### Step 8: Materialize all assets in the Dagster UI and explore the metadata, like Materialized from Cache and Execution Duration

In the top right corner of the lineage UI you should see a `Materialize all` button. Click it to materialize all assets in the workspace. 

Notice that, behind the scenes SDF is executing your sample workspace entirely locally, using the [SDF DB](https://docs.sdf.com/database/introduction#sdf-db-overview). With everything executing locally against a blazing fast execution engine, you should see the assets materialize in milli-seconds. (Note: The dagster UI may take a few seconds to update)

You should notice your assets are materialized, and you can explore the metadata, like `Materialized from Cache` and `Execution Duration`, which are available at runtime.

### Step 9: Materialize again... and again... and again

SDF is designed to be fast and efficient, and it caches the results of your workspace operations. This means that if you materialize an asset that has already been materialized, SDF will not rerun the workspace operation. Instead, it will surface the cached result, and you can see this in the Dagster UI by observing the `Materialized from Cache` metadata key.

Clicking Materialize All without making any changes to the workspace will show that the assets are materialized from cache. This holds true for both local and remote execution.

### Step 10: (Optional) Break someting! 

Navigate to the workspace again and introduce a breaking change. Ideally choose a model like `APP_INSTALLS_V2` that has downstream deps.

Once you've sufficiently broken the workspace :) run `sdf compile` to catch the error.

This is a key feature of SDF, as it allows you to catch errors at compile time, before you run the workspace. This is especially useful when you have a large workspace with many dependencies, as it allows you to catch errors early and avoid wasting time running a broken workspace.

Without implementing any fixes, try to materialize the broken asset and observe the failure in the Dagster UI.

You should notice that SDF only reran the modified asset, and the downstream assets were not rerun due to the detected upstream failure. This is a key feature of SDF, as it allows you to avoid rerunning the entire workspace when you make a change, even if it is breaking.

### Step 11: Fix the breaking change and re-materialize all assets

Again, navigate to the workspace and fix the breaking change. Once you've fixed the change, run `sdf compile` to validate that everything works.

Navigate back to the Dagster UI and materialize all assets. You should notice that SDF only reran the modified asset and that the downstream assets were rerun because their upstream table was changed!

# Next Steps

Follow the Getting Started guide in the [SDF documentation](https://docs.sdf.com/integrations/dagster/getting-started) to learn more about setting up an existing or new SDF Workspace with Dagster.