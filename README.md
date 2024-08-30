# Operator Installation

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-adp
spec: {}
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  annotations:
    olm.providedAPIs: Backup.v1.velero.io,BackupRepository.v1.velero.io,BackupStorageLocation.v1.velero.io,CloudStorage.v1alpha1.oadp.openshift.io,DataDownload.v2alpha1.velero.io,DataProtectionApplication.v1alpha1.oadp.openshift.io,DataUpload.v2alpha1.velero.io,DeleteBackupRequest.v1.velero.io,DownloadRequest.v1.velero.io,PodVolumeBackup.v1.velero.io,PodVolumeRestore.v1.velero.io,Restore.v1.velero.io,Schedule.v1.velero.io,ServerStatusRequest.v1.velero.io,VolumeSnapshotLocation.v1.velero.io  
  generateName: openshift-adp-
  namespace: openshift-adp
spec:
  targetNamespaces:
  - openshift-adp
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  labels:
    operators.coreos.com/redhat-oadp-operator.openshift-adp: ""
  name: redhat-oadp-operator
  namespace: openshift-adp
spec:
  channel: stable-1.3
  installPlanApproval: Automatic
  name: redhat-oadp-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  startingCSV: oadp-operator.v1.3.3
```

# OADP configuration for OpenShift

Cloud Credentials secret in namespace OADP

Secret for S3 compatible storage access (AWS)
```console
AWS_CLIENT_ID=TODO         # <<--- HERE YOUR AWS CLIENT ID
AWS_CLIENT_SECRET=TODO     # <<--- HERE YOUR AWS CLIENT SECRET
cat <<EOF > /tmp/credentials-oadp
[default]
aws_access_key_id=$AWS_CLIENT_ID
aws_secret_access_key=$AWS_CLIENT_SECRET
EOF
oc create secret generic cloud-credentials -n openshift-adp --from-file cloud=/tmp/credentials-oadp 
```

CloudStorage
```yaml
apiVersion: oadp.openshift.io/v1alpha1
kind: CloudStorage
metadata:
  name: rosa-csa-oadp
  namespace: openshift-adp
spec:
  creationSecret:
    key: cloud
    name: cloud-credentials
  enableSharedConfig: true
  name: rosa-csa-oadp
  provider: aws
  region: us-east-2
```


DataProtectionApplication

```yaml
apiVersion: oadp.openshift.io/v1alpha1
kind: DataProtectionApplication
metadata:
  name: rosa-csa-dpa
  namespace: openshift-adp
spec:
  backupImages: true
  backupLocations:
    - velero:
        credential:
          key: cloud
          name: cloud-credentials
        default: true
        objectStorage:
          bucket: rosa-csa-oadp
          prefix: velero
        provider: aws
  configuration:
    nodeAgent:
      enable: true
      uploaderType: kopia
    velero:
      defaultPlugins:
        - openshift
        - aws
        - csi
      defaultSnapshotMoveData: true
      featureFlags:
        - EnableCSI
```

# Observability 

## Scrape Prometheus metrics from Velero

See [GitHub repository README.md file about OADP Monitoring](https://github.com/openshift/oadp-operator/blob/master/docs/oadp_monitoring.md) for more information. ROSA comes with the User Workload Monitoring flag enabled, therefore it may only be needed to create the ServiceMonitor resource for Velero's port 8085 as per the previously referenced documentation: 

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    app: oadp-service-monitor
  name: oadp-service-monitor
  namespace: openshift-adp
spec:
  endpoints:
  - interval: 30s
    path: /metrics
    targetPort: 8085
    scheme: http
  selector:
    matchLabels:
      app.kubernetes.io/name: "velero"
```

## Custom Grafana Dashboard

We have used a custom Grafana dashboard to explore metrics in a more convenient way. The dashboard definition is located [here](grafana/oadp-dashboard.json). 

Firstly, we need to install the operator in the newly created namespace `grafana-oadp`. 

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: grafana-oadp
spec: {}
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  annotations:
    olm.providedAPIs: Grafana.v1beta1.grafana.integreatly.org,GrafanaAlertRuleGroup.v1beta1.grafana.integreatly.org,GrafanaContactPoint.v1beta1.grafana.integreatly.org,GrafanaDashboard.v1beta1.grafana.integreatly.org,GrafanaDatasource.v1beta1.grafana.integreatly.org,GrafanaFolder.v1beta1.grafana.integreatly.org,GrafanaNotificationPolicy.v1beta1.grafana.integreatly.org    
  generateName: grafana-oadp-
  namespace: grafana-oadp   
spec:
  targetNamespaces:
  - grafana-oadp
  upgradeStrategy: Default
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: grafana-operator
  namespace: grafana-oadp
spec:
  channel: v5
  installPlanApproval: Automatic
  name: grafana-operator
  source: community-operators
  sourceNamespace: openshift-marketplace
  startingCSV: grafana-operator.v5.12.0
```

After the operator is up and running, we need to create a grafana instance and give permission to the associated service account

```yaml
apiVersion: grafana.integreatly.org/v1beta1
kind: Grafana
metadata:
  name: grafana-oadp-instance
  labels:
    dashboards: grafana-oadp-instance
    folders: grafana-oadp-instance
  namespace: grafana-oadp
