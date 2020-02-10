<h1>Simulation time and synchrony</h1>

This section deals with two concepts that are fundamental to fully comprehend CARLA and gain control over it to achieve the desired results. There are different configurations that define how does time go by in the simulation and how does the server running said simulation work. The following sections will dive deep into these concepts:

  * [__Simulation time-step__](#simulation-time-step)  
	* Variable time-step
	* Fixed time-step  
	* Time-step limitations  
  * [__Client-server synchrony__](#client-server-synchrony)
	* Asynchronous mode  
	* Synchronous mode 
  * [__Synchrony and time-step__](#synchrony-and-time-step)

---------------
##Fixed time-step

The very first and essential concept to understand in this section is the difference between real time and simulation time. The simulated world has its own clock and time, conducted by the server. Between two steps of the simulation, there is the time spent to compute said steps (real time) and a time span that went by in those two moments of the simulation (simulated time). This latest is the time-step.   
Just an example for the sake of comprehension: When simulating, the server can take a few miliseconds to compute two steps of a simulation, but the time-step, the time that went by in the simulated world, can be configured to be, for instance, always a second.  
The time-step can be fixed or variable depending on user preferences, and CARLA can run in both modes. 

!!! Note
    After reading this section it would be a great idea to go for the following one, __Client-server synchrony__, especially the part about synchrony and time-step. Both are related concepts and affect each other when using CARLA. 

<h4>Variable time-step</h4>

This is the default mode in CARLA. When the time-step is variable, the simulation time that goes by between steps will be the time that the server takes to compute these. This is used for example in video games, where the main focus is realism and equal timing between the game and the player is a must.  
However, a variable time-step makes the simulation not repeatable. The reason for this is that the simulation is running with a time-step equal to the real one, but in the real world, the time between two moments is continuous, not exact, and the simulation represents time-steps using floats with decimal limitations. The time that is cropped for each step is an error that accumulates and prevents the simulatio from a precise repetition of what has happened. 

In order to set the simulation to a variable time-step the code will go as follows: 
```py
settings = world.get_settings()
settings.fixed_delta_seconds = None
world.apply_settings(settings)
```

<h4>Fixed time-step</h4>

Going for a fixed time-step makes possible to simulate longer periods in less time, and also gain repeatability by reducing the float-point arithmetic errors that a variable time-step introduces. Using the same time increment on each step is also the best way to gather data from the simulation, as physics and sensor data will correspond to an easy to comprehend moment of the simulation.   
To enable this mode set a fixed delta seconds in the world settings. For instance, to run the simulation at a fixed time-step of 0.05 seconds apply the following settings:

```py
settings = world.get_settings()
settings.fixed_delta_seconds = 0.05
world.apply_settings(settings)
```
Thus, the simulator will take twenty steps (1/0.05) to recreate one second of the simulated world. 

<h4>Time-step limitations</h4>

Physics must be computed within very low time gaps to be precise. The more time goes by, the more variables and chaos come to place and so, the more defective the simulation will be. 
CARLA uses 6 physical substeps to compute physics in every step, each with a maximum delta time of 0.016667s. To ease the comprehension of this, let's say that the time-step used gets divided in 6, one for each of this substeps, and the result of this division can never exceed the limit of the substep. If this happens, the physics will still work using their maximum delta time, but the time-step will be greater and thus, the delta-time and the physics will not be in synchrony.  

To sum up, __the time-step cannot be greater than 0.1s__, as 0.1/6=0.016667s, the maximum delta time per physical substep. If so, the physics will not be representative for the simulation. 

!!! Warning
    __Do not use a time-step greater than 0.1s.__<br>
    The reason for this is explained above this warning, but the original issue can be found here: Ref. [#695](https://github.com/carla-simulator/carla/issues/695)

----------------
##Client-server synchrony 

CARLA is built over a client-server architecture. This has been previously stated: the server runs the simulation and the client retrieves information and demands for changes in the world. But how do these two elements communicate?  
By default, CARLA runs in __asynchronous mode__, meaning that the server runs the simulation as fast as possible, without waiting for the client. On the contrary, running on __synchronous mode__ will make the server wait for a client tick, a "ready to go" message, before updating to the following simulation step.  

!!! Note
    In a multiclient architecture, only one client should make the tick. The server would react to receiving many as if these were all coming from one client and thus, take one step per tick.

Changing between synchronous and asynchronous mode is just a matter of a boolean state. In the following example, there is the code to make the simulation run on synchronous mode: 
```py
settings = world.get_settings()
settings.synchronous_mode = True
world.apply_settings(settings)
```

The synchronous mode becomes specially relevant when running with slow clients applications and when synchrony between different elements, such as sensors, is needed.  If the client is too slow and the server does not wait for it, the amount of information received will be impossible to manage and it can easily be mixed. On a similar tune, if there are ten sensors waiting to retrieve data and the server is sending all these information without waiting for all of them to have the previous one, it would be impossible to know if all the sensors are using data from the same moment in the simulation.  
As a little extension to the previous code, the following one would make for server waiting for a synchronous server waiting for a client tick when a camera sensor has received proper data. A more complex example regarding several sensors can be found [here][syncmodelink].
```py
settings = world.get_settings()
settings.synchronous_mode = True
world.apply_settings(settings)

camera = world.spawn_actor(blueprint, transform)
image_queue = queue.Queue()
camera.listen(image_queue.put)

while True:
    world.tick()
    image = image_queue.get()
```
[syncmodelink]: https://github.com/carla-simulator/carla/blob/master/PythonAPI/examples/synchronous_mode.py

!!! Important
    Data coming from GPU-based sensors (cameras) is usually generated with a delay of a couple of frames when compared with CPU based sensors, so synchrony is essential here. 


----------------
##Synchrony and time-step 

The configuration of both concepts explained in this page, simulation time-step and client-server synchrony leads for different types of simulation and results. Here is a brief summary on the possibilities and a better explanation for the reasoning behind it: 

|  | __Fixed time-step__ | __Variable time-step__ |
| --- | --- | --- |
| __Synchronous mode__ | Client is in total control over the simulation and its information. | Non realistic simulations. |
| __Asynchronous mode__ | Good time references for information. Server runs as fast as possible. | Video game paradigm. Non repeatable simulations. |

* __Synchronous mode + variable time-step:__ This is almost for sure a non-desirable state. Physics cannot run properly when the time-step is bigger than 0.1s and, if the server needs to wait for the client to compute the steps, this is prone to happen. Simulation time and physics then will not be in synchrony and thus, the simulation is not representative of anything realistic.  

* __Aynchronous mode + variable time-step:__ This is the default CARLA state and the configuration used in video games. Client and server are asynchronous but the simulation time and the real time run in sync. The simulation runs with a "realistic time", it is not repeatable due to the imprecise time-step and the information retrieved is loose regarding simulation time.  

* __Asynchronous mode + fixed time-step:__ The server will run as fast as possible, and yet, the information retrieved will be easily related with an exact moment in the simulation. This is the best mode to simulate great amounts of simulation time within much less real time. 

* __Synchronous mode + fixed time-step:__ In this mode, the client will have complete ruling over the simulation. The time step will be fixed and the server will not compute the following step until the client sends a tick saying so. This is the best mode when synchrony and precision is relevant, especially when dealing with slow clients or different elements retrieving information. 

!!! Warning 
    __In synchronous mode, always use a fixed time-step__. If the server has to wait for the user to compute the following step, and it is using a variable time-step, the simulation world will use time-steps too big for the physics to be realistic. This issue is better explained in the __time-step limitations__ section. 


----------------

So far so good. That is all there is to know about the roles of simulation time and the synchrony between client and server in CARLA. It is quiet a lot of information, but comprehending grants a much greater domain over CARLA. This is a great step, fundamental to achieve the full potential of CARLA.  
There are many things to play around now, after introducing these concepts. Open CARLA and mess around for a while to make sure that everything is clear and yet, if there are any doubts, feel free to post these in the CARLA forum. 

<div class="build-buttons">
<p>
<a href="https://forum.carla.org/" target="_blank" class="btn btn-neutral" title="Go to the CARLA forum">
CARLA forum</a>
</p>
</div>