## Overview - Alpine Maps
The AlpineMaps application is divided into the following two main namespaces *nucleus* and *gl\_engine*. gl\_engine contains code that is specific to OpenGL. That means that only code that needs OpenGL should be stored inside this namespace. All other code, that might be usable with other rendering means, should be stored in the nucleus namespace. 

The only class added for label rendering in the gl\_engine namespace was the MapLabelManager. This class manages the creation of the appropriate Vertex Buffer Objects (VBO) and Vertex Array Objects (VAO) and creates the GPU draw calls that are sent to the GPU for processing.

In the nucleus namespace the main classes added by this project are located in the *map\_label*, *vector\_tiles* and *picker* folders. 

Some preexisting classes of note that connect everything together are nucleus/tile\_scheduler/Scheduler, nucleus/Controller and gl\_engine/Window.
### nucleus
In the nucleus namespace, the **Controller** is the central part of the program. It connects multiple classes together using Qts signal/slot pattern. In this class many parts like Scheduler, Camera, LayerAssembler and TileLoadService are initialized and connected together.

While the Controller is the class that connects everything, the **Scheduler** might be interpreted as the beating heart of the application. Once the camera moves (or for the initial camera position), the Scheduler creates the *quads\_requested* signal with the appropriate tile ids. This request is further processed by some methods/classes that optimize the requests (e.g. limiting the amount of requests that will be sent out at once). Finally the LayerAssembler creates the tile\_requested signal which causes all initialized TileLoadService objects to make the actual network calls to the specified tile servers. 

 The **TileLoadService** is a class that makes network calls to specified URLs with certain tile coordinates, and it returns the received byte data using the load\_finished signal. The tile coordinates can follow arbitrary tile patterns (e.g. Z/X/Y). 

The **LayerAssembler** waits for all the tile types (height, ortho graphics and label vector information) to be loaded for a given tile id. If all data has been successfully downloaded it is combined into a *LayeredTile* and later a *TileQuad* and send back to the Scheduler where it is stored in the ram cache. 

The **Scheduler** periodically calls its update\_gpu\_quads() method. In this method it checks for new tile data that is stored in the cache and finally processes the byte data into appropriate formats. In the case for the label vector tile data it calls the *nucleus/vector\_tiles/VectorTileManager* *to\_vector\_tile* function which returns VectorTile objects. This VectorTile object is defined in *nucleus/vector\_tiles/VectorTileFeature.h* and is essentially just a set of *FeatureTXT* (Struct that contains: id, name, position, etc.).

Note the appropriate structs for LayeredTile, TileQuad, GpuLayeredTile and GpuLayeredQuad can be found in *nucleus/tile\_scheduler/tile\_types*. Additionally most of the above description is abbreviated to give only an overview of how everything works together.
### gl\_engine
The main class that connects everything in the *gl\_engine* namespace is the *gl\_engine/window*. This class derives from the *nucleus/AbstractRenderWindow* class to allow for a possible future change to other render engines. 

The *initialise\_gpu()* method initializes the components of the Window class (including the MapLabelManager which we implemented for this project). The paint method is called at every frame and as its name implies calls the draw commands of each individual component. 

While the Window class possesses a couple of access points from the nucleus Controller and Scheduler, the most notable method is *update\_gpu\_quads()*. In this method the Scheduler provides the Window with all vector tile quads that are newly available or have to be dismissed due to not being needed anymore. 