spec:
  config:
    auth:
      disable_login_form: 'false'
    log:
      mode: console
    security:
      admin_password: start
      admin_user: root
  route:
    spec:
      tls:
        termination: edge
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-monitoring-view-grafana
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-monitoring-view
subjects:
- kind: ServiceAccount
  name: grafana-oadp-instance-sa
  namespace: grafana-oadp
```

In a console terminal, we create a dashboard instance and set the Bearer token retrieved from the associated Grafana Service Account as Authorization Header to Prometheus

```console
cat <<EOF | oc create -f - 
apiVersion: grafana.integreatly.org/v1beta1
kind: GrafanaDatasource
metadata:
  name: prometheus-ds
  namespace: grafana-oadp
spec:
  datasource:
    access: proxy
    editable: true
    isDefault: true
    jsonData:
      httpHeaderName1: Authorization
      timeInterval: 5s
      tlsSkipVerify: true
    name: Prometheus
    secureJsonData:
      httpHeaderValue1: Bearer $(oc create token grafana-oadp-instance-sa --duration=8760h -n grafana-oadp)
    type: prometheus
    url: 'https://thanos-querier.openshift-monitoring.svc.cluster.local:9091'
  instanceSelector:
    matchLabels:
      dashboards: grafana-oadp-instance
EOF
```

Finally, we create the dashboard from the JSON specification by compressing it with gzip and transfoming the compressed output to base 64. 

```yaml
apiVersion: grafana.integreatly.org/v1beta1
kind: GrafanaDashboard
metadata:
  name: dashboard-main
  namespace: grafana-oadp
