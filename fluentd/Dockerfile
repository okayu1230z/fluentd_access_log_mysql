FROM fluent/fluentd:v1.14-debian-1
USER root
RUN apt update \
    && apt upgrade -y \
    && apt install -y default-libmysqlclient-dev

RUN buildDeps="sudo make gcc g++ libc-dev ruby-dev" \
    && apt update \
    && apt install -y --no-install-recommends $buildDeps \
    && sudo gem install \
           fluent-plugin-mysql \
    && sudo gem sources --clear-all \
    && SUDO_FORCE_REMOVE=yes \
       apt-get purge -y --auto-remove \
                     -o APT::AutoRemove::RecommendsImportant=false \
                     $buildDeps \
    && rm -rf /var/lib/apt/lists/* \
              /home/fluent/.gem/ruby/2.3.0/cache/*.gem
