# Backing up TAP

> **_NOTE:_**  This research is a work in progress

* Versions tested:
    * TAP Version 1.3.0
    * Velero Version 1.9.5

* Steps taken so far:

1. Establish a secret containing a ytt overlay for the metadata-store statefulset, shown below.  According to [this doc](https://docs.vmware.com/en/VMware-Tanzu-Mission-Control/services/tanzumc-concepts/GUID-C16557BC-EB1B-4414-8E63-28AD92E0CAE5.html#about-restic-and-volume-snapshots-1), this shouldn’t be needed, however through testing with these versions, it is.  The below sets an annotation on the statefulset that triggers velero/restic to not only restore kubernetes objects for the metadata-store (like the statefulset, service, pvc, pv, etc), but will also include restoration of the actual data contained within the postgres database/persistent volume that is used for backing the metadata-store.  Velero includes the option of setting a flag to have the data restored in a PV by default, but at this time it is unsure if this flag can be set through TMC--but again this should be the default dataprotection behavior.  The below uses the [Velero opt-out approach](https://velero.io/docs/v1.9/restic/#to-back-up).

    > **_NOTE:_**  Need to follow up with Velero support to check into why opt-out isn't working by default

2. It also sets runAsUser to resolve a postgres error: 

    `Error: container has runAsNonRoot and image has non-numeric user (nonroot), cannot verify user is non-root` 

...that prevents the metadata-store DB pods from starting.

    ```
    apiVersion: v1
    kind: Secret
    metadata:
      name: metadata-store-overlay-secret
      namespace: tap-install
    stringData:
      custom-package-overlay.yml: |
        #@ load("@ytt:overlay", "overlay")
        #@overlay/match by=overlay.subset({"kind":"StatefulSet","metadata":{"name": "metadata-store-db"}})
        ---
        spec:
          template:
            metadata:
              #@overlay/match missing_ok=True
              annotations:
                #@overlay/match missing_ok=True
                backup.velero.io/backup-volumes-excludes: ""
            spec:
              securityContext:
                #@overlay/match missing_ok=True
                runAsUser: 999
    ```

3. Establish another secret for the source-controller, as it also requires a numeric runAsUser value:

    ```
    apiVersion: v1
    kind: Secret
    metadata:
      name: source-controller-overlay-secret
      namespace: tap-install
    stringData:
      custom-package-overlay.yml: |
        #@ load("@ytt:overlay", "overlay")
        #@overlay/match by=overlay.subset({"kind":"Deployment","metadata":{"name": "source-controller-manager"}})
        ---
        spec:
          template:
            spec:
              securityContext:
                #@overlay/match missing_ok=True
                runAsUser: 999
    ```

4. Ensure tap-values.yaml is updated to include a package_overlays section:

    ```
    package_overlays:
    - name: metadata-store
      secrets:
      - name: metadata-store-overlay-secret
    - name: source-controller
      secrets:
      - name: source-controller-overlay-secret
    ```

5. Apply the update and check the metadata-store statefulset and source-system deployment to make sure the values are updated accordingly.
6. If the source TAP cluster doesn’t already use a database to persist TAP GUI data, make sure to create one.  This example namespace, statefulset & supporting resources or something similar could be used.  Be sure to note the velero backup annotation:
	
    ```
    ---
    apiVersion: v1
    kind: Namespace
    metadata:
      name: tap-gui-postgres
    ---
    apiVersion: v1
    kind: Secret
    metadata:
      name: tap-gui-postgres-secrets
      namespace: tap-gui-postgres
    type: Opaque
    data:
      POSTGRES_USER: <redacted>
      POSTGRES_PASSWORD: <redacted>
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: tap-gui-postgres-storage-claim
      namespace: tap-gui-postgres
    spec:
      storageClassName: vc01cl01-t0compute
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 10G
    ---
    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      name: tap-gui-postgres
      namespace: tap-gui-postgres
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: tap-gui-postgres
      serviceName: tap-gui-postgres
      template:
        metadata:
          labels:
            app: tap-gui-postgres
          annotations:
            backup.velero.io/backup-volumes-excludes: ""
        spec:
          initContainers:
          - name: "remove-lost-found"
            image: harbor-repo.vmware.com/jeffataprepo/library@sha256:3b3128d9df6bbbcc92e2358e596c9fbd722a437a62bafbc51607970e9e3b8869
            command:
            - 'sh'
            - '-c'
            - 'rm -rf /var/lib/postgresql/data/lost+found'
            volumeMounts:
            - mountPath: "/var/lib/postgresql/data"
              name: tap-gui-postgresdb
          containers:
            - name: tap-gui-postgres
              image: harbor-repo.vmware.com/jeffataprepo/library@sha256:c4c7a1585974706b5f72b8ab595e47399b23b2e03d93bbf75c1b0904be1803dc
              ports:
                - containerPort: 5432
              envFrom:
                - secretRef:
                    name: tap-gui-postgres-secrets
              volumeMounts:
                - mountPath: /var/lib/postgresql/data
                  name: tap-gui-postgresdb
          volumes:
            - name: tap-gui-postgresdb
              persistentVolumeClaim:
                claimName: tap-gui-postgres-storage-claim
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: tap-gui-postgres
      namespace: tap-gui-postgres
    spec:
      selector:
        app: tap-gui-postgres
      ports:
        - port: 5432
    ```

7. Now things should be set up for taking a backup.  Before beginning, make note of the resources you have available via the source TAP GUI.  This will be your current state to validate later once TAP GUI is restored in the backup cluster.
8. Make note of:
	
    * Your Organization Catalog, entries in the runtime resources tab
    * Workloads that show with last pipeline run, last builds, and all CVE’s details
    * Vulnerabilities that show in Security Analysis

9. Enable Data Protection via TMC on both clusters.  This is needed for restic (link).

    ```
    tmc cluster dataprotection create \
          --cluster-name="jeffa2-backup" \
          --management-cluster-name="jeffa2-mgmt-cluster" \
          --provisioner-name="devns"
    ```

10. Take note of all of the TAP oriented namespaces, as well as others you would like to have restored.  Do not restore the system namespaces, just the TAP oriented ones and cluster scoped resources.
11. Take backup of Full cluster
    * UI:
        * TMC -> Full cluster -> Data Protection -> Create Backup 
        * Back up selected namespaces
            * select only the above namespaces
            * now, make sure to include the accompanying cluster-scoped resources
                * Advanced Options -> Include cluster-scoped resources
    * CLI (preferred):

    ```
    tmc cluster dataprotection backup create --cluster-name="jeffa2-tap-full" --include-namespaces="accelerator-system,api-auto-registration,api-portal,app-live-view,app-live-view-connector,app-live-view-conventions,appsso,build-service,cartographer-system,cert-injection-webhook,cert-manager,conventions-system,developer-conventions,development,flux-system,image-policy-system,knative-eventing,knative-serving,knative-sources,kpack,metadata-store,scan-link-system,service-bindings,services-toolkit,source-system,spring-boot-convention,stacks-operator-system,tanzu-system-ingress,tap-gui,tap-gui-postgres,tap-install,tap-telemetry,tekton-pipelines,triggermesh,vmware-sources,vmware-system-telemetry" --include-cluster-resources --backup-location-name="jeffa-clusters-2" --management-cluster-name="jeffa2-mgmt-cluster" --provisioner-name="devns" --verbosity=4 --name="tap-ns-and-cs-backup"
    ```

12. Restore to the Backup cluster
    * TMC -> Backup cluster -> Data Protection -> Restore from another cluster
    * Select the source cluster and backup taken
    * Restore the entire backup
        * Select Advanced Option to Update Existing Resources (can’t see an option to do this via the CLI, so using TMC UI for now)

13. Run tanzu package installed list -n tap-install and observe Reconciliation failures.
14. Cleanup items
    * Delete the kapp-controller pod to fix reconciliation errors.  These tend to get out of sync after a restore, deleting them and letting a new kapp-controller spawn helps overcome “Reconcile failed: the server is currently unable to handle the request (get pack…” errors
        * run tanzu package installed list -n tap-install again
        * all but tap reconciled, I looked at the useful error and had to kick ootb-supply-chain-testing-scanning.  After kicking tap, and it all succeeded.
    * Deleted antrea controller based off some errors I saw, and let it respawn.
        * `k delete pod -n kube-system antrea-controller-bb59f5fbf-x9np7`
    * Deleted sourcescan to get past an error 
        * `kubectl -n development delete sourcescan tanzu-java-web-app`

15. Update the metadata-store bearer token (wish I didn’t have to do this).

    ```
    SERVICE_ACCOUNT_SECRET_NAME=`kubectl get secrets -n metadata-store |grep metadata-store-read-client-token |sed 's/ .*//'`
    METADATA_STORE_ACCESS_TOKEN=$(kubectl get secrets $SERVICE_ACCOUNT_SECRET_NAME -n metadata-store -o jsonpath="{.data.token}" | base64 -d)
    echo "METADATA_STORE_ACCESS_TOKEN is $METADATA_STORE_ACCESS_TOKEN"
    ```

16. Include new access token in tap-values.yaml and update of TAP on the backup cluster and make sure it reconciles.
17. Change DNS of your TAP GUI to point to the tanzu-system-ingress of your new backup TAP GUI
18. Observe the state of things in your backup TAP GUI:
    * Validate the following are available from what you noted before starting the recovery:

        * Your Organization Catalog, entries in the runtime resources tab
        * Workloads that show with last pipeline run, last builds, and all CVE’s details
        * Vulnerabilities that show in Security Analysis