spec:
  datasources:
    - datasourceName: Prometheus
      inputName: DS_PROMETHEUS
  gzipJson: H4sIAAAAAAAAA+1de2/bxrL/35+CUA8O7MJJLTu2k+C2QOye3hZIUt/YSXBPHQg0uZJ5QpHskvIjge9nv/vgm7MkJVESSY2AxhV3tM/Zmd/szA6/72jaYDSyHG8W+IPX2l/su6Z9F/+yEkefEvZ08Ovl6OLDn+/+dfX7vz5eDvajYlu/ITYv96g7JcEtmflJoUl8g1peYLkOJ0kKgkdPVGrqge67M2qQpMyzZxPL+cNUVSrL34fdukgoBMET+/fLvhwSscmUOGJQ35/CZ5T8PbMoAQYa9cnTHTaiuDlLdGSizyapPkaT8t/Zx3eE+tFYZW/24TYmVB/rjl5oJf84bidfkGppePD8xfOj8vagebaUM+wU5xZs+PnB84PydsG59AM9KDZ2mXlaeybBFgL9xgZW6yr7eMk2rCnxCbUIMHFXrEzLFxabi1lVdxyXDZ+VCl4VpQPb8oOYS5NesZKbmWUHf/CahvvJ09Qav05Rl3OBKJ9luS8uekrVThwxea+1gM5I6vmtZQJPLcN1zl3bpbxeOrnRdw/2tcPhkP1zfLyvDfdSHYhn7U0yC9o/tTc2oYGfpgt0OiFBfnC2NbX4w+HBQWZQUz0wbt84j6xorNs+yRQG+kRIgC/78Dz5tzeuTk14KpRkT+Lvl52QHBR/A2JaQW4qB2PLN3T7f4lO2S6gwTvXCW5ZuRjQYOKQQEjD4emLgxP5iOre7ZXr2oHlxXRiEZ2ZbYtvtuV8TYbIvt6R9+59ejIkWwOC0HBtW/d8YubWVcVhyT4pyouEu/7xPfn5Uzxjcd0TapkXrp+t+DbH4rz/hy9SDx6i0Yff+WofFOoWHRieJBokO/D04MvGWT3W7HgzGvNpkKJ6yrDd2CK2ee46Y2tSbM8kY31mB36hRC6U2GH5As79rtiWg+CW6bpb1zb9QY7oaT9f3VT3PMsBNkZY+lDcZbLEcrLLEE1V0nZpH/Ub37VnARnka9C4riBedqWST7FK8ZNoVhhLEeLkhw0OvU5tLtWdCdRHQXan2zM+mNPjptqjbP9VNPbqAGqs8OxL5crPHCFCBx5hPO8E2TnLkQ9cpsjYZpUQ6ouSq6ENLQr4pj7O1ck39lHu2UORqcT+HiqbFBvvKKsGXC+rVuMCxrWfrG+/E2tyG4iF24fKP1umEMWFYpcpd0dqK8HDs8DNCwK2gjOD/KnoAF9r3TZg5maw2g/eu8F7Js7zHFzcmkKA+BmEHZcJXvEjuV+6sGyv3l9Fe/Ytx/V+QbMXCd/p9CtDNgpK6xuTKPEEKVdOYvpPBVCbU9pc/xcnrDCvJfJbVlQlwwVVDTkOzGKo4V36LpRuBv9bpHkgU8/WKYROQgKPyok40K6dH9l//my6S/WA7DJsFeiWQ+jI8Gajma9PyMgn7KnpjwKGoOzvHFH5nm6Qn//vevCP+Nv14Omv4+mXvb1r5ydZ39fZDRl5rjlK6mTLKqZuJICVL9owXPa0pFZWYXGIlsOAvhOAjCGKA0IZd8Jsa5MJcczfXMpgHKcYjYAdJvbYWFqKb3JCK/UtB/KsQICvwTkflPaR4Rxtd8w4QZMD3ssxXWSxCUsPwoMIHRA6lNeG0CEkXid0yD+rAx0yv0HogNBBy306BR0C1/u6O9y/djQue/mfH/k/TO9rN4/aLlP7e9ouf6JpzeAKXpf45ydVOw3hjbCh1cOOVQGOK9fTLlwEHpmiSKl7uk2CgDwzmCTzLaN61xgzP3CncKX6g+WfudQk9PI2e/4FEJ4Tzh3E/DehbhUpH0e0UQPyEEBygBO+jRwkKoILmzE191MolIEgvNHpG9uaOCFdATHxWaf6/WXwKHnNZnsLqmds2fafbBdZgdCHUD0TqptcQUWDc1y4Kn7y+xtVzHu8l9TTyBkzPsJU09xZ30AVxD8AOuL7n1Ch/fySJeTz8wdfbM+1Y03MHzK5qiCP9PkQKPdcJlcurW+kCFBEOT/eJb9afkCtm1nYHDhp0VYNu1JrxFynXvAO+GUcxGSnUzUrTHIaX6XWBXs3oe7Mk8IPXKtpmmVq9T0B+hHrgg1HFbvjMVDvsubJVhghNYyClwsaBRvB/MOXAOg/yT2rAfqHmf2sBv2xOFNj8SJrmZbPoNpjJEqFS69A5KUVABVGRYGG7/G3GYFaOumJYC30NuJan20IG4CXvksDeA8/KSdyK+F1Ed82haL3ir0K3axwj+bDtN+/sx4/PQHQVpxSKHDzMrj34qMWYV82G3zSGPaF0W7Kq75NkBfP2ppSc3jWFhJv2k13Uq52T/GsLV2GZ23rBwNzuODmcJi9npKpSx9f05njsEkt95r9KGtNKpM/Ht27lNthDDgEo5vHgPivGVlZVV06B3snxoieNy33QTSAaCD+dBgNQJ63CjRwdIhoIFWGaKC9aKDoVQO8XQ3DBNBztzxoAE8aWo8b0IOGHrQsAXrQSmnQg1Y1YvSgVbSDHrTVetCU4F7o8c1A+9oOtgpsP6yJ7dHBhg62kABysFXD3XnPx1rtNIsPyTrsMcvdf7zgliIT506gXTIu0XMHM3ik1gktiEdqIfGmHWzDisCWV5nK8UwNz9S03Kc1Z2p5D5vNVPodk4VTMuLJIvyRoUuLu1rRxz41sJ6ZT8zFwEJbD8VCTZr2pkWT1anjMMQK4jlihWXaQ6wAPKuFFTJ2PWIFxApa7tMarFDlf4sVh1T8hq1b09AjtxSsyPvg5mmnHuzomDsugzyEO+7TudZlCIIeOfTIoUcuLkePnChHj1yepj32BXrkFFfequD+aT24jy65Hrjk6gJr/W6yKmTbgPcN6hToj1sc0BbPnCS25d63T+ed87515Iisc4pqGd1hOWfehpRHbbdS1cWtmm4lI21MAHhbJrEtI/gPM4GscaxIoNOinh4oXcgzxfPbMLgAENzCPiubmnvGQ2/1R3cWnR0ol7Vf50gNR6DkUy6O/REluulLpVcz32IXj3WubplhNLn1ZoH2gQ0YVn8ilTsqvkIf5G9Q8bVV8YE+kirF9yLzDBVfRIyKTxD2X/HdU4vrvHk0X7cy9AG67zMfMyo/dGIojqA54aU7Dt6FoR6HLyD/Q0wllP4zFRl6RDrjETkCytEjouh7Gzwi7cXVnfF8bBDE13d9VN1GeoWuj0xLfXZ9rO420mpOhRrM8rfL+7OnLXlv6ax0t2/RelfYQTzTwojzxHFpwgXtR4b9Vrvsom9Lr/v54qZTbDkJDtT+qYU96ugdNjxMxMPE8Nk8XrTDigiM05oB13iYmCrDw8SkopYEjWSzFzok4LeyGRYyiHVHFtCQXTotfC9Hq53pjnnPjw20D3LceFyI6i3/pLvqDfKVVaq3mgGGqN5SZajekopard4CZkT5UyvYOv12FQ4cFRz6w2r5ww6LV5+1gjcMJkJfGPrCVCNGX1hFO+gLkzTb7QurQunDmhFt6AtDX1hIUMMXVnIItFCg9JxeL9Ho0g6QPFreb/fiLuvUasCyWYNvK+pd2eqqPJeLmDsRN2n/1JKm5/Zj7eRaTVr8kymIO4vcD5KisCLKLICd1FpHLMWVnq17PjFzHKziwHLOSzgu+XnMbjGbgRpqkAOGQjMdphRK3jWSeRdjUrfowGFcEbOpHJlCJRGnfTUf5zkfLbUcuZkkpLqwCiHJYhBuBKlOtwRJNHn5nDOKHkm7wyNGoLBV+n/GuwhWrZcRC4arK8LGw4OVv/Y190toFfXAuCXwhomkxODm8T3TOLAhloDXwSc+MO2HW52azyj5e0b8oLDDYI5mgsNjprQF9hrueap7IeJV9jE154PfWee0D8rOLWScrGda+Y3v1k4rfyNI92f1GeUHNG2e2zDNE99ie8vKRtkTYVhXdQHMtbdoi1KlPk+rxor24epSOuhG58fAcC0pFZtLf5UZRiND43knVyX1N7M/hDCXb/lp274QovytomstnlIhcto5pVzUdHtG2yvC5byiAM8QbJMA31GVznEs/SLbOXksfZh7qIyNVLYp3wVZ71CaL3uciHbgT/Om/th1A3B7M+No5gQf3HulX6g0SkOGilBwq/LzteoYED/l3y21oTjh70Q3RWtQoliXBmeP9Qwsnl9bceqpZYUIG/X5jFKeBkkFpevyUL+CRlZxgh4foe5puzy9rHhxZPSqyL9nbqCrz1z3NT7in68HXIrzr9Hv2KPIlHhueLPrwROYWnYcn6wG+g3kD6k4u11P3EqJGd+3U/k0MyzECByAlDNCZ9mg5NihL2yQegtGM+xQLRfi12Ug/83DfxC67wsXrpL7pG3UdVZQnkUgC1TLg36wgPLspC8ssA5dlGWFlWui3nEeqIVS31RxDtF0aecXH7Vzl02R9j98WrXdZCrzL27g4Q9ygkJrvMJWC48xCAWcnWmbvtSJqKjVpRPd4ZGnZRUXNw15MOyZSc4iKxc89ruypqrgIFE+GvEpGo3KaOKwmjIitt4ihLaMRvKUUdqh/7g3ZcXxgpYReW4+nCRTHO2hOjRif5YR+jzkpbyuUAbmrqPIDxyHYJKHZFkhEkr4RKRoSg80SjZRFOgjJERcsgV3HTBYJVOKwSqbCVZp/PVt877OIVcPhrJgKEtFz1Pdw1AWZd82NrfoCc0QbJMndDP7A0NZMJQl7lsHZrS9IhxDWQACFOCZNprcG8lhSss2w3t1x5bgRmmqV7VtkzG8CaqXb0dVuopIpMLDGqFI2Zu2GIuUjkVSzly/gn7a4DaNfehTMnXpY5fdFxjN00w0Rfc5AQN6ViUb1hfT0zMuxLCepRz53ecGjOxZPr6i+1yAwT2rkAnriu/pFf8tHeLzTswGxvdERBjfUyzG+J581zC+Jz1bhW3Y1eSChWdLpDRZ8Lgyn0xbZE0CTiuhe5MVb7bKVIMpt2NiTLktCLsCTnnKbQEJmdYaJRnqIgU0ioz114Y3e01njsMkUV8ybstg7DAyheA711E1KSg2qZogP1qFaqrrRkPVlJShakoq6ohqkjb74oqpFacPi2qtMMgJVRaqLJBiXSoLeoVRPkikhsoaruQNRjolOuosDXVWSNWEzgLVAfwCI6aZRjNfn5CRT9hTs7l3FzWR33sFSkn76KMZhTpJRbFJnXSae1ZDJ9V8XQeqpFQZqqSkom7lR6thagmFZnBJX6a3rp1nwvOcf1GJQhtWvqTkuhgU3eRrLgA1uNArS+rqyDd3umUX3OGoJ5fXk477KYqSPyiu6qa1qHgXIKREi+1UV/bDv96cvTx6VXVb4OD5cYfUNniNYAHP3FHmLV41FbecMjwARc3dL80d+e82o7u1X7SDBfR3u49eP+voLUQzt3yoi+Y7WbU6XcCbuIg2RW8iKtOooo54E+NAFxnRvFUuxTBsOc5/g6oNVZtyqG1VbQt4HV+hatNQtUWf3qq2MFBmexUbBsugWqsaagvUGuS3HJ7Mr9YyP0HHZUyMak0QdkmtJdpMaq8Rf2c8k0gjnwQjsVV5NE2jumzTygpjaFBVlQ+1rarqVe5hDVWFYZ/8g6pKfjqtqhq1wGJH3BYqQIyQkUUYIaMYZZ3KakbIHB++OHz58qCpjKhtUM3gJcIF/H5Hi5iRGEaDyjmuqCPKeQHPX4PqmYfI9O28FSNkVqK+eW5vJssPii9CausLkh6Jbbv3jSl1aUVXqPRTMOZ1keZqvCBpCLe2BHIAE92vAjvkD5ZVZj0UgfuqAjrUdKwytv1kfYtTeZ8e70Plny0zuIWKe4oJrqJN+1a/IbYPivks4TudfmVKXUFpfWMiJZ4gBAchQSE5Y6N5cKL8i2u5Dto2GCDjZPlBdvi2kGj28nkQ4bdibAEo8HSbBAF5ZjAhw18cUsnQpS95fLD8M5eahF6m3i8AbCdOeC5eBEnMfxPqVpGmLTthnSsIhaACuSgkuLAZF0+JYFZITseEl+44eKcDeW8ExY1O3/C3ZIQ1FfAOXxeq318Gj5IPbatgawqiscVf/KIbVvCoqIaZrCZXLiVWq6DjOP03qliYeJup55lzruvageWV0txZ30D1wT8AtOGigVChufySNebT8wfnBs+1Yy3KH+rgi0J5SaSLh0C5SMB5yTOJFoCEKPeZriW/Wn5ArZtZ2Bw4adFeDrtSa8TCmOYd8MtYjElTp2pWmFA1vkqNCfZuQt2ZJ+UiuFbTNMvU6nuC0iPOBRuept0bxYqXNS7QhEjdm1uvEaFobnkrQnzdiCWR9wU+ANZFDUviRc00m7GcVQP8IsuH73yKZLzNRFMRBnlp1XXjBlwLF4i49HkbdaGQaTc/54nEL/Q22k0+26hQxmzfpQEsXJ6UE9k7zN4krL6mHP7T5g2AlbrdZvkYEUEVI/azlSP2wGJDJdRKOzlaAtuZLDOo5UV4Jtt/xPTrxfSrQez5W+GCBCE7Qva2QnZ3PG4esW8amatj2laEk18u6KVvDQAGr99WIeCaaaj6hoCnTBNbCIBzBOtJM9ggFM0fHu+3ewU2kZqjFRYFhe7+Csp66bDyJkfL17kjOajnWUG7eMlN0MVLdL6EVbgfXw7fL71N12K7sEsxHqcdivFYOww8ORguhgOrTkvbE3NxmDPW6gDF05qvfcCgCwy66HvQxSpVdnvDLqRi/inS1NoH8TZLUE9vX8QFhmFiGGaXIQEUhjk8VrYpz44yP0JIgJBAy326CwnmuaxRCMVs7i5le9FAKp0ARmFmi9Bji1GYORJ06RZmBF26ZRVjFKZihHWqwyjM9RgSdaMwKw2JmnfB++aExjDMlefnWhXq32T85RrA+pY62hCxrwuPY4wlAvIMTdsBOcZYbjLGUomB15QJaakQzEr4WzNNYd/gb0UMJr9pgei38QNljL5ca/RlS0wKDMDMEbTl3QYbjsFMWYYYhLneCI/j7kR4rB0nHm5pEGYVVDzF1FfpMgy5aK9qXUfIxYpVd3ujLzAWsyFNXXqAy489ha0gDmehbWQQfqipkoGCJJq8vIxQ9EjMjc2aEynK1QdmjDc9YgSg7OuPY7gzB1T1YUXul9Ck6oFxS2Amj4DA4ObxPRNe8GFoAgsGsWCsx3pshzOgFFhgV+HupvoUnjUpO5aa58G5umPzra5sWu7j53LLVrVtkzGcCn5+fLni5WO6rW0LdwF1CZcsXjLxDgTth/jApWXL90HZryXW8F4YElWitaH4l/Wsnzj6bNnafQT7hOuWX7dnPMThWVt3IBiF0ciqirOOquaVUWOLthryUhoKV/QBri4FJ21DBdTUhk1mEI0MLAV+lx1QCT4OKctQcllfw5/HGPeG90hJCU7CPNWXhVaGtBUBlg11RI3lc91QBF6G3VCU5IWZmnp+tuLHv1XHn400ZBLDmjLjcKHWWi3fdWZQ6dDm36xkfyO7pX3k6XK03aHZkFyfR1sPXzQUYrGe5QxdWi1byLeKXiHUgqHW+lYRIU40iO2FOMpDQflZGlqEt1G6BHIOnr9S96KdGAd08K4X48zf2mJmnhSPMBhoq3CXfY6N6GdUeJhaqafzbrBGlhoVTTQIVDSKz/rE+4m6D+0U7qddEnWMN90gsHksR8vk21VJz1Yt1FCgoUCbt/7aAu2wc0dyQ/gl3ngol3Z2Bq57a01u2yZGY3zI+qf9DnZw5cdyR8smG1iZPL3R+VXXSokqNvmzG934yu/iOaAjcquFa0B1x/d0qtCd8rO0iHXlfY2apwIHYBx12JXlBW0H5BGYV6RF4ujt8nlPUBqhNIp+jtKo5LdZigXvd5xmByvudxy+yD1UpdQ8HCobFbz2suZVYL7j4tsdAz9/W3cwdt0AFKls6WZO8MG9V17GL71wIW99UJA/+WW66uscfioXRqYMurDxO9FN0Rp0+8Klwdkj0BNAxYirxHAUv5aV3Yooq7qcs51XPdjCazePWnJheV/zXHNPW/H7Scbx3YpABMsXCCquXq7ncgcchtb2C7QN35WG2ePa0TT+CqHqtw2VMMZfx9Mve9dOZ/mjGGi6TbwheKCUQZpjEU3jfCIupdVpto7wEp0x+HtsSpqPWu4yh5aF1G4Tv5byjH432W2GWfc0Xt+Pmutou6z+fS2m2NNEKqQRvxCxK8liRmXsEMz8kXer++S7+Pfn68EHqVqvB6lKABbd0zrNnmBEYNsZc6X4atkXkXRj8aHIs94ve1a8GGMmVGJH4evQfdfHxVY5atu+4AiYQmnUH7g0GoHZDiHQ1EURtRbI1Jz2ihJN4ClEjiKnJ0vD+baJJ3evablAo03bgKJChtfF32dazfYbkuSycf5H/KM7Jv+zsUn4UYRNh3Pwy3bPxS9sLoZxb1otXip1XmkYCIqXNbA0Ey0bYuRFN9QKpuCXdU8BE2j8lmnbREskX3spYApxHXVeOH7xUZMvHWcLkSSgyRJzv7mck9D1WuGkCyMICQX84GkHbmmKIEWtLp3oDs/tXlZxUQiSB8OemeQscmmCcTVXligrJHIGeid75JjkIakSIqGEs2GKJi916y5YlGlUMGRcgqnLstVh6jJRiqnLWpJbP1fPikP9MLMZ0HZ30mRhZrPOLRlmNlM03b20DW1bO8xslqfAzGbpXuBt7GgQjQyskxHtmNks1w3MbNarS5SY2ay8E5jZTNEvzGyGmc2UvUKI0yGIg5nNgG5gZrPS1jCzGWY2m6sbqGiiQaCiUXzWmAioa8L98HmnkCxm5sFcGOGItlfatS8XBmblwaw8hS6gJJpnlCiJlK10SBLtqErXlpHnpbJRmZHnZeYZZuTBjDxdvjrc8JuYWx1OXiOQvFtXVBq8OC6XeHTv0q9siUc+CUYi3PM1+20fF7x7WXZasNgNXKvtvyjpenacxflMsoF+NxnxMPBRYE3J8kz319D8sicy34jEN6q8N7s1k948/fzzcK/bvIWpbRTJAfovW7qYOqLTemtrOKvb6UnaJV2Wh0l95LAeZBxpm7XNL3CL+/2rlJ3JxfV1DSvKB/KL+o58N8clrt53df92OplHG0BI5WZdgXRY/87llxC0/8KdKwnatnMXyZLxTkwTpsjAFBkRRb9TZIA5Z7QVJchoNvcF5rgQjRWelbuZW5TEQn1nvflgmLmyEoR5YxqY7QWd+seAU394WNOpnwmkgZz6mdrRqZ926itnDj3pIeSUYxwxFGvGX7hWj3DoHd+jhm4zPMoVg8/+l+HR0E9BieFOpwzKCfCU4NXsc38k61WD1H0tArg/Xw8Mb7bgUdG6MOmbJZK1fYimhph8KT5dvGkEj3YcOZY0W1wIUfqpkCkFJEsSK71OJzMCaVNbgVN/Zrad7eqwGh+ktkqaWBMjKk4TIuN0GSLj/KgRGWsNIOPCzHUj+xsC56WBc+FhLeR8gsg5RYjIuePIWZ4J9w48h2e4iJw1RM6InHufdhmxs7o2xM6yvZIkR4bONKMVPGKaI8w+0YfsEza3O57NcwsSkx3lerRcsqP5ReeKRZ7HbS/Gvk5wx3h8Sgxbtwq24MbPDpZIBb6jKl3yHm1tl1smOSNwcPAqUxEeHGzqHm2YuCtS+YXUXXivtuxMQ7+biDMNUJ6Epxg2s9TkU3Fy4Y+iuZbhbg3Heq/rDALGiH2JA11oXTmAXsmatiKKsL83Zle42OIqSPuERGsYqsze3Bb2yl6YXZzZxE3ZbnNDr6+2LiQG9Dvdsvmy9VcOxEOc26txEc+kdhm4gnOacGd0OjAdtHxHwlUwGikOhFfqSQlPF5NV5g6s+IvavaL9EKsFfkYpNoV2Hj0q+51AKqF5U0kYy5zX4FsNKn+eVWAqowqsBj6EyPJ1NG7F+USpsYaenaQoOtT0dCZiA/KMzbXPU9hVivdS986D5Z+5lBnulylbH9gnnPBcuICI+W9C3SpSPo7o1dMBeYAOyAXhW52pDFC8hgQXNlMOUyKkuDJ8ixNeuuPgnS7CQACKG52+4SdWYU2Fl2vydaH6/WXwKPe3bTmgF2hs8fP2aKtA1UyoblqslWj0jgvXxDXXb1SxMLH+Uc8z51zXtQPLK6W5s77NKTN9QoP3M9tWnjppcnr+4NzgubZQTdGc6aCLkJd8Fgkj4dXxXKaLL7lqea0dQb43HjbxKxMo1LqZhc2VOhrDrtQaMT+8uuAd8MtYjIEUp2pWGNow+E1AVe9EvhchpmFxOk2zTK2+J86SiHPBhmOHCCV6DRlcECH4vtpaLpgq58uqvL2wk2UZd6/SXVnq9F31KT0Y31d5TP+q3jF9LG+LSlm3DQXrh8fRkay3mYgqomMvrcJu3IBrY/Cs/W3UhUKsUH7aE8lf6G20q3y2YSGLiZ/Fw0LmSTmReOrd9gOzCjt4HkP3+3ew109PjcbzFS1frdLuSFA/sxYZTLHSISlRm3F7qYs3YW3aO91hDYmtmFCFddLoYnnIVhH3ckls6560yNL2qYrZy5k8Ye7k5zFnxxwNCsxBDj4V0gLnXZkyHXChbtGBozgLMLMoHGJnNzcaT1Gd/TeeVmMaDdE2QtsoTdN228gdjxcxjeKXmjEkinYTUFsrciBsIg//UbldclwzfKiGXQKwiAXFZimi3wZTooPkTEkWorcqbSCFoyZjBFERDVVtA4HGiwgkGrzVIVOLl/+aBBChDbWADWVaQQI/DP53cTvLn013qR6ksygb3oxZUgyGj3zCnpr+KHAD3S5zCx9PQbfwvGZN3IenJ+0njZk5rgkaNeKER8GAjVz/l96ZknRUZWYOeji2EaSjhwNRvJwRRPFlFS/q4UjB+MNCIcJ4dH+AP9m8kTGs8H68bM77gVZGuhytDK2PVobMfF8jmT5roEbe3V4bLaGvRk7IPJ4aNGHQhBkeQKYFGjFoxChGjEYMGjFlo69TWw1j4hC0XJa3JdpuR5xW2BHorUA7Quu5HZE1H4wxMxpuqRsETE1NXof7uPdg/7dL7SoedQbv+3NA/J1cQ9lG3rkOXzaugxOCnsdhHRXqFh04iQUrxmHBHZF19t8+wjgsNH5UI94i42f5OCy0faDaupA1cBOWz0m54XNc88UqaPikKdDw6ZThwx0o1a81nO9GSsdsnzC/NIZmIW5PCDA0C4E9AntV39Gr0Q5kj6FZwE82b1gMKy6AvMr8CC2LNloWU4aVLDQsNmhYyIAsVk11TFZU/xYbKwuHZO3k2iu0tY0umxeFuqXLJn5jGLps4I7IOvtv+qHLBi071Yi3yLJbv8umXwbcRlwzlnPmrcE38zJX51wmVMWrO19lNjVaUG20oNA30/zlljF/dahu+tLSWfj+fKM5wxq1iM7mtog+sPmIE4ldMYU4ufVmATpych9E84jmARpE81UjRjRf0Q6ieUTz4cNFHSLHJ5lnCOcRzkvC/sP5e2pxJN9XPH8+N57/zCcEAX1YhoA+T4CAHgF9MiMI6BHQ9wvQW27r4fzhUNmoUIynGN+EcF7bSjgvT+f7h+OXPJf/48+LS0TwuQ8ieETwAA0i+KoRI4KvaAdBuqTZapD+StmoBOmZbY0gHUF6SNh/kB6eubcNpf+0doyePWvvPkhn4taglhfBm2z/EcEjgkcEH5cjghflbUXwS5zBO+6nEPey/9WSV7pqvhT0Rf2CBoFobAmDYE35jpayCI4qMr0eZy49o0WAFkFI2CeLYIXv/16xpVD/3dwrshYUL+5GK0GUoZWQJ9iglYBGAhoJaZq2GwmbyJAEJy1TZk7adqsCcy0BP9m8UfOi6qZwpiI0atCoCQk3bdTMZ7Cswl4RCZYWqd7QJdDqkEnUuNUT5luKpqKZjEtRA9uYcum4ULdMuRT/DlMuwR2RdaKluJiliP4kNBUzNG03FfFOh3qUdWrr8yXtpbLWviw3pF5iyiU0pLQWGlIrjhdzSMDT1o4oMYh1R5a8qT23hbPe9LIf5CC193LQ2pnumPdc6XOf2YXbPV8PgnQE6QjS43IE6aIcQTqC9LjCToH0YYW74xjdHYjSte1F6QFDxv7UCvoN06/CUSJOR5xeIECcjjg9mRHE6YjT+4XTvdaD9MPDCpD+CkF6jgZBurY9ID06Sme44Subl76i9Pxh+oUcLkL0pAghOkL0Ig1C9KoRI0SvaAchOkJ0rQyiV2RHOsEUpgjRte2F6PE5es8xeuEkHUG6lvsgSEeQDtAgSK8aMYL0inYQpCNI10pA+tFJBUjHhEUI0rXtBen5c3STup5HzL5iddV5+q9y2AjZkyKE7AjZizQI2atGjJC9oh2E7AjZtRLI/qLiTb8nmWQ/CNkRsoeEWwLZC+fqPcfsyvP1hUH7Tq69pK2oiXrZc3bCOvjouF6KJ5AhjFsy1T/xvD8CXRyJcD3GyinFxnAq54UgreEZ0vCDFKcnXMxgNaVSCmQsDZ/YxAgIiOwkElZzfKxlBmPz3jw++vvh5NvL/3wloH0TpV1KJ92xHMOemeSNbRebZ0IwhNq/JvsyVTxlNokF/IxzLP+VCf4qkfZpdDD4e0boo3KkqfUZZp5OyEOO6Qf+V8v7SO3LR8cAOhfxQKpzO7mZgpcsPZFKOVUloaplU6YZMrYYF0domq/GSCy4L1Jyjdg2HlnO2N2PZURaNiy+3O+j6uZZbQf6UcViZ6Yu5oB5B5oWTpeB7pg6NT/p1OLa+H9EpeDsNsZSoUocAlz2d7p5KbU4IHuS0sOaJvwzGEvTjynW+2cnt2HrTFOHzwaZn3kWE6A0Zkvx7BvXyLH8igXiGZO1M0+bxvJQM3X/9sZlkyQp72IZdyy+3xPylU2jVPODnaed/wcSYYCtgxEDAA==
  instanceSelector:
    matchLabels:
      dashboards: grafana-oadp-instance
