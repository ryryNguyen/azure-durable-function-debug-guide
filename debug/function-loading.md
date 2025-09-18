# Overview

Azure Function is built on top of the Azure WebJobs SDK, which provides a powerful and flexible way to run background
tasks in the cloud. However, when working with Azure Durable Functions in Python, you may encounter some challenges
related to function loading and debugging.
This guide aims to help you understand how function loading works in Azure Durable Functions and provide tips for
debugging common issues.

# Order of Function Loading

The complete sequence from the deployment of
your [Python v2 Azure Function](https://learn.microsoft.com/en-us/azure/azure-functions/functions-reference-python?tabs=get-started%2Casgi%2Capplication-level&pivots=python-mode-decorators)
to it being ready to execute a new request
involves several stages of initialization between the Azure Functions host and the isolated worker process where your
code runs. This is a multi-step process that ensures everything is configured correctly before it starts handling live
traffic.

## Function App deployment and initialization

1. **Deployment**: You use the Core Tools or a CI/CD pipeline to package your function app code and its dependencies
   into a
   deployment package, which is uploaded to Azure.
2. **Function App Start**: The Azure Functions infrastructure receives the deployment and starts up a new Function App
   instance on a virtual machine.
3. **Host Process Initialization**: The Functions host process, which is part of the Azure Functions runtime and runs
   separately from your code, starts up. It reads the host.json file to apply configurations for triggers, bindings, and
   other settings.
4. **Configuration and Warmup**: If configured, the host can perform "warmup" actions to get the instance ready for
   traffic,
   such as pre-loading dependencies. This is especially useful for preventing cold starts.

## Isolated Python worker process startup

1. **Worker Process Creation**: The Functions host starts up the isolated Python worker process. This worker process is
   an
   independent executable that runs your Python code.
2. **Connection over gRPC**: The host and the worker establish a communication channel using gRPC, an inter-process
   communication (IPC) protocol. This is how the host tells the worker about events and how the worker sends results
   back.
3. **Worker Initialization**: The Python worker process starts up and loads the function_app.py file to discover all the
   decorated functions. This is where it parses the trigger and binding information you've defined with decorators (
   e.g., @app.route, @app.service_bus_queue_trigger).
4. **Loading Custom Extensions (if any)**: The worker loads any custom worker extensions you have registered, such as
   extensions with pre_invocation hooks. These allow for custom logic to be executed at various points during the
   function lifecycle.

## Runtime registration and readiness

1. **Function Registration**: The Python worker registers the discovered functions, along with their metadata (bindings,
   name, etc.), with the Functions host via gRPC.
   Host Acknowledgment: The host acknowledges the registration and is now aware of all the functions the worker can
   execute.
2. **Ready for Use**: At this point, the entire Function App, including the host and the isolated worker, is fully
   initialized and ready to start accepting and processing events. The host is listening for incoming triggers and will
   route them to the waiting worker process.

# Common Issues and Debugging Tips

## Issue: Function Not Found

This error typically occurs when the Functions host cannot find the function you are trying to invoke. This can happen
for several reasons.


`No job functions found. Try making your job classes and methods public. If you're using binding extensions (e.g. Azure Storage, ServiceBus, Timers, etc.) make sure you've called the registration method for the extension(s) in your startup code (e.g. builder.AddAzureStorage(), builder.AddServiceBus(), builder.AddTimers(), etc.).`

### Improper Azure Function Zip Package: Installation of dependencies in a way that they are not included in the deployment

package.

> [!TIP]
> [GitHub Issue #1262](https://github.com/Azure/azure-functions-python-worker/issues/1262#issuecomment-3009671538)

Here is a checklist:

- [ ] Build the function extensions:
    ```bash
    - bash: |
      if [ -f extensions.csproj ]
      then
          dotnet build extensions.csproj --runtime ubuntu.16.04-x64 --output ./bin
      fi
    workingDirectory: $(workingDirectory)
    displayName: 'Build extensions'
    ```
- [ ] Ensure that all dependencies are installed in the `./.python_packages/lib/site-packages` directory before
  packaging.

    ```bash
    uv pip install -r requirements.txt --target="./.python_packages/lib/site-packages"
    ```

See the sample `yaml` for Azure Pipeline:

- [azure-pipelines.yml](../samples/azure-pipelines.yml)

### External Library Exceptions: Errors in external libraries that prevent the function from being loaded correctly.

Whenever there's an exception in the global scope of your function file (e.g., syntax errors, import errors, or runtime
exceptions),
the function may fail to load, leading to a "Function Not Found" error.

See related issues:

- [GitHub Issue #1262](https://github.com/Azure/azure-functions-python-worker/issues/1262#issuecomment-2491933966)

Let's say you have the following code in your `function_app.py`:

```python
from external_library import some_function  # This will raise an ImportError if the library is not installed
import azure.functions as func

app = func.FunctionApp()


@app.function_name(name="HttpTrigger1")
@app.route(route="req")
def main(req: func.HttpRequest) -> str:
    user = req.params.get("user")
    return f"Hello, {user}!"
```

If `some_function` looks like this:

```python
# external_library
def some_function():
    return 1 / 0


some_function()
```

When the worker tries to load `function_app.py`, it will execute the import statement and the call to `some_function()`.
This will raise an `ImportError` if `external_library` is not installed, or a `ZeroDivisionError` when `some_function()`
is called. As a result, the function will fail to load, and you'll see the "Function Not Found" error.

> [!WARNING]  
> You will not be able to see the error in the Azure Portal logs, as the function fails to load before it can log anything. 
> Sometimes these errors are not visible in the local development environment either, especially if the environment is set up differently than the deployment environment.

1. **Access the SSH Web Interface**:
    *   In the Azure Portal, navigate to your function app page.
    *   Go to **Development Tools** -> **SSH** on the side menu.
2. **Navigate to the Project Directory**:
    
    `cd ~/site/wwwroot`
    
3. **Install Azure Functions Package**:
    
    `apt-get install azure-functions-core-tools-4`
    
4. **Start the Project Using the Local Environment Command**:
    
    `func host start`

From here, you should be able to see the error messages in the console output, which will help you identify and fix the
issues in your code.

