## Label Rendering
### gl\_engine::MapLabelManager
The MapLabelManager is part of the gl\_engine namespace and is created by the Window class. The init() method initializes the manager. This method mainly creates the various OpenGL buffers it needs to function. One of these buffers is the icon texture buffer, another is the font texture buffer. The creation of both of those textures is handled by the *nucleus::maplabel::LabelFactory*. Lastly it also creates the index buffer.  Since every single character is rendered onto a quad we are using instance drawing and our index buffer contains only indices for one quad.

The update\_labels() method of the MapLabelManager is another important access point to this class. This method is triggered by the *MapLabelFilter::filter\_finished()* signal. This signal is propagated from the nucleus::Controller through nucleus::AbstractRenderWindow to the gl\_engine::Window to the MapLabelManager. It contains the tiles and features that are currently visible and should be rendered and the tile ids that are no longer needed and should be removed from the GPU memory. The method calls remove\_tile() and upload\_to\_gpu() methods respectively. The update\_labels() method furthermore calls the *LabelFactory renew\_font\_atlas()* method which updates the font texture if necessary and rebinds the texture if needed.

The remove\_tile() method is straight forward.  It searches the m\_gpu\_tiles map for the tile ids and removes them. Additionally it also explicitly destroys the created Vertex Array Objects (VAOs).

The upload\_to\_gpu() method first creates a GPUVectorTile struct where tile id, vertex buffer, VAO and instance count are stored. The VAO is created and bound to store the buffers needed for rendering. The previously created index buffer is bound, and the vertex buffer is created and filled with the data from the features. This data is created by the *nucleus/map\_label/LabelFactory* create\_labels() method. This method creates an object which can be directly allocated to the vertexbuffer. After the allocation of the vertexbuffer the data offsets are defined and bound to locations indices accessible by the shader. After everything has been initialized the VAO is released and the GPUVectorTile is stored in a map that can be easily processed in the draw method.

The draw method binds all the relevant uniforms (like font, icon and depth textures, inverse view rotation matrix and other settings for enabling and disabling label distance scaling). Furthermore it also enables the GL Blending option. The method then iterates over every GPUVectorTile that is stored and renders them using glDrawElementsInstanced with the stored instance count. Labels are drawn twice. Once for the outline of the label and the second draw call for the label itself. The draw method uses a shader program that uses the labels.vert vertex shader and the labels.frag fragment shader.

The draw\_picker method works in a similar fashion to the draw method. The main difference is that no font/icon textures are bound, The blending option is not being enabled and labels are only drawn once. Furthermore the shader program being used connects the previously used labels.vert to a new labels\_picker.frag fragment shader.

Both the draw method and the draw\_picker method is called by the *Window paint()* method. The draw\_picker method is drawn into a picker FrameBuffer, while the draw method draws into the decoration FrameBuffer. The decoration FrameBuffer is later copied into the FrameBuffer that is actually passed to the display, while the picker FrameBuffer is only used to read specific pixels on the CPU once a click event happens.

![[glengine.svg|center]]
### nucleus::map\_label::LabelFactory
The LabelFactory class manages all the parts that are necessary to create labels for a rendering engine without being tied to a specific rendering approach (like OpenGL). The main parts are creation of the icon texture, creation of VertexData structs from FeatureTXT objects and calling the FontRenderer to create the font texture.

At the initialization stage, that is triggered by the MapLabelManager, the FontRenderer is also initialized, and the first font texture is created including a font\_data object which holds valuable information about the individual characters that can be rendered (including UV coordinates on the font texture, kerning and sizing information). 

Simultaneously the label\_icons() method is also called at the initialization. This method loads and merges all the individual icons of the feature types into one texture element. 

