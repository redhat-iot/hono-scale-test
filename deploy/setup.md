# Setup

Simply install Hono according to the deployment guide for OpenShift with S2I (see <https://www.eclipse.org/hono/deployment/openshift_s2i>).

As the setup guide only reflects the most recent version of Hono, the following is a brief summary of how
we installed it.

## IoT cluster

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

## Simulation cluster

