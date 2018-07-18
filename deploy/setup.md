# Setup

Simply install Hono according to the deployment guide for OpenShift with S2I (see <https://www.eclipse.org/hono/deployment/openshift_s2i>).

As the setup guide only reflects the most recent version of Hono, the following is a brief summary of how
we installed it.

## OpenShift setup

Allow the OpenShift router to process 20.000 * 2 (in + out) connections at the same time:

    oc env deploymentconfigs/router ROUTER_MAX_CONNECTIONS=40000

## IoT cluster - Hono 0.6

    git clone -b 0.6.x https://github.com/eclipse/hono.git

    oc new-project enmasse --display-name='EnMasse Instance'
    curl -LO https://github.com/EnMasseProject/enmasse/releases/download/0.21.0/enmasse-0.21.0.tgz
    tar xzf enmasse-0.21.0.tgz
    
    ./enmasse-0.21.0/deploy.sh -n enmasse -m https://wonderful.iot-playground.org:8443 -user <user>
    
    curl -X POST --insecure -T addresses.json -H "content-type: application/json" https://$(oc -n enmasse get route restapi -o jsonpath='{.spec.host}')/apis/enmasse.io/v1alpha1/namespaces/enmasse/addressspaces/default/addresses

    oc new-project hono --display-name='Eclipse Hono™'
    oc create configmap influxdb-config --from-file="../influxdb.conf"
    oc process -f hono-template.yml   -p "ENMASSE_NAMESPACE=enmasse"   -p "GIT_REPOSITORY=https://github.com/eclipse/hono.git"   -p "GIT_BRANCH=0.6.x"| oc create -f -
    oc env -n hono dc/hono-service-device-registry HONO_REGISTRY_SVC_MAX_DEVICES_PER_TENANT=10000

    oc new-project grafana --display-name='Grafana Dashboard'
    oc process -f grafana-template.yml   -p ADMIN_PASSWORD=admin | oc create -f -
    ../configure_grafana.sh "$(oc get route grafana -o jsonpath='{.spec.host}')" 80

## IoT cluster - Hono 0.7-SNAPSHOT

    git clone -b scaletest2/0.7-SNAPSHOT https://github.com/redhat-iot/hono.git

    oc new-project enmasse --display-name='EnMasse Instance'
    curl -LO https://github.com/EnMasseProject/enmasse/releases/download/0.21.0/enmasse-0.21.0.tgz
    tar xzf enmasse-0.21.0.tgz
    
    ./enmasse-0.21.0/deploy.sh -n enmasse -m https://wonderful.iot-playground.org:8443 -user <user>

    oc new-project hono --display-name='Eclipse Hono™'
    cd example/src/main/deploy/openshift_s2i
    oc create configmap influxdb-config --from-file="../influxdb.conf"
    oc process -f hono-template.yml   -p "ENMASSE_NAMESPACE=enmasse"   -p "GIT_REPOSITORY=https://github.com/redhat-iot/hono.git"   -p "GIT_BRANCH=scaletest/2/0.7-SNAPSHOT"| oc create -f -

    oc new-project grafana --display-name='Grafana Dashboard'
    oc create configmap grafana-provisioning-datasources --from-file=../../config/grafana/provisioning/datasources
    oc create configmap grafana-provisioning-dashboards --from-file=../../config/grafana/provisioning/dashboards
    oc create configmap grafana-dashboard-defs --from-file=../../config/grafana/dashboard-definitions
    oc process -f grafana-template.yml -p "GIT_REPOSITORY=https://github.com/redhat-iot/hono.git" -p GIT_BRANCH=scaletest/2/0.7-SNAPSHOT -p ADMIN_PASSWORD=admin | oc create -f -


## Simulation cluster

# Scenarios

## Scenario – Dummy device registry

**Note:** The dummy device registry is currently only available in a forked version of Hono.
It might become part of the upstream project at a later time. Or never at all.

Enable the dummy implementation by executing the following command on the IoT cluster:

    oc env -n hono dc/hono-service-device-registry HONO_APP_TYPE=dummy

## Using node ports

On the IoT cluster:

    oc -n hono create -f hono/nodeport.yml

Switch to node port:

    oc -n simulator env dc/simulator-http HONO_HTTP_URL=http://http.wonderful.iot-playground.org:30080

Revert change:

    oc -n simulator env dc/simulator-http HONO_HTTP_URL=http://hono-adapter-http-vertx-hono.wonderful.iot-playground.org
