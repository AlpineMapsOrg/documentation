## Filter
This next section describes the label filtering process of the Qt application. The procedure is separated into two main parts: 
- a **GUI** to allow the user to choose what to filter
- a **Manager** that does the actual filtering
### GUI
For the GUI we are using qts [QML](https://doc.qt.io/qt-6/qtqml-index.html) (Qt Modeling Language) markup language. QML allows us to specify the structure of GUI elements (positions, colors, margins, ...). We are also able to connect the GUI elements with the c++ code in a most efficient way. By using a Modelbinding approach we can essentially link variables defined in c++ code and connect them bi-directionally with the GUI elements. This means that if the user changes a parameter on the GUI it is not only directly reflected in the value of the c++ variable, but it can also notify the c++ code that it changed and trigger further method calls from there. In the same way if something changes a variable in the code, this change is directly seen on the GUI without the need to call some additional update method.

In order to make the Modelbinding work a Q\_PROPERTY in the app/TerrainRendererItem.h class has to be set. This than allows us to connect to the set property in the QML code. 

In our specific example we created the nucleus::map\_label::FilterDefinitions struct. This struct is created in the TerrainRendererItem as a class variable and additionally set as a Q\_PROPERTY. Additionally the struct needs to have the Q\_GADGET macro applied and additional Q\_PROPERTY's specified for its own member variables. 

With those preparations done we can finally create the FilterWindow.qml file. This QML file essentially will hold the GUI elements that the user can set for himself:
![[filter.png | center | 200]]
The FilterWindow.qml is attached to the Main.qml file. It can be shown by either using the F7 key or by clicking on the bottom left most button and selecting the filter icon from the resulting submenu.

The filter page itself consists of CheckGroup elements. These elements are the checkboxes seen next to the feature type names. If this checkbox is not ticked it collapses all the elements within it. This was a great element to include here, since if (for example) a user doesn't want to show any peaks, he also would not want to see any filter options that are related to peaks like elevation or the checkboxes regarding crosses or registers. Once the checkbox is ticked again those options are naturally shown again. 

The other GUI element of note here is the RangeSlider which allows the user to define the specific range of elevations that should be shown.
### Manager
The manager that does the actual filtering of labels is the nucleus::map\_label::MapLabelFilter. This class is created by the Controller. The filter process can be started by two means. Either the FilterDefinition changed on the GUI through a change from the user, or a new tile has been loaded and this tile needs to undergo the filtering process. 

The entry point that detects GUI changes is the update\_filter() qt slot. It is connected to the change of the FilterDefinition signal that is sent from the GUI (in specific the app/TerrainRendererItem.h class). A change of the FilterDefinition requires that all the previously loaded tiles have to be reevaluated using this new definition. This means that the MapLabelFilter class needs to store all the features of each tile without any filtering. Furthermore it also possesses a Queue element that stores all the tiles that are due to be filtered. In the case of a definition change this queue is immediately filled with all the previously loaded tiles and the filter() method is called.

The main problem with this approach is that the user could slowly change the range slider and this method would be called hundreds of times within a small period of time. Since the filtering operation over a large amount of tiles takes quite a significant amount of time (at average it took about 100 milliseconds for a one time filter update of all tiles loaded after having done some movements in the application), we are limiting the filter method to be only executed once every 400 milliseconds. This way the user can still enjoy continuous updates when he slowly changes a ranged slider, while the application does not cause any stutters for trying to complete a queue of filter updates that are all already out of date. 

The second entry point is the update\_quads() qt slot. This slot connects directly to the Scheduler::gpu\_quads\_updated signal that the MapLabelManager originally received directly from the Scheduler. By connecting directly to this signal we can store all the tiles with all the features that are being loaded in this class and directly apply our filter method to them. 

The filter method itself (regardless if called by update\_filter or update\_quads) traverses over all the tiles in the queue that are to be processed and further iterates over all the individual features. Each feature is passed to the apply\_filter method which looks at the actual attributes of the FeatureTXT struct. Each Feature Type only applies filters that correspond to its specific type. If a feature passes all conditions it is automatically inserted into the m\_visible\_features object. 

After the filtering is done by either of those two specified entry points, the filter\_finished signal is emitted which propagates all the changed tiles to the MapLabelManager for further processing and rendering. Specifically the signal contains a reference to the m\_visible\_features object for all the tiles that need to be updated, and an additional list of removed tile ids (to indicate that those tiles should be deleted from GPU memory).

![[map_label_filter.svg|center]]