```

**NOTE**: For getting the Base 64 encoding, simply execute on the JSON dashboard specification, the command `cat dashboard-file.json | gzip | base64 -w0`.


# Petclinic installation for benchmarking

```console
export NAMESPACE_PETCLINIC=petclinic-benchmark${FACTOR}gb
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: $NAMESPACE_PETCLINIC
spec: {}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
  labels:
    app: spring-petclinic
    app.kubernetes.io/component: web
    app.kubernetes.io/instance: spring-petclinic
    app.kubernetes.io/name: spring-petclinic
    app.kubernetes.io/part-of: spring-petclinic
    app.openshift.io/runtime: java
  name: spring-petclinic
  namespace: $NAMESPACE_PETCLINIC
spec:
  replicas: 1
  selector:
    matchLabels:
      app: spring-petclinic
  template:
    metadata:
      labels:
        app: spring-petclinic
    spec:
      containers:
      - name: spring-petclinic
        imagePullPolicy: Always
        image: quay.io/rdiazgav/spring-petclinic:latest
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 45
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        ports:
        - containerPort: 8080
          protocol: TCP
        - containerPort: 8443
          protocol: TCP
        - containerPort: 8778
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 45
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 5
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-deployment
  namespace: $NAMESPACE_PETCLINIC
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:latest
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: root
            - name: MYSQL_DATABASE
              value: petclinic
            - name: MYSQL_USER
              value: petclinic
            - name: MYSQL_PASSWORD
              value: petclinic
          ports:
            - containerPort: 3306
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-data
          persistentVolumeClaim:
            claimName: mysql-data-pvc-${FACTOR}gb
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: $NAMESPACE_PETCLINIC
spec:
  selector:
    app: mysql
  ports:
    - protocol: TCP
      port: 3306
      targetPort: 3306
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data-pvc-${FACTOR}gb
  namespace: $NAMESPACE_PETCLINIC
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: ${FACTOR}Gi # Specify the desired storage size
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: spring-petclinic
  name: spring-petclinic
  namespace: $NAMESPACE_PETCLINIC
