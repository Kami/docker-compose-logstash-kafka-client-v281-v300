# We build custom version of logstash-input-kafka input which depends on newer version of
# Kafka client library which fixes some consensus bugs and security issue
FROM jruby:9.2.20.1-jdk17 as builder

WORKDIR /tmp
RUN ruby --version

# Download and install patched ruby maven which is needed to build gems under newer JDKs
RUN git clone --depth 1 https://github.com/jruby/ruby-maven.git --branch master
WORKDIR /tmp/ruby-maven

RUN ls -la
RUN gem list installed
RUN gem build ruby-maven.gemspec
RUN gem install ruby-maven-3.3.12.gem
RUN gem list installed

# This branch contains updates to the plugin to use Kafka client 2.8.1 and various other fixes
# and changes which are needed to build the project
# NOTE: There are some changes in terms of kafka input option names in the new client version:
#
# * max_poll_records value now needs to be a string and not an int / long
# * partition_assignment_strategy now needs to contain a full class name, e.g. org.apache.kafka.clients.consumer.CooperativeStickyAssignor
# * decorate_events is now a boolean so it must contain string "true"
WORKDIR /tmp

RUN git clone --depth 1 https://github.com/Kami/logstash-input-kafka.git --branch upgrade_kafka_client

WORKDIR /tmp/logstash-input-kafka

# Needed so we can build the gem
ENV JAVA_OPTS="--add-opens java.base/java.lang=ALL-UNNAMED"

# Build them gem with all the JAR dependencies
RUN cp gradle/wrapper/gradle-wrapper.properties.gradle73 gradle/wrapper/gradle-wrapper.properties
RUN mv build.gradle.gradle73 build.gradle

# By default we use and test Kafka client library, v2.8.1, to test against v3.0.0, uncomment the
# lines below
# To confirm that correct version is used, tail logstash container and check for a line which looks
# like this. Kafka version: should match either v2.8.1 or v3.0.0 depending if the lines below are
# commented out or not.
# logstash_1   | [2022-01-10T15:32:23,802][INFO ][org.apache.kafka.common.utils.AppInfoParser][main][b22072d516211e5e582ae859207c513cb034b3630d008756273bfa6d8515bd15] Kafka version: 3.0.0
RUN sed -i "s#2.8.1#3.0.0#g" build.gradle
RUN sed -i "s#2.8.1#3.0.0#g" lib/logstash-input-kafka_jars.rb

RUN cat gradle/wrapper/gradle-wrapper.properties
RUN cat build.gradle

RUN bundle install
RUN gem list installed
RUN bundle exec rake vendor
RUN gem build logstash-input-kafka.gemspec

RUN ls -lah .
RUN ls -lah vendor/
RUN ls -lah vendor/jar-dependencies
RUN ls -lah vendor/jar-dependencies/*/*/*/*/*
RUN ls -lah *.gem

FROM docker.elastic.co/logstash/logstash-oss:7.16.2 as main

USER root
RUN yum -y install wget curl zip

COPY --from=builder /tmp/logstash-input-kafka/logstash-input-kafka-9.2.0.gem .

# Install OpenJDK 17
RUN wget https://download.java.net/java/GA/jdk17.0.1/2a2082e5a09d4267845be086888add4f/12/GPL/openjdk-17.0.1_linux-x64_bin.tar.gz
RUN echo "1c0a73cbb863aad579b967316bf17673b8f98a9bb938602a140ba2e5c38f880a openjdk-17.0.1_linux-x64_bin.tar.gz" > shasums
RUN set -e && sha256sum -c shasums
RUN tar xvf openjdk-17.0.1_linux-x64_bin.tar.gz
RUN ls -la .

RUN mv jdk-17.0.1 /opt/
RUN rm openjdk-17.0.1_linux-x64_bin.tar.gz
RUN rm shasums

# By default logstash uses bundled JRE 11, but we want to use JDK 17 for performance reasons
# NOTE: logstash >= v0.7.0 utilizes LS_ prefixed environment variables.
ENV JAVA_HOME=/opt/jdk-17.0.1
ENV LS_JAVA_HOME=/opt/jdk-17.0.1
ENV PATH=$PATH:$JAVA_HOME/bin

USER logstash

RUN logstash-plugin list --verbose

# We need to remove bundled kafka integration (input + output) so we can install updated version of the input
RUN logstash-plugin remove logstash-integration-kafka
RUN logstash-plugin install logstash-input-kafka-9.2.0.gem

RUN logstash-plugin list --verbose
RUN find /opt/logstash -name "kafka-clients*.jar" 2>&1

# Remove old Java Kafka client version
RUN rm -rf /opt/logstash/vendor/bundle/jruby/2.5.0/gems/logstash-integration-kafka-10.8.2-java/
RUN rm -rf /usr/share/logstash/vendor/bundle/jruby/2.5.0/gems/logstash-integration-kafka-10.8.2-java

# For some reason Elasticsearch doesn't scan and use gem local vendored jars so
# we need to copy all kafka clients vendor jars into global jar class path :/
# Right now we don't use any compression so we don't also copy over snappy and zstd jar. If we
# ever enabled snappy or zstd compression, those jars would need to be copied over as well
# See https://github.com/elastic/logstash/blob/main/logstash-core/lib/logstash/environment.rb#L154
RUN ls -la /opt/logstash/vendor/local_gems/*/logstash-input-kafka-9.2.0/vendor/jar-dependencies/org/apache/kafka/kafka-clients/*/
RUN cp /opt/logstash/vendor/local_gems/*/logstash-input-kafka-9.2.0/vendor/jar-dependencies/org/apache/kafka/kafka-clients/*/kafka-clients-*.jar /usr/share/logstash/logstash-core/lib/jars/
RUN find /opt/logstash -name "*kafka*clients*.jar"
RUN find /usr/share/logstash -name "*kafka*clients*.jar"

USER root
RUN chown 1000:1000 -R /usr/share/logstash/pipeline && \
    chown 1000:1000 -R /usr/share/logstash/config

USER logstash

# Recent versions of logstash use LS_JAVA_OPTS and not JAVA_OPTS variable.
#
# 1. --adds-open is added since we use custom JDK and not the one which is bundled by logstash
# where that option is already set
ENV LS_JAVA_OPTS="--add-opens java.base/sun.nio.ch=ALL-UNNAMED --add-opens java.base/java.io=ALL-UNNAMED"
