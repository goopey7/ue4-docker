{% if combine %}
FROM source as builder
{% else %}
ARG NAMESPACE
ARG TAG
ARG PREREQS_TAG
FROM ${NAMESPACE}/ue4-source:${TAG}-${PREREQS_TAG} AS builder
{% endif %}

# Remove the .git directory to disable UBT `git status` calls and speed up the build process
RUN rm -rf /home/ue4/UnrealEngine/.git

# Ensure UBT is built before we create the Installed Build, since Build.sh explicitly sets the
# target .NET Framework version, whereas InstalledEngineBuild.xml just uses the system default,
# which can result in errors when running the built UBT due to the wrong version being targeted
RUN if [ -f ./Engine/Build/BatchFiles/BuildUBT.sh ]; then \
		./Engine/Build/BatchFiles/BuildUBT.sh; \
	else \
		./Engine/Build/BatchFiles/Linux/Build.sh ShaderCompileWorker Linux Development -SkipBuild -buildubt; \
	fi

# Create an Installed Build of the Engine
WORKDIR /home/ue4/UnrealEngine
RUN ./Engine/Build/BatchFiles/RunUAT.sh BuildGraph \
	-target="Make Installed Build Linux" \
	-script=Engine/Build/InstalledEngineBuild.xml \
	-set:HostPlatformOnly=true \
	-set:WithClient=true \
	-set:WithDDC={% if excluded_components.ddc == true %}false{% else %}true{% endif %} \
	-set:WithServer=true \
	{{ buildgraph_args }} && \
	rm -R -f /home/ue4/UnrealEngine/LocalBuilds/InstalledDDC

# Ensure UnrealVersionSelector is built, since the prebuilt binaries may not be up-to-date
RUN ./Engine/Build/BatchFiles/Linux/Build.sh UnrealVersionSelector Linux Shipping

# Copy InstalledBuild.txt from the Installed Build and run UnrealVersionSelector to populate Install.ini with any custom Build ID specified in the BuildGraph flags
# (Note that the `-unattended` flag used below requires Unreal Engine 4.22 or newer, so this will break under older versions)
# (Note also that custom Build IDs are supported by Unreal Engine 5.3.1 and newer, and older versions will just use a GUID as the Build ID)
RUN cp /home/ue4/UnrealEngine/LocalBuilds/Engine/Linux/Engine/Build/InstalledBuild.txt /home/ue4/UnrealEngine/Engine/Build/InstalledBuild.txt && \
	./Engine/Binaries/Linux/UnrealVersionSelector-Linux-Shipping -register -unattended

{% if enable_ushell %}
# Ensure ushell is copied to the Installed Build
RUN rm -rf ./LocalBuilds/Engine/Linux/Engine/Extras/ushell && \
	cp -r ./Engine/Extras/ushell ./LocalBuilds/Engine/Linux/Engine/Extras/ushell && \
	bash -c 'set -e; shopt -s globstar; cd /home/ue4/UnrealEngine/LocalBuilds/Engine/Linux/Engine/Extras/ushell && chmod +x ./**/*.sh'
{% endif %}

# Split out both optional components (DDC, debug symbols, template projects) and large subdirectories so they can be copied
# into the final container image as separate filesystem layers, avoiding creating a single monolithic layer with everything
COPY split-components.py /tmp/split-components.py
RUN python3 /tmp/split-components.py /home/ue4/UnrealEngine/LocalBuilds/Engine/Linux /home/ue4/UnrealEngine/Components

# Copy the Installed Build into a clean image, discarding the source build
{% if combine %}
FROM prerequisites as minimal
{% else %}
ARG NAMESPACE
FROM ${NAMESPACE}/ue4-build-prerequisites:${PREREQS_TAG}
{% endif %}

