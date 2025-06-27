# 构建阶段：准备 Node 环境和安装依赖
FROM node:20-slim as nodebuilder

FROM python:3.11-slim-bullseye as builder

ARG QL_MAINTAINER="whyour"
LABEL maintainer="${QL_MAINTAINER}"
ARG QL_URL=https://github.com/${QL_MAINTAINER}/qinglong.git
ARG QL_BRANCH=debian

ENV QL_DIR=/ql \
    QL_BRANCH=${QL_BRANCH}

# 从 nodebuilder 复制 node 和 npm
COPY --from=nodebuilder /usr/local/bin/node /usr/local/bin/
COPY --from=nodebuilder /usr/local/lib/node_modules/. /usr/local/lib/node_modules/

# 创建 npm 软链接并安装基础包，配置 git，拉取源码
RUN set -x && \
    ln -s /usr/local/lib/node_modules/npm/bin/npm-cli.js /usr/local/bin/npm && \
    apt-get update && \
    apt-get install --no-install-recommends -y libatomic1 git && \
    git config --global user.email "qinglong@@users.noreply.github.com" && \
    git config --global user.name "qinglong" && \
    git config --global http.postBuffer 524288000 && \
    git clone --depth=1 -b ${QL_BRANCH} ${QL_URL} ${QL_DIR}

# 复制 package.json、.npmrc、pnpm-lock.yaml 便于安装依赖
RUN mkdir /tmp/build
RUN cp ${QL_DIR}/package.json ${QL_DIR}/.npmrc ${QL_DIR}/pnpm-lock.yaml /tmp/build/

# 安装 pnpm，并安装生产依赖
RUN npm i -g pnpm@8.3.1 && \
    cd /tmp/build && \
    pnpm install --prod


# 最终镜像阶段
FROM python:3.11-slim-bullseye

ARG QL_MAINTAINER="whyour"
LABEL maintainer="${QL_MAINTAINER}"
ARG QL_URL=https://github.com/${QL_MAINTAINER}/qinglong.git
ARG QL_BRANCH=debian

ENV PNPM_HOME=/root/.local/share/pnpm \
    PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/root/.local/share/pnpm:/root/.local/share/pnpm/global/5/node_modules:$PNPM_HOME \
    NODE_PATH=/usr/local/bin:/usr/local/pnpm-global/5/node_modules:/usr/local/lib/node_modules:/root/.local/share/pnpm/global/5/node_modules \
    LANG=C.UTF-8 \
    SHELL=/bin/bash \
    PS1="\u@\h:\w \$ " \
    QL_DIR=/ql \
    QL_BRANCH=${QL_BRANCH} \
    DEBIAN_FRONTEND=noninteractive \
    APT::Acquire::Retries=3 \
    APT::Acquire::http::Timeout="10"

# 复制 node 和 npm
COPY --from=nodebuilder /usr/local/bin/node /usr/local/bin/
COPY --from=nodebuilder /usr/local/lib/node_modules/. /usr/local/lib/node_modules/

# 创建 npm 和 npx 软链接，更新系统，安装依赖软件包，配置时区和 git，安装 pnpm 和其他全局 npm 包，清理缓存
RUN set -x && \
    ln -s /usr/local/lib/node_modules/npm/bin/npm-cli.js /usr/local/bin/npm && \
    ln -s /usr/local/lib/node_modules/npm/bin/npx-cli.js /usr/local/bin/npx

RUN apt-get update && apt-get upgrade -y

RUN apt-get install --no-install-recommends -y \
    git \
    curl \
    sudo \
    build-essential \
    default-jdk \
    default-jre \
    g++ \
    gcc \
    libzbar0 \
    fish \
    ffmpeg \
    nmap \
    ca-certificates \
    zsh \
    cron \
    wget \
    tzdata \
    perl \
    openssl \
    openssh-client \
    nginx \
    jq \
    procps \
    netcat \
    sshpass \
    rclone \
    unzip \
    libatomic1

RUN apt-get clean && rm -rf /var/lib/apt/lists/*

RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo "Asia/Shanghai" > /etc/timezone

RUN git config --global user.email "qinglong@@users.noreply.github.com" && \
    git config --global user.name "qinglong" && \
    git config --global http.postBuffer 524288000

RUN npm install -g pnpm@8.3.1 pm2 ts-node

RUN rm -rf /root/.pnpm-store /root/.local/share/pnpm/store /root/.cache /root/.npm

RUN chmod u+s /usr/sbin/cron

RUN ulimit -c 0


# 安装 code-server 固定版本
RUN curl -fsSL https://code-server.dev/install.sh | sh -s -- --version=4.96.4

# 安装 ollama
RUN curl -fsSL https://ollama.com/install.sh | sh


# 拉取 QingLong 代码，配置权限，复制静态资源，删除旧入口文件
ARG SOURCE_COMMIT
RUN git clone --depth=1 -b ${QL_BRANCH} ${QL_URL} ${QL_DIR} && \
    cd ${QL_DIR} && \
    cp -f .env.example .env && \
    chmod 777 ${QL_DIR}/shell/*.sh && \
    chmod 777 ${QL_DIR}/docker/*.sh && \
    git clone --depth=1 -b ${QL_BRANCH} https://github.com/${QL_MAINTAINER}/qinglong-static.git /static && \
    mkdir -p ${QL_DIR}/static && \
    cp -rf /static/* ${QL_DIR}/static && \
    rm -rf /static && \
    rm -f ${QL_DIR}/docker/docker-entrypoint.sh

# 复制自定义入口文件和配置
COPY docker-entrypoint.sh ${QL_DIR}/docker
COPY front.conf ${QL_DIR}/docker/front.conf

# 创建目录及修改权限
RUN mkdir -p /ql/data/config /ql/data/log /ql/data/db /ql/data/scripts /ql/data/repo /ql/data/raw /ql/data/deps && \
    chmod -R 777 /ql /var /usr/local /etc/nginx /run /usr /root

# 复制构建阶段安装好的 node_modules 和服务相关文件
COPY --from=builder /tmp/build/node_modules/. /ql/node_modules/
COPY services.json /ql/node_modules/nodemailer/lib/well-known/
COPY notify.py /notify.py


# 创建 coder 用户并配置 sudo 无密码
RUN useradd -m -s /bin/zsh coder && \
    echo 'coder ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers

# 创建 code-server 配置目录并设置权限
RUN mkdir -p /home/coder/.local/share/code-server/User && chmod -R 777 /home/coder

# 写入 code-server 设置文件（启用暗黑主题和遥测）
RUN echo '{ \
   "workbench.colorTheme": "Default Dark Modern", \
    "telemetry.enableTelemetry": true, \
    "telemetry.enableCrashReporter": true \
}' > /home/coder/.local/share/code-server/User/settings.json

# 设置配置目录归属 coder 用户
RUN chown -R coder:coder /home/coder/.local/share/code-server

# 切换到 coder 用户
USER coder

ENV HOME=/home/coder \
    PATH=/home/coder/.local/bin:$PATH

WORKDIR ${QL_DIR}

# 运行 rclone 配置帮助验证
RUN rclone config -h

# 健康检查接口
HEALTHCHECK --interval=5s --timeout=2s --retries=20 \
  CMD curl -sf --noproxy '*' http://127.0.0.1:5400/api/health || exit 1

ENTRYPOINT ["./docker/docker-entrypoint.sh"]

VOLUME /ql/data

EXPOSE 5700
