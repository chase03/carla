<h1>Configuring the simulation</h1>

Before you start running your own experiments there are few details to take into
account at the time of configuring your simulation. In this document we cover
the most important ones.


Changing the map
----------------

The map can be changed from the Python API with

```py
world = client.load_world('Town01')
```

this creates an empty world with default settings. The list of currently
available maps can be retrieved with

```py
print(client.get_available_maps())
```

To reload the world using the current active map, use

```py
world = client.reload_world()
```

Graphics Quality
----------------

<h4>Vulkan vs OpenGL</h4>

Vulkan _(if installed)_ is the default graphics API used by Unreal Engine and CARLA on Linux.  
It consumes more memory but performs faster.  
On the other hand, OpenGL is less memory consuming but performs slower than Vulkan.

!!! note
    Vulkan is an experimental build so it may have some bugs when running the simulator.

OpenGL API can be selected with the flag `-opengl`.

```sh
> ./CarlaUE4.sh -opengl
```

<h4>Quality levels</h4>

Currently, there are two levels of quality, `Low` and `Epic` _(default)_. The image below shows
how the simulator has to be started with the appropiate flag in order to set a quality level
and the difference between qualities.

![](img/epic_quality_capture.png)  |  ![](img/low_quality_capture.png)
:-------------------------:|:-------------------------:
`./CarlaUE4.sh -quality-level=Epic`  |  `./CarlaUE4.sh -quality-level=Low`

**Low mode runs significantly faster**, ideal for users that don't rely on quality precision.

Running off-screen
------------------

In Linux, you can force the simulator to run off-screen by setting the
environment variable `DISPLAY` to empty

!!! important
    **DISPLAY= only works with OpenGL**<br>
    Unreal Engine currently crashes when Vulkan is used when running
    off-screen. Therefore the `-opengl` flag must be added to force the engine to
    use OpenGL instead. We hope that this issue is addressed by Epic in the near
    future.

```sh
# Linux
DISPLAY= ./CarlaUE4.sh -opengl
```

This launches the simulator without simulator window, of course you can still
connect to it normally and run the example scripts. Note that with this method,
in multi-GPU environments, it's not possible to select the GPU that the
simulator will use for rendering. To do so, follow the instruction in
[Running without display and selecting GPUs](carla_headless.md).

No-rendering mode
-----------------

It is possible to completely disable rendering in the simulator by enabling
_no-rendering mode_ in the world settings. This way is possible to simulate
traffic and road behaviours at very high frequencies without the rendering
overhead. Note that in this mode, cameras and other GPU-based sensors return
empty data.

```py
settings = world.get_settings()
settings.no_rendering_mode = True
world.apply_settings(settings)
```

Command-line options
--------------------------

!!! important
    Some of the command-line options are not available in `Linux` due to the "Shipping" build.
    Therefore, the use of [`config.py`][configlink] script is needed to configure the simulation.

[configlink]: https://github.com/carla-simulator/carla/blob/master/PythonAPI/util/config.py

Some configuration examples:

```sh
> ./config.py --no-rendering      # Disable rendering
> ./config.py --map Town05        # Change map
> ./config.py --weather ClearNoon # Change weather
...
```

To check all the available configurations, run the following command:

```sh
> ./config.py --help
```

Commands directly available:

  * `-carla-rpc-port=N` Listen for client connections at port N, streaming port is set to N+1 by default.
  * `-carla-streaming-port=N` Specify the port for sensor data streaming, use 0 to get a random unused port.
  * `-quality-level={Low,Epic}` Change graphics quality level.
  * [Full list of UE4 command-line arguments][ue4clilink] (note that many of these won't work in the release version).

Example:

```sh
> ./CarlaUE4.sh -carla-rpc-port=3000
```

[ue4clilink]: https://docs.unrealengine.com/en-US/Programming/Basics/CommandLineArguments
