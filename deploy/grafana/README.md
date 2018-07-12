# Grafana Dashboard for Simulator

    oc new-project grafana
    oc create configmap grafana-provisioning-datasources --from-file=config/provisioning/datasources
    oc create configmap grafana-provisioning-dashboards --from-file=config/provisioning/dashboards
    oc create configmap grafana-dashboards --from-file=config/dashboards
    oc process -f template.yml -p ADMIN_PASSWORD=admin | oc create -f -