spec:
  ports:
  - name: 8080-tcp
    port: 8080
    protocol: TCP
    targetPort: 8080
  - name: 8443-tcp
    port: 8443
    protocol: TCP
    targetPort: 8443
  - name: 8778-tcp
    port: 8778
    protocol: TCP
    targetPort: 8778
  selector:
    app: spring-petclinic
  sessionAffinity: None
  type: ClusterIP
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  labels:
    app: spring-petclinic
  name: spring-petclinic
  namespace: $NAMESPACE_PETCLINIC
spec:
  port:
    targetPort: 8080-tcp
  to:
    kind: Service
    name: spring-petclinic
    weight: 100
  tls: 
    termination: edge
---
EOF
```

# Create benchmark information 
```console
FILE_BENCHMARK_SIZE=$(( $FACTOR / 2 ))
oc exec -n $NAMESPACE_PETCLINIC deploy/mysql-deployment -it -- \
  /bin/sh -c "head -c ${FILE_BENCHMARK_SIZE}G < /dev/urandom > /var/lib/mysql/testfile.benchmark${FACTOR}gb.bin"
oc exec -n $NAMESPACE_PETCLINIC deploy/mysql-deployment -it -- \
  /bin/sh -c "sha1sum /var/lib/mysql/testfile.benchmark${FACTOR}gb.bin > /var/lib/mysql/testfile.sha1.txt && cat /var/lib/mysql/testfile.sha1.txt"
