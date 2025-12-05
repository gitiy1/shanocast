FROM beardedtek/shanocast-builder:base AS builder

# Install findutils for batch file processing
RUN dnf -y install findutils

# Fix compiler warnings and create FFmpeg include symlinks
RUN cd openscreen && \
    # Fix the ignored-attributes warning by directly modifying the file
    sed -i '127s/.*/using FileUniquePtr = std::unique_ptr<FILE, int(*)(FILE*)>;/' cast/receiver/channel/static_credentials.cc && \
    # Fix uninitialized variable warnings in socket code
    sed -i '226s/int domain;/int domain = 0;/' platform/impl/stream_socket_posix.cc && \
    sed -i '106s/int domain;/int domain = 0;/' platform/impl/udp_socket_posix.cc && \
    # Create complete symlinks for FFmpeg headers
    mkdir -p /usr/local/include && \
    ln -sf /usr/include/ffmpeg/libavcodec /usr/local/include/ && \
    ln -sf /usr/include/ffmpeg/libavformat /usr/local/include/ && \
    ln -sf /usr/include/ffmpeg/libavutil /usr/local/include/ && \
    ln -sf /usr/include/ffmpeg/libswscale /usr/local/include/

# Build binary
RUN cd openscreen && \
    # Generate build files
    gn gen out/Default --args="is_debug=false use_custom_libcxx=false have_ffmpeg=true have_libsdl2=true cast_allow_developer_certificate=true is_clang=false" && \
    # -------------------------------------------------------------------------
    # NUCLEAR OPTION: Fix flags globally across ALL ninja files
    # -------------------------------------------------------------------------
    # 1. Brutally replace -Werror with -Wno-error in EVERY generated ninja file.
    #    This ensures third-party libs (zlib) also lose the -Werror flag.
    find out/Default -name "*.ninja" -exec sed -i 's/-Werror/-Wno-error/g' {} + && \
    # 2. Remove the Clang-specific flag that GCC fails on.
    find out/Default -name "*.ninja" -exec sed -i 's/-Wno-unknown-warning-option//g' {} + && \
    # 3. Inject stdint.h and legacy C support into the compiler command RULE directly.
    #    We target the 'command =' line in toolchain.ninja.
    #    We replace '${cflags}' with '${cflags} -include stdint.h ...' 
    #    This ensures our flags are always appended to the command, overriding any target-specific flags.
    sed -i 's/\${cflags}/& -include stdint.h -Wno-old-style-definition -Wno-deprecated-non-prototype/g' out/Default/toolchain.ninja && \
    #    Also handle the case where it uses $cflags (without braces) just in case
    sed -i 's/\$cflags/& -include stdint.h -Wno-old-style-definition -Wno-deprecated-non-prototype/g' out/Default/toolchain.ninja && \
    # -------------------------------------------------------------------------
    # Fix include paths for dependencies (ffmpeg)
    sed -i 's/-I..\/..\"/-I..\/..\"/g' out/Default/toolchain.ninja && \
    sed -i 's/-I..\/..\"/\"-I..\/..\" \"-I\/usr\/local\/include\"/' out/Default/toolchain.ninja && \
    # Build with ninja
    ninja -C out/Default cast_receiver && \
    cp out/Default/cast_receiver /build/shanocast

# Create a smaller runtime image
FROM fedora:42

# Install runtime dependencies
RUN dnf -y update && dnf -y install \
    ffmpeg-free \
    SDL2 \
    mesa-dri-drivers \
    mesa-libGL \
    libva \
    libvdpau \
    pulseaudio-libs \
    alsa-lib \
    avahi \
    avahi-tools \
    nss-mdns \
    iproute

# Create app directory
WORKDIR /app

# Copy binary from builder stage
COPY --from=builder /build/shanocast /app/

# Set the entrypoint
ENTRYPOINT ["/app/shanocast"]
