# syntax=docker/dockerfile:experimental

ARG BASEIMAGE=acsldocker/px4
ARG BASETAG=gazebo-0.0.1

ARG ARG_MERGE_STAGE_VNC_BASE=stage_novnc
ARG ARG_MERGE_STAGE_BROWSER_BASE=stage_chromium
ARG ARG_FINAL_STAGE_BASE=merge_stage_vnc


###############
### stage_cache
###############

FROM ${BASEIMAGE}:${BASETAG} as stage_cache

RUN rm -f /etc/apt/apt.conf.d/docker-clean; echo 'Binary::apt::APT::Keep-Downloaded-Packages "true";' > /etc/apt/apt.conf.d/keep-cache
RUN apt-get update


####################
### stage_essentials
####################

FROM ${BASEIMAGE}:${BASETAG} as stage_essentials

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

RUN \
    --mount=type=cache,target=/var/cache/apt,from=stage_cache,source=/var/cache/apt \
    --mount=type=cache,target=/var/lib/apt,from=stage_cache,source=/var/lib/apt \
    DEBIAN_FRONTEND=noninteractive apt-get install -y \
        jq \
        nano \
        psmisc \
        sudo \
        tini \
        wget


#################
### stage_xserver
#################

FROM stage_essentials as stage_xserver
ARG ARG_APT_NO_RECOMMENDS

ENV \
    FEATURES_BUILD_SLIM_XSERVER=${ARG_APT_NO_RECOMMENDS:+1} \
    NO_AT_BRIDGE=1

RUN \
    --mount=type=cache,target=/var/cache/apt,from=stage_cache,source=/var/cache/apt \
    --mount=type=cache,target=/var/lib/apt,from=stage_cache,source=/var/lib/apt \
    DEBIAN_FRONTEND=noninteractive apt-get install -y ${ARG_APT_NO_RECOMMENDS:+--no-install-recommends} \
        dbus-x11 \
        xauth \
        xinit \
        x11-xserver-utils \
        xdg-utils


##############
### stage_xfce
##############

FROM stage_xserver as stage_xfce
ARG ARG_APT_NO_RECOMMENDS

ENV FEATURES_BUILD_SLIM_XFCE=${ARG_APT_NO_RECOMMENDS:+1}

RUN \
    --mount=type=cache,target=/var/cache/apt,from=stage_cache,source=/var/cache/apt \
    --mount=type=cache,target=/var/lib/apt,from=stage_cache,source=/var/lib/apt \
    DEBIAN_FRONTEND=noninteractive apt-get install -y ${ARG_APT_NO_RECOMMENDS:+--no-install-recommends} \
        xfce4 \
        xfce4-terminal


###############
### stage_tools
###############

FROM stage_xfce as stage_tools
ARG ARG_APT_NO_RECOMMENDS
ARG ARG_FEATURES_SCREENSHOOTING
ARG ARG_FEATURES_THUMBNAILING

ENV \
    FEATURES_BUILD_SLIM_TOOLS=${ARG_APT_NO_RECOMMENDS:+1} \
    FEATURES_SCREENSHOOTING=${ARG_FEATURES_SCREENSHOOTING:+1} \
    FEATURES_THUMBNAILING=${ARG_FEATURES_THUMBNAILING:+1}

RUN \
    --mount=type=cache,target=/var/cache/apt,from=stage_cache,source=/var/cache/apt \
    --mount=type=cache,target=/var/lib/apt,from=stage_cache,source=/var/lib/apt \
    DEBIAN_FRONTEND=noninteractive apt-get install -y ${ARG_APT_NO_RECOMMENDS:+--no-install-recommends} \
        curl \
        mousepad \
        python3 \
        systemctl \
        fonts-nanum \
        fcitx-hangul \
        firefox \
        gnome-terminal \
        ${ARG_FEATURES_SCREENSHOOTING:+ristretto xfce4-screenshooter} \
        ${ARG_FEATURES_THUMBNAILING:+tumbler} \
    && curl -fsSL https://get.docker.com -o get-docker.sh \
    && sh get-docker.sh


