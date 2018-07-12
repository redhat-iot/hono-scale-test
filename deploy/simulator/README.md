
Logged in to the "IoT cluser" execute:

    oc extract -n enmasse secret/external-certs-messaging --to=certs

Then log in to the "Simulation Cluster" and execute:

    oc create  configmap simulator-config --from-file=server-cert.pem=certs/ca.crt

On the Simulation Cluster master execute:

    oc create -f pv-influxdb.yml

The deploy the simulator:

    oc process -f template.yml | oc create -f -

**Note:** By default the simulators (HTTP and MQTT) will have zero (0) replicas.
You will need to scale them up in order to generate some load.