# Copy the Installed Build files from the builder image
COPY --from=builder --chown=ue4:ue4 /home/ue4/UnrealEngine/LocalBuilds/Engine/Linux /home/ue4/UnrealEngine
COPY --from=builder --chown=ue4:ue4 /home/ue4/UnrealEngine/Components/Binaries /home/ue4/UnrealEngine
COPY --from=builder --chown=ue4:ue4 /home/ue4/UnrealEngine/Components/Content /home/ue4/UnrealEngine
COPY --from=builder --chown=ue4:ue4 /home/ue4/UnrealEngine/Components/Extras /home/ue4/UnrealEngine
COPY --from=builder --chown=ue4:ue4 /home/ue4/UnrealEngine/Components/Intermediate /home/ue4/UnrealEngine
COPY --from=builder --chown=ue4:ue4 /home/ue4/UnrealEngine/Components/Plugins /home/ue4/UnrealEngine
COPY --from=builder --chown=ue4:ue4 /home/ue4/UnrealEngine/Components/Source /home/ue4/UnrealEngine
{% if excluded_components.ddc == false %}
COPY --from=builder --chown=ue4:ue4 /home/ue4/UnrealEngine/Components/DDC /home/ue4/UnrealEngine
{% endif %}
{% if excluded_components.debug == false %}
COPY --from=builder --chown=ue4:ue4 /home/ue4/UnrealEngine/Components/DebugSymbols /home/ue4/UnrealEngine
{% endif %}
{% if excluded_components.templates == false %}
COPY --from=builder --chown=ue4:ue4 /home/ue4/UnrealEngine/Components/TemplatesAndSamples /home/ue4/UnrealEngine
{% endif %}

# Copy Install.ini from the builder image, so it can be used by tools that read the list of engine installations (e.g. ushell)
COPY --from=builder --chown=ue4:ue4 /home/ue4/.config/Epic/UnrealEngine/Install.ini /home/ue4/.config/Epic/UnrealEngine/Install.ini
WORKDIR /home/ue4/UnrealEngine

{% if not disable_labels %}
# Add labels to the built image to identify which components (if any) were excluded from the build that it contains
LABEL com.adamrehn.ue4-docker.excluded.ddc={% if excluded_components.ddc == true %}1{% else %}0{% endif %} 
LABEL com.adamrehn.ue4-docker.excluded.debug={% if excluded_components.debug == true %}1{% else %}0{% endif %} 
LABEL com.adamrehn.ue4-docker.excluded.templates={% if excluded_components.templates == true %}1{% else %}0{% endif %} 
{% endif %}

{% if enable_ushell %}
# Add ushell to the system PATH and alias `ushell` to `ushell.sh`
ENV PATH="$PATH:/home/ue4/UnrealEngine/Engine/Extras/ushell"
RUN echo 'alias ushell="ushell.sh"' >> /home/ue4/.bashrc

# Perform first-run setup for ushell
RUN bash -c 'set -e; source /home/ue4/UnrealEngine/Engine/Extras/ushell/ushell.sh && exit 0'
{% endif %}

# Perform first-run setup for Mono, UnrealBuildTool and AutomationTool, which makes it possible to build Unreal projects and plugins as users other than `ue4`
# (Note that this will only work with 4.26.0 and newer, older Engine versions will always require write access to `/home/ue4/UnrealEngine`)
# See the comments on this issue for details, including the need to ensure $HOME is set correctly: <https://github.com/adamrehn/ue4-docker/issues/141>
COPY print-editor-target.py /tmp/print-editor-target.py
RUN EDITOR_TARGET=$(python3 /tmp/print-editor-target.py /home/ue4/UnrealEngine) && \
	./Engine/Build/BatchFiles/Linux/Build.sh "$EDITOR_TARGET" Linux Development -SkipBuild && \
	mkdir -p ./Engine/Programs/AutomationTool/Saved && \
	chmod a+rw ./Engine/Programs/AutomationTool/Saved

# Enable Vulkan support for NVIDIA GPUs
USER root
RUN apt-get update && apt-get install -y --no-install-recommends libvulkan1 && \
	rm -rf /var/lib/apt/lists/* && \
	VULKAN_API_VERSION=`dpkg -s libvulkan1 | grep -oP 'Version: [0-9|\.]+' | grep -oP '[0-9|\.]+'` && \
	mkdir -p /etc/vulkan/icd.d/ && \
	echo \
	"{\
		\"file_format_version\" : \"1.0.0\",\
		\"ICD\": {\
			\"library_path\": \"libGLX_nvidia.so.0\",\
			\"api_version\" : \"${VULKAN_API_VERSION}\"\
		}\
	}" > /etc/vulkan/icd.d/nvidia_icd.json
USER ue4
