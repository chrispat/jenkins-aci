FROM jenkinsci/jnlp-slave

USER root

RUN apt-get update \
    && apt-get install -y --no-install-recommends cifs-utils

USER jenkins