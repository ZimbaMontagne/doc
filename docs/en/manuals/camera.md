---
title: Camera component manual
brief: This manual describes the functionality of the Defold camera component.
---

# Cameras

A camera in Defold is a component that changes the viewport and projection of the game world. The camera component defines a bare bones perspective or orthographic camera that provides a view and projection matrix to the render script.

A perspective camera is typically used for 3D games where the view of the camera and the size and perspective of objects is based on a view frustum and the distance and view angle from the camera to the objects in the game.

For 2D games, it is often desirable to render the scene with an orthographic projection. This means that the view of the camera is no longer dictated by a view frustum, but by a box. Orthographic projection is unrealistic in that it does not alter the size of objects based on their distance. An object 1000 units away will render at the same size as an object right in front of the camera.

![projections](images/camera/projections.png)


## Creating a camera

To create a camera, <kbd>right click</kbd> a game object and select <kbd>Add Component ▸ Camera</kbd>. You can alternatively create a component file in your project hierarchy and add the component file to the game object.

![create camera component](images/camera/create.png)

The camera component has the following properties that defines the camera *frustum*:

![camera settings](images/camera/settings.png)

Id
: The id of the component

Aspect Ratio
: (**Perspective camera only**) - The ratio between the frustum width and height. 1.0 means that you assume a quadratic view. 1.33 is good for a 4:3 view like 1024x768. 1.78 is good for a 16:9 view. This setting is ignored if *Auto Aspect Ratio* is set.

Fov
: (**Perspective camera only**) - The *vertical* camera field of view expressed in _radians_. The wider the field of view, the more the camera will see. Note that the current default value (45) is misleading. For a 45 degree field of view, change the value to 0.785 (`π / 4`).

Near Z
: The Z-value of the near clipping plane.

Far Z
: The Z-value of the far clipping plane.

Auto Aspect Ratio
: (**Perspective camera only**) - Set this to let the camera automatically calculate the aspect ratio.

Orthographic Projection
: Set this to switch the camera to an orthographic projection (see below).

Orthographic Zoom
: (**Orthographic camera only**) - The zoom used for the orthographic projection (> 1 = zoom in, < 1 = zoom out).


## Using the camera

To activate a camera and have it feed its view and projection matrices to the render script, you send the component an `acquire_camera_focus` message:

```lua
msg.post("#camera", "acquire_camera_focus")
```

Each frame, the camera component that currently has camera focus will send a `set_view_projection` message to the "@render" socket, i.e. it will arrive to your render script:

```lua
-- builtins/render/default.render_script
--
function on_message(self, message_id, message)
    if message_id == hash("set_view_projection") then
        self.view = message.view                    -- [1]
        self.projection = message.projection
    end
end
```
1. The message posted from the camera component includes a view matrix and a projection matrix.

The camera component supplies the render script with either a perspective or orthographic projection matrix depending on the *Orthographic Projection* property of the camera. The projection matrix also takes into account the defined near and far clipping plane, the field of view and the aspect ratio settings of the camera.

The view matrix provided by the camera defines the position and orientation of the camera. A camera with an *Orthographic Projection* will center the view on the position of the game object it is attached to, while a camera with a *Perspective Projection* will have the lower left corner of the view positioned on the game object it is attached to.

::: important
For reasons of backwards compatibility the default render script ignores the projection provided by the camera and always uses an orthographic stretch projection. Learn more about the render script and the view and projection matrices in the [Render manual](/manuals/render/#default-view-projection).
:::

You can tell the render script to use the projection provided by the camera by sending a message to the render script:

```lua
msg.post("@render:", "use_camera_projection")
```


### Panning the camera

You pan/move the camera around the game world by moving the game object the camera component is attached to. The camera component will automatically send an updated view matrix based on the current x and y axis position of the camera.

### Zooming the camera

You can zoom in and out when using a perspective camera by moving the game object the camera is attached to along the z-axis. The camera component will automatically send an updated view matrix based on the current z-position of the camera.

You can zoom in and out when using an orthographic camera by changing the *Orthographic Zoom* property of the camera:

```lua
go.set("#camera", "orthographic_zoom", 2)
```

### Following a game object

You can have the camera follow a game object by setting the game object the camera component is attached to as a child of the game object to follow:

![follow game object](images/camera/follow.png)

An alternative way is to update the position of the game object the camera component is attached to every frame as the game object to follow moves.

### Converting mouse to world coordinates

When the camera has panned, zoomed or changed it's projection from the default orthographic Stretch projection the mouse coordinates provided in the `on_input()` lifecycle function will no longer match to the world coordinates of your game objects. You need to manually account for the change in view or projection. Converting from mouse/screen coordinates to world coordinates from the default render script is done like this:

::: sidenote
The [third-party camera solutions mentioned in this manual](/manuals/camera/#third-party-camera-solutions) provides functions for converting to and from screen coordinates.
:::

```Lua
-- builtins/render/default.render_script
--
local function screen_to_world(x, y, z)
	local inv = vmath.inv(self.projection * self.view)
	x = (2 * x / render.get_width()) - 1
	y = (2 * y / render.get_height()) - 1
	z = (2 * z) - 1
	local x1 = x * inv.m00 + y * inv.m01 + z * inv.m02 + inv.m03
	local y1 = x * inv.m10 + y * inv.m11 + z * inv.m12 + inv.m13
	local z1 = x * inv.m20 + y * inv.m21 + z * inv.m22 + inv.m23
	return x1, y1, z1
end
```


## Runtime manipulation
You can manipulate cameras in runtime through a number of different messages and properties (refer to the [API docs for usage](/ref/camera/)).

A camera has a number of different properties that can be manipulated using `go.get()` and `go.set()`:

`fov`
: The camera field-of-view (`number`).

`near_z`
: The camera near Z-value (`number`).

`far_z`
: The camera far Z-value (`number`).

`orthographic_zoom`
: The orthographic camera zoom (`number`).


## Third-party camera solutions

There is a library camera solution that implements common camera features such as game object follow, screen shake, screen to world coordinate conversion and so on. It is available from the Defold community assets portal:

- [Orthographic camera](https://defold.com/assets/orthographic/) (2D only) by Björn Ritzl.