```

# Execute Benchmark Backup

## Scripts

```console
cat <<EOF | oc create -f - 
apiVersion: velero.io/v1
kind: Backup
metadata:
  generateName: backup-${NAMESPACE_PETCLINIC}-
  namespace: openshift-adp
spec:
  csiSnapshotTimeout: 24h00m0s
  itemOperationTimeout: 24h0m0s
  includedNamespaces:
    - $NAMESPACE_PETCLINIC
  snapshotMoveData: true
EOF
```

# Results

## Disk 10G / Data > 5G 
- *Backup*: 3m 27s 
- *Restore*: 3m 05s

### Node Metrics from 'Observe' (backup):
![Node metrics view 1](screenshoots/node-backup-10gb-1.jpg)
![Node metrics view 2](screenshoots/node-backup-10gb-2.jpg)

### POD Metrics from 'Observe' (backup):
![POD metrics view 1](screenshoots/agent-backup-10gb-1.jpg)
![POD metrics view 2](screenshoots/agent-backup-10gb-2.jpg)

### Node Metrics from 'Observe' (restore):
![Node metrics view 1](https://github.com/CSA-RH/oadp-benchmark-demo/blob/main/screenshoots/node_view1.png)
![Node metrics view 2](https://github.com/CSA-RH/oadp-benchmark-demo/blob/main/screenshoots/node_view2.png)

### POD Metrics from 'Observe' (restore):
![POD metrics view 1](https://github.com/CSA-RH/oadp-benchmark-demo/blob/main/screenshoots/pod_view1.png)
![POD metrics view 2](https://github.com/CSA-RH/oadp-benchmark-demo/blob/main/screenshoots/pod_view2.png)

## Disk 100G / Data > 50G 
- *Backup*: 1h 14m 
- *Restore*: 37m
  
### Node Metrics from 'Observe' (backup):
![Node metrics view 1](screenshoots/node-exporter-backup-100gb-1.jpg)
![Node metrics view 2](screenshoots/node-exporter-backup-100gb-2.jpg)

### POD Metrics from 'Observe' (backup):
![POD metrics view 1](screenshoots/node-agent-backup-100gb-1.jpg)
![POD metrics view 2](screenshoots/node-agent-backup-100gb-2.jpg)
