Janus WebRTC Server
===================
[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-brightgreen.svg)](COPYING)
![janus-ci](https://github.com/meetecho/janus-gateway/workflows/janus-ci/badge.svg)
[![Coverity Scan Build Status](https://scan.coverity.com/projects/13265/badge.svg)](https://scan.coverity.com/projects/meetecho-janus-gateway)
[![Fuzzing Status](https://oss-fuzz-build-logs.storage.googleapis.com/badges/janus-gateway.svg)](https://bugs.chromium.org/p/oss-fuzz/issues/list?sort=-opened&can=1&q=proj:janus-gateway)

Janus is an open source, general purpose, WebRTC server designed and developed by [Meetecho](https://www.meetecho.com).

## Using janus as a container running on docker

# Dockerfile.janus using a lighter base image
FROM debian:bullseye-slim

# Set non-interactive mode to avoid prompts
ENV DEBIAN_FRONTEND=noninteractive

# Install essential dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    apt-utils \
    build-essential \
    libmicrohttpd-dev \
    libjansson-dev \
    libsrtp2-dev \
    libsofia-sip-ua-dev \
    libglib2.0-dev \
    libopus-dev \
    libogg-dev \
    libcurl4-openssl-dev \
    liblua5.3-dev \
    libconfig-dev \
    libssl-dev \
    pkg-config \
    gengetopt \
    libtool \
    automake \
    git \
    wget \
    ffmpeg \
    tzdata \
    ca-certificates \
    libnice-dev \
    zlib1g-dev \
    libwebsockets-dev \
    libgstreamer1.0-dev \
    gstreamer1.0-plugins-base \
    gstreamer1.0-plugins-good \
    gstreamer1.0-plugins-bad && \
    rm -rf /var/lib/apt/lists/*

# Set timezone
RUN ln -snf /usr/share/zoneinfo/Europe/London /etc/localtime && \
    dpkg-reconfigure -f noninteractive tzdata

# Install libusrsctp
RUN git clone https://github.com/sctplab/usrsctp.git /opt/usrsctp && \
    cd /opt/usrsctp && \
    git checkout 0.9.5.0 && \
    ./bootstrap && \
    ./configure && \
    make && make install && \
    ldconfig && \
    cd .. && rm -rf /opt/usrsctp

# Clone and build Janus Gateway with minimal plugins
RUN git clone --depth 1 https://github.com/jdtheefirst/janus-gateway.git /opt/janus-gateway && \
    cd /opt/janus-gateway && \
    sh autogen.sh && \
    ./configure --enable-websockets --enable-data-channels --disable-rabbitmq --disable-mqtt --disable-docs && \
    make && \
    make install && \
    make configs

# Copy custom janus-plugin-rtmp if needed
RUN cp /opt/janus-gateway/src/plugins/libjanus_rtmp.so /usr/local/lib/janus/plugins/ && \
    cp /opt/janus-gateway/conf/janus.plugin.rtmp.cfg.sample /usr/local/etc/janus/

# Copy the SSL certificates from host to the container
COPY fullchain.pem /usr/local/etc/certificates/fullchain.pem
COPY privkey.pem /usr/local/etc/certificates/privkey.pem

# Expose necessary ports
EXPOSE 8088 8089 8188 8189

# Start Janus
CMD ["janus", "-F", "/usr/local/etc/janus"]


# Service
`  janus:
    build:
      context: .
      dockerfile: Dockerfile.janus
    ports:
      - "8088:8088" # HTTP REST API
      - "8188:8188" # WebSockets API (exposed only internally)
      - "10000-10200:10000-10200/udp" # Media ports for streaming
    environment:
      - JANUS_LOG_LEVEL=4
`

## Example Usage
`
useEffect(() => {
    const initialize = async () => {
      try {
        // Check permissions
        const cameraPermission = await navigator.permissions.query({
          name: "camera",
        });
        const micPermission = await navigator.permissions.query({
          name: "microphone",
        });

        if (
          cameraPermission.state !== "granted" ||
          micPermission.state !== "granted"
        ) {
          console.warn("Camera or microphone permissions are not granted.");
          // Optionally show a UI prompt here to notify the user
        } else {
          console.log("Permissions granted for camera and microphone.");
        }

        // Initialize Janus
        initializeJanus();
      } catch (error) {
        console.error(
          "Error initializing Janus or checking permissions:",
          error
        );
      }
    };

    initialize();

    // Cleanup logic on component unmount
    return () => {
      cleanUp();
    };
  }, []);

  const initializeJanus = () => {
    Janus.init({
      debug: "all",
      callback: () => {
        const janus = new Janus({
          server: "ws://janus:8188",
          success: () => {
            console.log("Janus Gateway initialized!");
            attachPlugin(janus);
          },
          error: (err) => {
            console.error("Error initializing Janus Gateway:", err);
          },
        });
        janusRef.current = janus;
      },
    });
  };

  const attachPlugin = (janus) => {
    janus.attach({
      plugin: "janus.plugin.rtmp",
      success: (plugin) => {
        console.log("Streaming plugin attached!", plugin);
        setRtmpPlugin(plugin);
        setConnected(true);
        attachStream(plugin);
      },
      error: (err) => {
        console.error("Error attaching plugin:", err);
      },
    });
  };

  const attachStream = async (plugin) => {
    try {
      const stream = await navigator.mediaDevices.getUserMedia({
        video: true,
        audio: true,
      });
      localVideoRef.current.srcObject = stream;
      localStreamRef.current = stream;

      plugin.createOffer({
        media: { video: true, audio: true },
        stream,
        success: (jsep) => {
          console.log("Generated JSEP:", jsep);
          plugin.send({
            message: { request: "configure", audio: true, video: true },
            jsep,
          });
        },
        error: (error) => {
          alert("An error occurred while creating the WebRTC offer.");
          console.error("Error creating WebRTC offer:", error);
        },
      });
    } catch (error) {
      console.error("Error accessing user media:", error);
    }
  };

  const startStreaming = () => {
    if (!rtmpPlugin) {
      console.error("RTMP plugin not attached.");
      return;
    }
    const rtmpUrl = "rtmp://<your streaming platform and key>";

    rtmpPlugin.send({
      message: { request: "publish", rtmp_url: rtmpUrl },
      success: () => {
        console.log("Publishing to RTMP successfully!");
        setStreaming(true);
      },
      error: (err) => {
        console.error("Error publishing to RTMP:", err);
      },
    });
  };
`

