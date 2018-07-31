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
    
    ./enmasse-0.21.0/deploy.sh -n enmasse -m https://wonderful.iot-playground.org:8443 -u <user>
    # wait until enmasse is ready
    curl -X POST --insecure -T example/src/main/deploy/openshift_s2i/addresses.json -H "content-type: application/json" https://$(oc -n enmasse get route restapi -o jsonpath='{.spec.host}')/apis/enmasse.io/v1alpha1/namespaces/enmasse/addressspaces/default/addresses

    oc new-project hono --display-name='Eclipse Hono™'
    cd example/src/main/deploy/openshift_s2i
    oc create configmap influxdb-config --from-file="../influxdb.conf"
    oc process -f hono-template.yml   -p "ENMASSE_NAMESPACE=enmasse"   -p "GIT_REPOSITORY=https://github.com/redhat-iot/hono.git"   -p "GIT_BRANCH=scaletest/2/0.7-SNAPSHOT"| oc create -f -

    oc new-project grafana --display-name='Grafana Dashboard'
    oc create configmap grafana-provisioning-datasources --from-file=../../config/grafana/provisioning/datasources
    oc create configmap grafana-provisioning-dashboards --from-file=../../config/grafana/provisioning/dashboards
    oc create configmap grafana-dashboard-defs --from-file=../../config/grafana/dashboard-definitions
    oc process -f grafana-template.yml -p "GIT_REPOSITORY=https://github.com/redhat-iot/hono.git" -p GIT_BRANCH=scaletest/2/0.7-SNAPSHOT -p ADMIN_PASSWORD=admin | oc create -f -

### Move qdrouterd to infra region

On the master node of the IoT cluster:

    oc annotate ns/enmasse --overwrite 'openshift.io/node-selector='
    oc patch  deploy/qdrouterd -p '{"spec":{"template":{"spec":{"nodeSelector":{"region":"infra"}}}}}

## Simulation cluster

Extract the EnMasse certificate:

    oc -n enmasse extract secret/external-certs-messaging
    # Creates three files, we only need "tls.crt" 

On the master node:

    git clone https://github.com/redhat-iot/hono-scale-test.git
    cd hono-scale-test
    
    oc create -f deploy/simulator/pv-influxdb.yml
    oc create -f deploy/grafana/pv-grafana.yml

    oc new-project simulator
    oc process -f deploy/simulator/template.yml | oc create -f -
    oc create configmap simulator-config --from-file=server-cert.pem=tls.crt --dry-run -o yaml | oc replace -f -

    oc new-project grafana
    oc process -f deploy/grafana/template.yml | oc create -f -

# Scenarios

## Scenario – Dummy device registry

**Note:** The dummy device registry is currently only available in a forked version of Hono.
It might become part of the upstream project at a later time. Or never at all.

Enable the dummy implementation by executing the following command on the IoT cluster:

    oc env -n hono dc/hono-service-device-registry HONO_APP_TYPE=dummy

## Using node ports

On the IoT cluster:

    oc -n hono create -f hono/nodeport.yml

Switch to node port on the simulation cluster:

    oc -n simulator env dc/simulator-http HONO_HTTP_URL=http://http.wonderful.iot-playground.org:30080

Revert change:

    oc -n simulator env dc/simulator-http HONO_HTTP_URL=http://hono-adapter-http-vertx-hono.wonderful.iot-playground.org

## Using qdrouter node ports

On the IoT cluster:

    oc -n enmasse create -f enmasse/nodeport.yml

Switch to node port on the simulation cluster:

    oc -n simulator env dc/simulator-consumer-influxdb MESSAGING_SERVICE_HOST=hono-adapter-http-vertx-hono.wonderful.iot-playground.org MESSAGING_SERVICE_PORT_AMQP=30671

Revert change:

    oc -n simulator env dc/simulator-consumer-influxdb MESSAGING_SERVICE_HOST=messaging-enmasse.wonderful.iot-playground.org MESSAGING_SERVICE_PORT_AMQP=443

## Link capacity

This scenario sets the link capacity on all components to 5.000. Defaults otherwise range from 50 to 200 over
various components.

On the IoT cluster:

    oc -n enmasse env deployment/qdrouterd -c router LINK_CAPACITY=5000
    oc -n hono env dc/hono-adapter-http-vertx -c eclipsehono-hono-adapter-http-vertx HONO_HTTP_RECEIVER_LINK_CREDIT=5000 HONO_REGISTRATION_INITIAL_CREDITS=5000 HONO_TENANT_INITIAL_CREDITS=5000
    oc -n hono env dc/hono-adapter-http-vertx -c eclipsehono-hono-service-device-registry HONO_REGISTRY_AMQP_RECEIVER_LINK_CREDIT=5000

On the simulation cluster:

    oc -n simulator env dc/simulator-consumer-influxdb HONO_INITIAL_CREDITS=5000

