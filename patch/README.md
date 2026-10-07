# Local patches for third-party components

## rmw_cyclonedds_take_response_leak.patch

Fixes a native memory leak in `rmw_cyclonedds_cpp` (`rmw_take_response_request`,
present in jazzy 2.2.3 and still in rolling as of 2026-07).

DDS request/reply shares one reply topic per service, so every client receives
every other client's responses and the rmw layer filters them by GUID. The
filtering loop deserializes each queued reply into the same message and skips
foreign ones **without freeing the deserialized content** — the next iteration
overwrites the sequence/string pointers and orphans the payload. Any deployment
where several processes hold clients for the same service (e.g.
server_side_segmentation + publish_camera_images + tracking all calling
`get_frame` on the same cameras) leaks one full response payload per skipped
foreign reply: measured 13–30 MB/min per process at site 10.123.205.10.

The patch fini's and re-initializes the message before each skipped iteration.
Its typesupport lookup tries the C introspection typesupport first and resets the
rcutils error state when that lookup fails (RTDTK-392): before, the lookup failed
for every reply to an rclpy client and the error state stayed set, so the next
unrelated error on the thread printed rcutils' "This error state is being
overwritten" warning (e.g. in every python node with a `bitbots_tf_buffer`).

## rmw_cyclonedds_deserialize_error_reset.patch

`rmw_deserialize` (jazzy 2.2.3) tries the C introspection typesupport first and
does not reset the rcutils error state when that fails, which it does for every
C++ (rclcpp) message, e.g. each `bitbots_tf_buffer` lookup from python. The
patch resets it like `get_typesupport()` does (RTDTK-392). Rolling no longer has
this: its `rmw_deserialize` goes through `make_message_value_type()`, which resets.

## How the patches are applied (the proper path)

The patches are carried as a fork: `logivations/rmw_cyclonedds`, branch
`jazzy-fix-service-reply-leak` = upstream jazzy 2.2.3 (`54fbce9`) + these
patches. The fork is referenced in
`docker/dockerfile_dependencies/common/ros_nav_dependencies.repos`, so it is
built into `/code/ros2_ws` during the docker image build and its
`librmw_cyclonedds_cpp.so` shadows the stock `/opt/ros/jazzy` one through the
normal workspace overlay. The `.patch` files are the source of truth if the fork
branch ever needs to be recreated (apply them in the order listed here).

Remove the fork from the `.repos` file once both are fixed upstream in
ros2/rmw_cyclonedds (check `rmw_take_response_request` and `rmw_deserialize` in
rmw_cyclonedds_cpp/src/rmw_node.cpp).

## Site hotfix without an image rebuild

Applied on 10.123.205.10 on 2026-07-10; superseded by the image-build path at
the next release. Build inside the deep_cv container:

```bash
mkdir -p /code/rmw_patch_ws/src && cd /code/rmw_patch_ws/src
git clone -b jazzy https://github.com/ros2/rmw_cyclonedds.git
cd rmw_cyclonedds && git checkout 54fbce9   # 2.2.3 == version in /opt/ros/jazzy
git apply /code/deep_cv/patches/rmw_cyclonedds_take_response_leak.patch
git apply /code/deep_cv/patches/rmw_cyclonedds_deserialize_error_reset.patch
cd /code/rmw_patch_ws
export CMAKE_PREFIX_PATH=/opt/ros/jazzy/install
export AMENT_PREFIX_PATH=/opt/ros/jazzy/install
export PYTHONPATH=/opt/ros/jazzy/install/lib/python3.12/site-packages:$PYTHONPATH
colcon build --base-paths src --build-base /code/rmw_patch_ws/build \
  --install-base /code/rmw_patch_ws/install \
  --packages-select rmw_cyclonedds_cpp --cmake-args -DCMAKE_BUILD_TYPE=Release
```

(The explicit `--build-base/--install-base` matter: the container's
`/root/.colcon/defaults.yaml` otherwise writes into `/code/ros2_ws` and its
prefix scripts get regenerated — if `/opt/ros/jazzy` is not sourced at that
moment, the environment chain to it is lost for every later service start.)

Then prepend for each affected service in its monit conf:

```
LD_LIBRARY_PATH=/code/rmw_patch_ws/install/rmw_cyclonedds_cpp/lib:$LD_LIBRARY_PATH
```

and verify after restart with
`grep rmw_cyclonedds /proc/<pid>/maps` → must show `/code/rmw_patch_ws/...`.
