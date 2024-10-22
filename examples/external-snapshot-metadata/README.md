# Example snapshot-metadata Service with hostpath CSI Driver

This document illustrates how to install a hostpath CSI driver with the external-snapshot-metadata sidecar. You can use this example as a reference when installing a CSI driver with Changed Block Metadata (CBT) support in your cluster.


## Installation

In this example, we will deploy the snapshot-metadata service alongside a hostpath CSI driver. While this example uses a hostpath CSI driver, the steps may vary depending on the specific CSI driver you are using. Use the appropriate steps to deploy the CSI driver in your environment.

**Steps to deploy snapshot-metadata with a hostpath CSI driver:**

1. Create `SnapshotMetadataService` CRD
   ```bash
   $ kubectl create -f crd/cbt.storage.k8s.io_snapshotmetadataservices.yaml
   ```

1. Provision TLS Certs

   Generate self-signed certificates using following commands

   ```bash
   NAMESPACE="default"

   # 1. Create extension file
   echo "subjectAltName=DNS:.default,DNS:csi-hostpathplugin.default,IP:0.0.0.0" > server-ext.cnf 

   # 2. Generate CA's private key and self-signed certificate
   openssl req -x509 -newkey rsa:4096 -days 365 -nodes -keyout ca-key.pem -out ca-cert.pem -subj "/CN=csi-hostpathplugin.${NAMESPACE}"

   openssl x509 -in ca-cert.pem -noout -text
   
   # 3. Generate web server's private key and certificate signing request (CSR)
   openssl req -newkey rsa:4096 -nodes -keyout server-key.pem -out server-req.pem -subj "/CN=csi-hostpathplugin.${NAMESPACE}"
   
   # 4. Use CA's private key to sign web server's CSR and get back the signed certificate
   openssl x509 -req -in server-req.pem -days 60 -CA ca-cert.pem -CAkey ca-key.pem -CAcreateserial -out server-cert.pem -extfile server-ext.cnf
   
   openssl x509 -in server-cert.pem -noout -text
   ```

2. Create a TLS secret 

   ```bash
   $ kubectl create secret tls csi-hostpathplugin-certs --namespace=default --cert=server-cert.pem --key=server-key.pem 
   ```

3. Create `SnapshotMetadataService` resource

   The name of the `SnapshotMetadataService` resource must match the name of the CSI driver for which you want to enable the CBT feature. In this example, we will create a `SnapshotMetadataService` for the `hostpath.csi.k8s.io` CSI driver.

   Create a file named `snapshotmetadataservice.yaml` with the following content:

   ```yaml
   apiVersion: cbt.storage.k8s.io/v1alpha1
   kind: SnapshotMetadataService
   metadata:
     name: hostpath.csi.k8s.io
   spec:
     address: csi-hostpathplugin.default:6443
     caCert: GENERATED_CA_CERT
     audience: 005e2583-91a3-4850-bd47-4bf32990fd00
   ```

   Encode the CA Cert:

   ```bash
   $ base64 -i ca-cert.pem
   ```

   Copy the output and replace `GENERATED_CA_CERT` in the `SnapshotMetadataService` CR.
   Update `spec.address` and `spec.audience` if required.

   Create `SnapshotMetadataService` resource using the command below:

   ```bash
   $ kubectl create -f snapshotmetadataservice.yaml
   ```

6. Deploy the hostpath CSI driver with snapshot-metadata sidecar service

   ```bash
   $ ./deploy/deploy.sh 
   ```

5. Create ClusterRole and ClusterRoleBinding for snapshot-metadata service

   ```bash
   $ kubectl create -f csi-snapshot-metadata-rbac.yaml
   ```

7. Create a service to expose communication to CSI driver pod

   ```bash
   $ kubectl create -f csi-snapshot-metadata-service.yaml
   ```
