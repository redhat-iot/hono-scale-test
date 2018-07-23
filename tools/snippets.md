# Snippets

Some snippets of code which helped during the test.

## Show qdrouter statistics

~~~bash
for i in $(oc get pod -l name=qdrouterd -o 'jsonpath={..metadata.name}') ; do echo $i; oc rsh -c router $i qdstat -n -b  amqps://messaging.enmasse.svc:5671  --ssl-key=/etc/enmasse-certs/tls.key --ssl-certificate=/etc/enmasse-certs/tls.crt ; done
~~~

~~~bash
for i in $(oc get pod -l name=qdrouterd -o 'jsonpath={..metadata.name}') ; do echo $i; oc rsh -c router $i qdstat -a -b  amqps://messaging.enmasse.svc:5671  --ssl-key=/etc/enmasse-certs/tls.key --ssl-certificate=/etc/enmasse-certs/tls.crt ; done
~~~

## Locate pods

Show which pods are running where:

~~~bash
oc get pods  -o 'jsonpath={range .items[*]}{.metadata.name}{"  "}{.spec.nodeName}{"\n"}{end}}' | sort
~~~