Revert the changes:

    oc -n enmasse env deployment/qdrouterd -c router LINK_CAPACITY=50
    oc -n hono env dc/hono-adapter-http-vertx -c eclipsehono-hono-adapter-http-vertx HONO_HTTP_RECEIVER_LINK_CREDIT- HONO_REGISTRATION_INITIAL_CREDITS=200 HONO_TENANT_INITIAL_CREDITS-
    oc -n hono env dc/hono-adapter-http-vertx -c eclipsehono-hono-service-device-registry HONO_REGISTRY_AMQP_RECEIVER_LINK_CREDIT-

    oc -n simulator env dc/simulator-consumer-influxdb HONO_INITIAL_CREDITS-

## Max Instances

Setting the "max instance" value to some value. More instances require more CPU as well. But only little more memory.
These settings allow over overcommitting CPU but not memory. For the device registry a good ratio of registry instances vs adapter instances seems
to be 1:3.

Example for `maxInstances=1/1`:

    oc -n hono rollout pause dc/hono-adapter-http-vertx
    
    oc -n hono env dc/hono-adapter-http-vertx -c eclipsehono-hono-adapter-http-vertx HONO_APP_MAX_INSTANCES=1
    oc -n hono set resources dc/hono-adapter-http-vertx -c eclipsehono-hono-adapter-http-vertx --limits=cpu=2,memory=512Mi --requests=cpu=1,memory=512Mi
    
    oc -n hono env dc/hono-adapter-http-vertx -c eclipsehono-hono-service-device-registry HONO_APP_MAX_INSTANCES=1
    oc -n hono set resources dc/hono-adapter-http-vertx -c eclipsehono-hono-service-device-registry --limits=cpu=1,memory=512Mi --requests=cpu=300m,memory=512Mi
    
    oc -n hono rollout resume dc/hono-adapter-http-vertx

Example for `maxInstances=9/3`:

    oc -n hono rollout pause dc/hono-adapter-http-vertx
    
    oc -n hono env dc/hono-adapter-http-vertx -c eclipsehono-hono-adapter-http-vertx HONO_APP_MAX_INSTANCES=9
    oc -n hono set resources dc/hono-adapter-http-vertx -c eclipsehono-hono-adapter-http-vertx --limits=cpu=12,memory=2048Mi --requests=cpu=9,memory=2048Mi
    
    oc -n hono env dc/hono-adapter-http-vertx -c eclipsehono-hono-service-device-registry HONO_APP_MAX_INSTANCES=9
    oc -n hono set resources dc/hono-adapter-http-vertx -c eclipsehono-hono-service-device-registry --limits=cpu=6,memory=1024Mi --requests=cpu=1,memory=1024Mi
    
    oc -n hono rollout resume dc/hono-adapter-http-vertx

Example for `maxInstances=6/2`:

    oc -n hono rollout pause dc/hono-adapter-http-vertx
    
    oc -n hono env dc/hono-adapter-http-vertx -c eclipsehono-hono-adapter-http-vertx HONO_APP_MAX_INSTANCES=6
    oc -n hono set resources dc/hono-adapter-http-vertx -c eclipsehono-hono-adapter-http-vertx --limits=cpu=9,memory=2048Mi --requests=cpu=6,memory=2048Mi
    
    oc -n hono env dc/hono-adapter-http-vertx -c eclipsehono-hono-service-device-registry HONO_APP_MAX_INSTANCES=3
    oc -n hono set resources dc/hono-adapter-http-vertx -c eclipsehono-hono-service-device-registry --limits=cpu=4,memory=1024Mi --requests=cpu=2,memory=1024Mi
    
    oc -n hono rollout resume dc/hono-adapter-http-vertx

Example for `maxInstances=3/1`:

    oc -n hono rollout pause dc/hono-adapter-http-vertx
    
    oc -n hono env dc/hono-adapter-http-vertx -c eclipsehono-hono-adapter-http-vertx HONO_APP_MAX_INSTANCES=3
    oc -n hono set resources dc/hono-adapter-http-vertx -c eclipsehono-hono-adapter-http-vertx --limits=cpu=6,memory=2048Mi --requests=cpu=3,memory=2048Mi
    
    oc -n hono env dc/hono-adapter-http-vertx -c eclipsehono-hono-service-device-registry HONO_APP_MAX_INSTANCES=1
    oc -n hono set resources dc/hono-adapter-http-vertx -c eclipsehono-hono-service-device-registry --limits=cpu=2,memory=1024Mi --requests=cpu=1,memory=1024Mi
    
    oc -n hono rollout resume dc/hono-adapter-http-vertx

## Using Eclipe OpenJ9

On the IoT cluster:

    oc -n hono create -f hono/openj9.yml
    oc -n hono patch bc/fabric8-s2i-java-custom-build --patch '{"spec":{"strategy":{"dockerStrategy":{"from":{"name":"ctron-s2i-java-openj9:2.2"}}}}}'

Revert back:

    oc -n hono patch bc/fabric8-s2i-java-custom-build --patch '{"spec":{"strategy":{"dockerStrategy":{"from":{"name":"fabric8-s2i-java:2.2"}}}}}'

## Switch to events

On the Simulation Cluster:

    oc -n simulator env dc/simulator-http TELEMETRY_MS=0 EVENT_MS=1000

Revert back:

    oc -n simulator env dc/simulator-http TELEMETRY_MS=1000 EVENT_MS=0