#############
### stage_vnc
#############

FROM stage_tools as stage_vnc
ARG ARG_VNC_COL_DEPTH
ARG ARG_VNC_DISPLAY
ARG ARG_VNC_PORT
ARG ARG_VNC_PW
ARG ARG_VNC_RESOLUTION
ARG ARG_VNC_VIEW_ONLY

# wget -qO- https://github.com/accetto/tigervnc/releases/download/v1.12.0-mirror/tigervnc-1.12.0.x86_64.tar.gz | tar xz --strip 1 -C / \
RUN \
    wget -qO- https://sourceforge.net/projects/tigervnc/files/stable/1.12.0/tigervnc-1.12.0.x86_64.tar.gz | tar xz --strip 1 -C / \
    && ln -s /usr/libexec/vncserver /usr/bin/vncserver \
    && sed -i 's/exec(@cmd);/print "@cmd";\nexec(@cmd);/g' /usr/libexec/vncserver

ENV \
    DISPLAY=${ARG_VNC_DISPLAY:-:1} \
    FEATURES_VNC=1 \
    VNC_COL_DEPTH=${ARG_VNC_COL_DEPTH:-24} \
    VNC_PORT=${ARG_VNC_PORT:-5901} \
    VNC_PW=${ARG_VNC_PW:-headless} \
    VNC_RESOLUTION=${ARG_VNC_RESOLUTION:-1920x1080} \
    VNC_VIEW_ONLY=${ARG_VNC_VIEW_ONLY:-false}

EXPOSE ${VNC_PORT}


###############
### stage_novnc
###############

FROM stage_vnc as stage_novnc
ARG ARG_APT_NO_RECOMMENDS
ARG ARG_NO_VNC_PORT

ENV \
    FEATURES_BUILD_SLIM_NOVNC=${ARG_APT_NO_RECOMMENDS:+1} \
    FEATURES_NOVNC=1 \
    NO_VNC_HOME=/usr/libexec/noVNCdim \
    NO_VNC_PORT=${ARG_NO_VNC_PORT:-6901}

RUN \
    mkdir -p "${NO_VNC_HOME}"/utils/websockify \
    && wget -qO- https://github.com/novnc/noVNC/archive/v1.3.0.tar.gz | tar xz --strip 1 -C "${NO_VNC_HOME}" \
    && wget -qO- https://github.com/novnc/websockify/archive/v0.10.0.tar.gz | tar xz --strip 1 -C "${NO_VNC_HOME}"/utils/websockify \
    && chmod +x -v "${NO_VNC_HOME}"/utils/novnc_proxy

### add 'index.html' for choosing noVNC client
RUN echo -e \
"<!DOCTYPE html>\n" \
"<html>\n" \
"    <head>\n" \
"        <title>noVNC</title>\n" \
"        <meta charset=\"utf-8\"/>\n" \
"    </head>\n" \
"    <body>\n" \
"        <p><a href=\"vnc_lite.html\">noVNC Lite Client</a></p>\n" \
"        <p><a href=\"vnc.html\">noVNC Full Client</a></p>\n" \
"    </body>\n" \
"</html>" \
> "${NO_VNC_HOME}"/index.html

EXPOSE ${NO_VNC_PORT}


###################
### merge_stage_vnc
###################

FROM ${ARG_MERGE_STAGE_VNC_BASE} as merge_stage_vnc
ARG ARG_HEADLESS_USER_NAME
ARG ARG_HOME

ENV HOME=${ARG_HOME:-/home/${ARG_HEADLESS_USER_NAME:-headless}}

WORKDIR ${HOME}


###############
### FINAL STAGE
###############

FROM ${ARG_FINAL_STAGE_BASE} as stage_final
ARG ARG_FEATURES_USER_GROUP_OVERRIDE
ARG ARG_HEADLESS_USER_NAME
ARG ARG_SUDO_PW