The renew\_font\_atlas() method is triggered at every update\_labels() call from the MapLabelManager. It checks (by using the [[Label Rendering#nucleus::map_label::Charset|Charset]] class) whether or not new chars have to be rendered by the FontRenderer and generates new textures and font\_data if necessary. 

The create\_label() method is used to create the VertexData objects that are later used directly by the Shader. Each individual character in each visible feature has exactly one VertexData object. This object is created by first converting the text to UTF-16 characters, where a character is encoded using 16 bits. Initially a create\_text\_meta() function is called this iterates over each character in a label and calculates the leading/kerning values that are needed for all characters in combination with their neighbors. Additionally it also calculates the total width of the label and uses this information to center it. 
The create\_label method continues by iterating over all the characters again. In this iteration the final vertex positions are calculated. Each character creates a VertexData struct where the data necessary for the shader is stored. 
This VertexData contains:
 
-  world position of the label (identical for each character in one label)
- character position relative to the world position (vec4 where first two elements store the upper left position and last to elements store the offset to this position)
- UV positions (vec4 where first two elements store the upper left UV position and last to elements store the offset)
- label importance (float between 0-1)
- picker color (vec4 that encodes a unique value which can be identified by the [[Picker#PickerManager|PickerManager]])
- texture index (identifies which font texture in texture array should be used)

The create\_label method furthermore creates one additional VertexData object to store the icon for each label. All the created VertexData are collected in a vector which is later returned to the MapLabelManager for upload to the GPU.
### STB\_Truetype
[STB](https://github.com/nothings/stb) are single file c++ libraries in the public domain. Each header file can be used separately to accomplish a task. In our application we are using the [stb_truetype.h](https://github.com/nothings/stb/blob/master/stb_truetype.h) file in order to render .ttf font files into a font atlas. A font atlas here is a texture image with each (previously selected) character rendered into certain positions. Those positions are then stored by us and can later be used to render a single character onto a quad. This kind of font rendering algorithm is a similar approach as explained in the book "Learn OpenGL" by De Vries \[[[References and Links#Vries 2020|Vries 2020]]\].
### nucleus::map\_label::FontRenderer
The FontRenderer uses the aforementioned [stb_truetype](https://github.com/nothings/stb/blob/master/stb_truetype.h) to create the font textures and corresponding metadata for each created character. The font metadata holds UV regions, character sizes and other values needed like leading/kerning and ascender/descender (e.g. k and g) placement. 

The init() method loads the True Type Font file (.ttf extension) and initializes some values that it needs to accurately render characters. Additionally it also generates an empty 2 channel Raster object which is used to hold the texture. We are using 2 channels for the font texture because this allows us to store the character on one channel and the outline of the character in the second channel. This separation is needed to render and color them separately.

The render() method is used to render the actual characters to the font texture and later create the outlines around the characters. The actual font rendering to the texture is done mostly by the functions with the stbtt\_ prefixes. Since the STB library only supports rendering into a one channel texture we first generate a temporary texture and additionally collect some meta information that the subsequent stbtt\_ methods need (like a scaling factor for the font to render a specific font size). We iterate over each individual character we want to render, calculate the dimensions it needs on the texture and find a fitting place on the texture for it. The stbtt\_MakeGlyphBitmap method does the actual rendering on the provided coordinates. Finally we also need to store those coordinates in the font\_data so that it can be later used for rendering in the LabelFactory. At the end of the render\_text section the temporary texture is merged with the actual 2 channel font texture.
If the current texture does not have any place for further characters a new texture is being generated. Currently every character at the current font size should easily fit onto one texture. But for scalability reasons we decided that implementing this now was necessary. 

After all the new characters have been rendered on the textures the outline is generated. For each new character all the pixels within the vicinity of the character are being traversed, and a filter kernel is applied.

### nucleus::map\_label::Charset
The Charset class is a helper class that holds all the encountered characters of the VectorTileManager. The class is accessible using a singleton and is currently only used by the VectorTileManager and the LabelFactory. At the initialization of the class a charset.txt file is loaded where the most common characters are already set. Those common characters are also initially copied to the VectorTileManager and the LabelFactory. This way both classes can periodically call the is\_update\_necessary() method and pass their current set size as an argument to this method to compare if an action has to be taken to synchronize.

If the VectorTileManager recognizes that it has gathered more characters than the amount of characters present in the Charset class, it immediately calls the add\_chars() method. This method increases the characters stored in the Charset class and automatically triggers the update for the LabelFactory the next time it calls the is\_update\_necessary() method.

Once the LabelFactory finds this discrepancy it calls the char\_diff() method of the Charset class. This method compares the set of the LabelFactory with the current set of the Charset class and returns the difference, i.e. the new characters that have to be rendered to the font texture. This in result triggers this update of the font texture.

In conclusion, the Charset class is the central point that makes it possible that the application doesn't have to worry, that through some update on the Vector Tile Server some new feature contain special characters that weren't previously present in the application, and is able to render them without any problems.

![[map_label_rendering.svg|center]]
### Shader
The shaders for the labels can be found in the *gl\_engine/shaders* directory and are called labels.vert, labels.frag and labels\_picker.frag.

The labels.vert shader is used both by labels.frag and labels\_picker.frag as the vertex shader in their respective shader programs. This way we make sure that the picker renders to the exact same positions as the labels that are visible on the screen.
#### labels.vert
First the vertex shader calculates the distance between the camera and the label. This distance can later be used for a soft distance scaling, where labels that are further away are rendered a little bit smaller. To enable this feature the shader uniform label\_dist\_scaling has to be set to true. The smallest size a label is shown by using distance scaling is 36% of the original size.

Another scaling factor that is applied is the importance scaling. Depending on the importance of the feature the label is scaled down a bit. The current scaling factor for importance scaling, scales values between 71% and 100% of the original scale. 

An additional usage of the calculated distance between feature and camera is used in the label\_visible() function. This function hides labels that are too far away from the camera depending on the importance (e.g. labels with low importance are only shown if the camera is within 3 km while labels with a very high visibility are shown if the camera is within 500 km with additional steps specified in between). Please note that those values are still prone to be changed due to fine-tuning and/or will be made dependent on feature type.
Furthermore the label\_visible() function also hides labels if the feature is too far below the terrain (e.g. hidden by a mountain).

Finally if this function succeeds the actual label positions and UV positions are calculated and stored for the subsequent shaders in the program.

For the positioning of the labels in the vertex shader we are first creating a relative\_to\_cam vector where the world position of the label is subtracted from the world position of the camera. This value is then multiplied by the view projection matrix of the camera in order to calculate the correct label positioning. Since we are using perfect quads for the rendering of the individual characters, but every character has slightly different sizes, we are using the z and w components of the VertexData.positions as offsets. Those offsets are multiplied with an offset mask array with the vertex id as an index to determine what kind of offset we need to apply to the position in order to visualize all four corners of the quad. 

In order to better visualize the last sentence here is an example:
```
position = 5,5
offset = 10,10
top left corner should be 5,5
bottom right corner should be 15,15
and so on...

In order to visualize the top right corner (15,5) we would need to use an offset_mask of 1,0 (-> apply the offset for the x position)

how it looks in the shader:
pos = vec4(5,5, 10,10)
offset_mask[1] = vec2(1,0) // each corner has a different mask
vertexID = 1 // is determined dynamically using gl_VertexID

pos.xy + pos.zw * offset_mask[vertexID] // results in 15,5
```
Doing it this way allows us to save on the amount of data that we would need to send to the GPU (one vec4 instead of four vec2). The same principle is also used with the UV coordinates.

#### labels.frag
In the fragment shader the outline and the actual character font is rendered individually using a drawing\_outline uniform as the deciding factor. The outline uses the green channel of the font texture and renders the outline using the outlineColor, while the font uses the red channel and fontColor (currently the used colors are hard-coded in the shader).

The label icons are using the same shader as the individual characters. In order to decide when the font atlas and when the icon sampler should be used we are using texture coordinates in the 10-11 range for the icons (instead of the customary 0-1 range). We therefore have to subtract 10 from the texture coordinates. 

Additionally it is worth noting that the gl\_FragDepth for the actual font is drawn slightly closer to the camera (depth multiplied by 0.99999). This was done to prevent any possible z-fighting to appear.
#### labels\_picker.frag
The labels picker fragment shader is a very simple shader. It simply returns the picker color to the out\_Color variable. In the case for the picker the whole quad is filled with this color instead of only the character itself. This was done to prevent users to accidentally click in the hole of, for example O characters, and the application not recognizing that the user wants to actually click on this label. 
