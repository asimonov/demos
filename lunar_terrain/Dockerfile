# Copyright 2021 Open Source Robotics Foundation, Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
# A Docker configuration script to build the Space ROS image.
#
# The script provides the following build arguments:
#
#   VCS_REF     - The git revision of the Space ROS source code (no default value).
#   VERSION     - The version of Space ROS (default: "preview")

FROM osrf/space-ros:jazzy-2025.10.0

# Define arguments used in the metadata definition
ARG VCS_REF
ARG VERSION="preview"

# Specify the docker image metadata
LABEL org.label-schema.schema-version="1.0"
LABEL org.label-schema.name="Lunar Terrain"
LABEL org.label-schema.description="Lunar terrain demo on the Space ROS platform"
LABEL org.label-schema.vendor="Open Robotics"
LABEL org.label-schema.version=${VERSION}
LABEL org.label-schema.url="https://github.com/space-ros"
LABEL org.label-schema.vcs-url="https://github.com/space-ros/demos"
LABEL org.label-schema.vcs-ref=${VCS_REF}

# Clone all space-ros sources
RUN mkdir -p ${HOME}/spaceros_ws/src \
    && cd ${HOME}/spaceros_ws \
    && vcs import src < "${SPACEROS_DIR}/scripts/spaceros.repos" \
    && sudo chown -R spaceros-user:spaceros-user "${SPACEROS_DIR}"


# Define a few key variables
ENV DEMO_DIR=${HOME}/demos_ws
ENV GZ_VERSION=harmonic
ENV GZ_PARTITION=spaceros
ENV GZ_SIM_RESOURCE_PATH=$DEMO_DIR/src/simulation/models/lunar_terrain/models
ENV ROS_DISTRO=jazzy

# Disable prompting during package installation
ARG DEBIAN_FRONTEND=noninteractive

# Update the ROS package keys
ADD --chmod=644 https://raw.githubusercontent.com/ros/rosdistro/master/ros.key /usr/share/keyrings/ros-archive-keyring.gpg

# Make sure the latest versions of packages are installed
# Using Docker BuildKit cache mounts for /var/cache/apt and /var/lib/apt ensures that
# the cache won't make it into the built image but will be maintained between steps.
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
  sudo apt-get update \
  && sudo apt-get dist-upgrade -y

RUN sudo rosdep init \
    && rosdep update


# Install the various build and test tools
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
  sudo apt install -y \
    build-essential \
    clang-format \
    cmake \
    git \
    git-lfs \
    libbullet-dev \
    python3-colcon-common-extensions \
    python3-flake8 \
    python3-pip \
    python3-pytest-cov \
    python3-rosinstall-generator \
    python3-rosdep \
    python3-setuptools \
    python3-vcstool \
    wget \
    xterm # for spawning keyboard teleop in separate window

# install git-lfs for big files in simulation repo  
RUN  git lfs install

RUN sudo add-apt-repository ppa:kisak/kisak-mesa


# Install Gazebo Harmonic, GZ ROS packages and rviz2 from offical binaries

RUN sudo wget https://packages.osrfoundation.org/gazebo.gpg -O /usr/share/keyrings/pkgs-osrf-archive-keyring.gpg \
  && echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/pkgs-osrf-archive-keyring.gpg] http://packages.osrfoundation.org/gazebo/ubuntu-stable $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/gazebo-stable.list > /dev/null 

RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
  --mount=type=cache,target=/var/lib/apt,sharing=locked \
  sudo apt-get update -y \
  && sudo apt-get install -y gz-$GZ_VERSION \
  && sudo apt-get install -y ros-${ROS_DISTRO}-ros-gz \
  && sudo apt-get install -y ros-${ROS_DISTRO}-rviz2


# Generate repos file for demo dependencies, excluding packages from Space ROS core.
COPY --chown=spaceros-user:spaceros-user demo-pkgs.txt /tmp/
COPY --chown=spaceros-user:spaceros-user excluded-pkgs.txt /tmp/
RUN  /bin/bash -c 'source /opt/ros/${ROS_DISTRO}/setup.bash \
  && source ${SPACEROS_DIR}/setup.bash \
  && export ROS_PACKAGE_PATH=${AMENT_PREFIX_PATH} \
  && rosinstall_generator \
      --rosdistro ${ROS_DISTRO} \
      --deps \
      --exclude-path ${HOME}/spaceros_ws/src \
      --exclude RRP $(cat /tmp/excluded-pkgs.txt) \
      -- $(cat /tmp/demo-pkgs.txt) \
      > /tmp/demo_generated_pkgs.repos'


# create workspace for our demo
RUN mkdir -p ${DEMO_DIR}/src
WORKDIR ${DEMO_DIR}
RUN vcs import src < /tmp/demo_generated_pkgs.repos

# Install system dependencies
# this should have been done when building space-ros image...
#RUN /bin/bash -c 'source ${SPACEROS_DIR}/setup.bash' \
#  && rosdep install --from-paths ${SPACEROS_DIR}/src src \
#      --ignore-src \
#      --rosdistro ${ROS_DISTRO} \
#     -r -y \
#     --skip-keys "console_bridge generate_parameter_library fastcdr fastrtps rti-connext-dds-5.3.1 urdfdom_headers rmw_connextdds ros_testing rmw_connextdds rmw_fastrtps_cpp rmw_fastrtps_dynamic_cpp composition demo_nodes_py lifecycle rosidl_typesupport_fastrtps_cpp rosidl_typesupport_fastrtps_c ikos diagnostic_aggregator diagnostic_updater joy qt_gui rqt_gui rqt_gui_py"


# Get the source for the dependencies
# RUN vcs import src < /tmp/demo_generated_pkgs.repos
COPY --chown=spaceros-user:spaceros-user demo_manual_pkgs.repos /tmp/
RUN vcs import src < /tmp/demo_manual_pkgs.repos 


RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
  sudo apt-get update -y \
  && /bin/bash -c 'source /opt/ros/${ROS_DISTRO}/setup.bash \
  && source "${SPACEROS_DIR}/setup.bash" \
  && rosdep install \
     --simulate \
     --from-paths src \
     --ignore-src \
     -r -y \
     --rosdistro ${ROS_DISTRO} > /tmp/rosdep-commands.sh \
  && chmod +x /tmp/rosdep-commands.sh \
  && /tmp/rosdep-commands.sh'

# Copy the demo source code
COPY --chown=spaceros-user:spaceros-user lunar_sun_gz_plugin src/lunar_sun_gz_plugin
COPY --chown=spaceros-user:spaceros-user lunar_terrain_gz_bringup src/lunar_terrain_gz_bringup
COPY --chown=spaceros-user:spaceros-user lunar_terrain_gz_worlds src/lunar_terrain_gz_worlds

# Build the demo
RUN /bin/bash -c 'source /opt/ros/${ROS_DISTRO}/setup.bash \
  && source ${SPACEROS_DIR}/setup.bash \
  && colcon build --cmake-args -DCMAKE_BUILD_TYPE=Release'

# Create the render group if it doesn't exist
RUN sudo groupadd -f render

# Add the user to the render group so that the user can access /dev/dri/renderD128
RUN sudo usermod -aG render spaceros-user


# Setup the entrypoint
COPY --chown=spaceros-user:spaceros-user ./entrypoint.sh /
# RUN chown appuser:appuser /entrypoint.sh && chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
CMD ["bash"]