ENV \
    FEATURES_USER_GROUP_OVERRIDE=${ARG_FEATURES_USER_GROUP_OVERRIDE:+1} \
    FEATURES_VERSION_STICKER=1 \
    STARTUPDIR=/dockerstartup \
    LANG=C.UTF-8 \
    LC_ALL=C.UTF-8 \
    ROS_DISTRO=foxy \
    QT_X11_NO_MITSHM=1 \
    NVIDIA_VISIBLE_DEVICES=all \
    NVIDIA_DRIVER_CAPABILITIES=graphics,utility,compute,display

COPY ./novnc/src/xfce-startup "${STARTUPDIR}"/

COPY ./novnc/xfce/src/home/config "${HOME}"/.config/
COPY ./novnc/xfce/src/home/Desktop "${HOME}"/Desktop/
COPY ./novnc/xfce/src/home/readme*.md "${HOME}"/

### Create the default application user (non-root, but member of the group zero)
### and allow the group zero to modify '/etc/passwd' and '/etc/group'.
### Providing the build argument ARG_SUPPORT_USER_GROUP_OVERRIDE (set to anything) allows any user
### to modify both files and makes user group overriding possible (like 'run --user x:y').
RUN \
    chmod 664 /etc/passwd /etc/group \
    && echo "${ARG_HEADLESS_USER_NAME:-headless}:x:1001:0:Default:${HOME}:/bin/bash" >> /etc/passwd \
    #&& adduser "${ARG_HEADLESS_USER_NAME:-headless}" sudo \
    && usermod -aG sudo,root user \
    && echo "${ARG_HEADLESS_USER_NAME:-headless}:${ARG_SUDO_PW:-${VNC_PW}}" | chpasswd \
    && echo "root:${ARG_SUDO_PW:-${VNC_PW}}" | chpasswd \
    && ${ARG_FEATURES_USER_GROUP_OVERRIDE/*/chmod a+w /etc/passwd /etc/group} \
    && ln -s "${HOME}"/readme.md "${HOME}"/Desktop/README \
    && chmod 755 -R "${STARTUPDIR}" \
    && "${STARTUPDIR}"/set_user_permissions.sh "${STARTUPDIR}" "${HOME}"

COPY ./novnc/run_sitl.sh /root/run_sitl.sh

USER 1001

ENTRYPOINT [ "/usr/bin/tini", "--", "/dockerstartup/startup.sh" ]
# ENTRYPOINT [ "/usr/bin/tini", "--", "tail", "-f", "/dev/null" ]


# ##################
# ### METADATA STAGE
# ##################

# FROM stage_final as stage_metadata
# ARG ARG_CREATED
# ARG ARG_DOCKER_TAG
# ARG ARG_VCS_REF
# ARG ARG_VERSION_STICKER

# LABEL \
#     org.opencontainers.image.authors="accetto" \
#     org.opencontainers.image.created="${ARG_CREATED}" \
#     org.opencontainers.image.description="Headless Ubuntu/Xfce/VNC/noVNC containers with Internet browsers" \
#     org.opencontainers.image.documentation="https://github.com/accetto/ubuntu-vnc-xfce-g3" \
#     org.opencontainers.image.source="https://github.com/accetto/ubuntu-vnc-xfce-g3" \
#     org.opencontainers.image.title="accetto/ubuntu-vnc-xfce-g3" \
#     org.opencontainers.image.url="https://github.com/accetto/ubuntu-vnc-xfce-g3" \
#     org.opencontainers.image.vendor="https://github.com/accetto" \
#     org.opencontainers.image.version="${ARG_DOCKER_TAG}"

# LABEL \
#     org.label-schema.vcs-url="https://github.com/accetto/ubuntu-vnc-xfce-g3" \
#     org.label-schema.vcs-ref="${ARG_VCS_REF}"

# LABEL \
#     any.accetto.version-sticker="${ARG_VERSION_STICKER}"
