FROM python:3.11.3-alpine
LABEL version="6.2.0"
LABEL maintainer="Arnaud,Jorge,Anthony"

RUN sed -i '2s/^# *//' /etc/apk/repositories && \
    apk update && \
    apk add --no-cache openjdk17 curl unzip

ENV ACTIVEMQ_VERSION=6.1.6
ENV ACTIVEMQ=apache-activemq-${ACTIVEMQ_VERSION}
ENV ACTIVEMQ_HOME=/opt/activemq

# Téléchargement ActiveMQ
RUN curl -sSL "https://www.apache.org/dyn/closer.cgi?filename=/activemq/${ACTIVEMQ_VERSION}/${ACTIVEMQ}-bin.tar.gz&action=download" \
    | tar -xz -C /opt && \
    mv /opt/${ACTIVEMQ} ${ACTIVEMQ_HOME}

# Ajout user + permissions
RUN addgroup -S activemq && \
    adduser -S -H -G activemq -h ${ACTIVEMQ_HOME} activemq && \
    chown -R activemq:activemq ${ACTIVEMQ_HOME} && \
    chown -h activemq:activemq ${ACTIVEMQ_HOME}

# Modif jetty avec sed + chown/chgrp + cp
RUN cat ${ACTIVEMQ_HOME}/conf/jetty.xml | \
    sed -E 's/(.*)("127\.0\.0\.1")(.*)/\1"0.0.0.0"\3/g;t' > ${ACTIVEMQ_HOME}/conf/jetty2.xml && \
    chown activemq ${ACTIVEMQ_HOME}/conf/jetty2.xml && \
    chgrp activemq ${ACTIVEMQ_HOME}/conf/jetty2.xml && \
    cp ${ACTIVEMQ_HOME}/conf/jetty2.xml ${ACTIVEMQ_HOME}/conf/jetty.xml

# Modif activemq.xml pour JMS transformer
RUN sed -E 's#(5672\?maximumConnections=1000&amp;wireFormat\.maxFrameSize=104857600)#\1\&amp;transport.transformer=jms#g' \
    ${ACTIVEMQ_HOME}/conf/activemq.xml > ${ACTIVEMQ_HOME}/conf/activemq.xml.tmp && \
    mv ${ACTIVEMQ_HOME}/conf/activemq.xml.tmp ${ACTIVEMQ_HOME}/conf/activemq.xml

# Télécharger AMQC front (static)
RUN curl -LOk https://github.com/snuids/AMQC/archive/master.zip && \
    unzip master.zip && \
    rm master.zip && \
    mv AMQC-master /opt/static

# Télécharger AMQC proxy (server) depuis le repo spécifique
RUN curl -LOk https://github.com/acarpriau/amqc-proxy/archive/main.zip && \
    unzip main.zip && \
    rm main.zip && \
    mv amqc-proxy-main/server /opt/server && \
    rm -rf amqc-proxy-main

# Install Python deps
RUN pip3 install --no-cache-dir -r /opt/server/requirements.txt

# Expose ports (documentaire)
EXPOSE 1883 5672 8161 8168 8180 61613 61614 61616

# CMD : build config dynamique + start ActiveMQ + start FastAPI
CMD \
    echo "${ACTIVEMQ_USER}:${ACTIVEMQ_PASS}" > ${ACTIVEMQ_HOME}/conf/users.properties && \
    cat ${ACTIVEMQ_HOME}/conf/log4j2.properties | \
        sed -E "s/(rootLogger\.level=INFO)/rootLogger.level=${ACTIVEMQ_LOG_LEVEL:-INFO}/g" > ${ACTIVEMQ_HOME}/conf/log4j2.properties.tmp && \
    mv ${ACTIVEMQ_HOME}/conf/log4j2.properties.tmp ${ACTIVEMQ_HOME}/conf/log4j2.properties && \
    ACTIVEMQ_OPTS_MEMORY="${ACTIVEMQ_OPTS_MEMORY}" /opt/activemq/bin/activemq start && \
    uvicorn main:app --host ${AMQC_HOST:-0.0.0.0} --port ${AMQC_PORT:-8180} --app-dir /opt/server
