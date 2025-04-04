FROM quay.io/jupyter/minimal-notebook:latest
ARG NOTEBOOK_HOME=/home/${NB_USER}

WORKDIR ${NOTEBOOK_HOME}/data

USER root
WORKDIR ${NOTEBOOK_HOME}

# Install dependencies: Java 21 and Maven
RUN apt-get update && apt-get install -y openjdk-21-jdk maven

# Copy the pre-created Maven project and jjava-glue project
COPY ./jupyter/java ${NOTEBOOK_HOME}/java
COPY ./jupyter/install.py ${NOTEBOOK_HOME}/install.py
COPY ./jjava-glue ${NOTEBOOK_HOME}/jjava-glue

# Build the jjava-glue project first and install it locally
WORKDIR ${NOTEBOOK_HOME}/jjava-glue
RUN mvn clean install -DskipTests

# Create a list of jjava-glue artifacts directly
RUN mvn -q dependency:build-classpath -Dmdep.outputFile=${NOTEBOOK_HOME}/jjava-glue-classpath.txt

# Use Maven to download dependencies for JJava
WORKDIR ${NOTEBOOK_HOME}/java

# Download the JJava jar directly
RUN mvn dependency:get -Dartifact=org.dflib.jjava:jjava:1.0-M3 -Ddest=./ -Dtransitive=false
RUN ln -sf jjava-1.0-M3.jar jjava.jar

# Download all dependencies
RUN mvn dependency:copy-dependencies -DoutputDirectory=./lib

# Create a list of dependencies for the classpath, add jjava.jar, jjava-glue, jjava-glue
RUN cat <<EOF > ${NOTEBOOK_HOME}/java/classpath.txt
$(find ./lib -name "*.jar" | tr '\n' ':')\
${NOTEBOOK_HOME}/java/jjava.jar:\
$(cat ${NOTEBOOK_HOME}/jjava-glue-classpath.txt):\
${NOTEBOOK_HOME}/jjava-glue/core-glue/target/core-glue-0.1.0-SNAPSHOT.jar:\
${NOTEBOOK_HOME}/jjava-glue/dflib-glue/target/dflib-glue-0.1.0-SNAPSHOT.jar:\
${NOTEBOOK_HOME}/jjava-glue/jfreechart-glue/target/jfreechart-glue-0.1.0-SNAPSHOT.jar:\
${NOTEBOOK_HOME}/jjava-glue/commons-math-glue/target/commons-math-glue-0.1.0-SNAPSHOT.jar
EOF

# Install the kernel with classpath configuration
WORKDIR ${NOTEBOOK_HOME}
RUN python install.py --prefix /opt/conda/ --classpath $(cat ${NOTEBOOK_HOME}/java/classpath.txt)

# Clean up Maven artifacts but keep the jjava.jar and lib directory
RUN rm -rf ${NOTEBOOK_HOME}/java/target ${NOTEBOOK_HOME}/java/.m2 ${NOTEBOOK_HOME}/java/pom.xml \
    ${NOTEBOOK_HOME}/java/classpath.txt ${NOTEBOOK_HOME}/jjava-glue-classpath.txt \
    && rm -f ${NOTEBOOK_HOME}/install.py

# Install conda packages from environment.yml
COPY ./jupyter/environment.yml /tmp/
RUN conda env update -f /tmp/environment.yml && \
    conda clean --all -f -y && \
    fix-permissions "${CONDA_DIR}" && \
    fix-permissions "/home/${NB_USER}"

# Install jupyter_darkmodern theme
RUN pip install jupyter_darkmodern && \
    fix-permissions "${CONDA_DIR}" && \
    fix-permissions "/home/${NB_USER}"

WORKDIR ${NOTEBOOK_HOME}
USER $NB_UID
