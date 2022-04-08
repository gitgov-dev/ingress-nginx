# Copyright 2015 The Kubernetes Authors. All rights reserved.
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

ARG BASE_IMAGE

FROM ${BASE_IMAGE}

ARG TARGETARCH
ARG VERSION
ARG COMMIT_SHA
ARG BUILD_ID=UNSET

ENV NGINX_VERSION nginx-1.19.10

LABEL org.opencontainers.image.title="NGINX Ingress Controller for Kubernetes"
LABEL org.opencontainers.image.documentation="https://kubernetes.github.io/ingress-nginx/"
LABEL org.opencontainers.image.source="https://github.com/kubernetes/ingress-nginx"
LABEL org.opencontainers.image.vendor="The Kubernetes Authors"
LABEL org.opencontainers.image.licenses="Apache-2.0"
LABEL org.opencontainers.image.version="${VERSION}"
LABEL org.opencontainers.image.revision="${COMMIT_SHA}"

LABEL build_id="${BUILD_ID}"

WORKDIR /etc/nginx

RUN apk update && \
    apk upgrade && \
    apk add diffutils && \
    apk add git && \
    apk del nginx && \
    apk add openssl-dev pcre-dev zlib-dev wget build-base git libxslt-dev libxml2-dev geoip-dev


# ----- BUILD NGINX -----

RUN mkdir -p /tmp/src && \
    cd /tmp/src && \
    git clone https://github.com/arut/nginx-dav-ext-module.git && \
    git clone https://github.com/pschultz/ngx_http_allow_methods_module.git && \
    git clone https://github.com/vision5/ngx_devel_kit.git && \
    git clone https://github.com/openresty/headers-more-nginx-module.git && \
    git clone --depth 1 --branch v0.10.20 https://github.com/openresty/lua-nginx-module.git && \
    git clone --depth 1 --branch v0.0.10 https://github.com/openresty/stream-lua-nginx-module.git && \
    git clone https://github.com/yaoweibin/nginx_ajp_module.git && \
    wget http://nginx.org/download/${NGINX_VERSION}.tar.gz && \
    tar -zxvf ${NGINX_VERSION}.tar.gz && \
    cd /tmp/src/${NGINX_VERSION} && \
    export LUAJIT_LIB=/usr/local/lib/libluajit-5.1.so.2.1.0 && \
    export LUAJIT_INC=/usr/local/include/luajit-2.1 && \
    ./configure \
      --with-http_ssl_module \
      --with-http_gzip_static_module \
      --with-http_dav_module \
      --with-http_geoip_module \
      --with-http_v2_module \
      --with-http_stub_status_module \
      --with-stream \
      --with-stream_ssl_module \
      --add-module=/tmp/src/nginx-dav-ext-module \
      --add-module=/tmp/src/ngx_http_allow_methods_module \
      --add-module=/tmp/src/ngx_devel_kit \
      --add-module=/tmp/src/headers-more-nginx-module \
      --add-module=/tmp/src/lua-nginx-module \
      --add-module=/tmp/src/stream-lua-nginx-module \
      --add-module=/tmp/src/nginx_ajp_module \
      --pid-path=/tmp/nginx.pid \
      --http-log-path=/var/log/nginx/access.log \
      --error-log-path=/var/log/nginx/error.log \
      --with-compat && \
    make && \
    make install && \
    touch /tmp/nginx.pid && \
    chown www-data:www-data /tmp/nginx.pid && \
    apk del build-base && \
    rm -rf /tmp/src && \
    rm -rf /var/cache/apk/*


COPY --chown=www-data:www-data etc /etc
COPY --chown=www-data:www-data extra/nginx.conf /usr/local/nginx/conf/nginx.conf
COPY --chown=www-data:www-data extra/modules/ /modules/
COPY --chown=www-data:www-data bin/${TARGETARCH}/dbg /
COPY --chown=www-data:www-data bin/${TARGETARCH}/nginx-ingress-controller /
COPY --chown=www-data:www-data bin/${TARGETARCH}/wait-shutdown /

# Check nginx dummy config for errors
RUN nginx -t

# ----- BUILD NGINX END -----

# Fix permission during the build to avoid issues at runtime
# with volumes (custom templates)
RUN bash -xeu -c ' \
  writeDirs=( \
    /etc/ingress-controller \
    /etc/ingress-controller/ssl \
    /etc/ingress-controller/auth \
    /var/log \
    /var/log/nginx \
  ); \
  for dir in "${writeDirs[@]}"; do \
    mkdir -p ${dir}; \
    chown -R www-data.www-data ${dir}; \
  done'

RUN apk add --no-cache libcap \
  && setcap    cap_net_bind_service=+ep /nginx-ingress-controller \
  && setcap -v cap_net_bind_service=+ep /nginx-ingress-controller \
  && setcap    cap_net_bind_service=+ep /usr/local/nginx/sbin/nginx \
  && setcap -v cap_net_bind_service=+ep /usr/local/nginx/sbin/nginx \
  && setcap    cap_net_bind_service=+ep /usr/bin/dumb-init \
  && setcap -v cap_net_bind_service=+ep /usr/bin/dumb-init \
  && apk del libcap

# Create symlinks to redirect nginx logs to stdout and stderr docker log collector
RUN ln -sf /dev/stdout /var/log/nginx/access.log && \
    ln -sf /dev/stderr /var/log/nginx/error.log

ENTRYPOINT ["/usr/bin/dumb-init", "--"]

CMD ["/nginx-ingress-controller"]
