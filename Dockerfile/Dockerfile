FROM  harbor.justbeta.pro/devops/jnlp-slave:v1.1
ENV MAVEN_VERSION 3.8.4
ENV MAVEN_HOME /opt/maven/apache-maven-${MAVEN_VERSION}
#ENV JAVA_HOME /usr/local/java
#ENV CLASSPATH .:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
# build java
# COPY ./usr/local/java ${JAVA_HOME}
# COPY ./libltdl.so.7 /usr/lib/libltdl.so.7
# build maven
COPY apache-maven-${MAVEN_VERSION}-bin.tar.gz /tmp/maven/apache-maven-${MAVEN_VERSION}-bin.tar.gz
# COPY settings.xml /tmp/maven/settings.xml
USER root:root
RUN mkdir -p /opt/maven/repository \
&& cd /opt/maven \
&& tar -zxvf /tmp/maven/apache-maven-${MAVEN_VERSION}-bin.tar.gz \
# && cp /tmp/maven/settings.xml ${MAVEN_HOME}/conf/settings.xml \
&& rm -rf /tmp/maven \
&& apt-get -yq update \
&& apt-get -yq --no-install-recommends --no-install-suggests install sshpass \
&& apt-get clean -y \
&& mkdir -p /home/jenkins/workspace/
# build kubectl
RUN curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" \
&&  install -o jenkins -g jenkins -m 0755 kubectl /usr/local/bin/kubectl  && chmod +x /usr/local/bin/kubectl
ENV PATH ${MAVEN_HOME}/bin:${PATH}
# build docker
RUN apt-get -y install apt-transport-https  \
    ca-certificates     curl     gnupg2     software-properties-common \
&&    curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/debian/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg \ 
&&    echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://mirrors.aliyun.com/docker-ce/linux/debian \
    buster stable" |tee /etc/apt/sources.list.d/docker.list > /dev/null && apt-get update && apt-get -y install docker-ce docker-ce-cli containerd.io
USER jenkins