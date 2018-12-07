---
layout: page
title: SFTP Kubernetes Pod with Azure File Share on AKS
permalink: /wip-aks-sftp/
---

This post assumes you already have an AKS Kubernetes cluster up and running. If you do not, please refer to my previous post "Deploying an Azure Kubernetes Service cluster quickly and painlessly". We will be doing everything here through the Azure Console, although you can do this on your own terminal as well.

Create Azure File Share: https://docs.microsoft.com/en-us/azure/aks/azure-files-volume

Azure File Share mounting requires the kubernetes secret 'azure-secret'

```bash
AKS_PERS_STORAGE_ACCOUNT_NAME=blah
STORAGE_KEY=bluh
kubectl create secret generic azure-secret --from-literal=azurestorageaccountname=$AKS_PERS_STORAGE_ACCOUNT_NAME --from-literal=azurestorageaccountkey=$STORAGE_KEY
```

Log into cluster `az aks get-credentials --resource-group FOO --name BAR`

Password for the ftp user

sftp-server-sec

`kubectl create secret generic sftp-server-sec --from-literal=password=password`

```yaml
kind: Service
apiVersion: v1
metadata:
  name: sftp
  namespace: default
  labels:
    environment: test
spec:
  ports:
  - name: "ssh"
    port: 22
    targetPort: 22
  selector:
    app: sftp

---

kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: sftp
  namespace: default
  labels:
    environment: test
    app: sftp
spec:
  # how many pods and indicate which strategy we want for rolling update
  replicas: 1
  minReadySeconds: 10

  template:

    metadata:
      labels:
        environment: test
        app: sftp
      annotations:
        container.apparmor.security.beta.kubernetes.io/sftp: runtime/default

    spec:
      #secrets and config
      volumes:
      - name: azure
        azureFile:
          secretName: azure-secret
          shareName: papicatbkp
          readOnly: false
      
      containers:
        #the sftp server itself
        - name: sftp
          image: atmoz/sftp:latest
          imagePullPolicy: Always
          env:
          - name: PASSWORD
            valueFrom:
              secretKeyRef:
                name: sftp-server-sec
                key: password
          args: ["sftp_user:$(PASSWORD):1001:100:incoming,outgoing"] #create users and dirs
          ports:
            - containerPort: 22
          volumeMounts:
            - name: azure
              mountPath: /home/sftp_user/backups
          securityContext:
            capabilities:
              add: ["SYS_ADMIN"]
          resources: {}
```



kubectl apply -f sftp.yaml

Check pod, describe pod, service, explain...

Test cluster by using a simple debian pod

kubectl run debian --image=debian

kubectl exec -it debian-asdf-asdf- /bin/bash

```
sftp commands from inside container to test connection across pods
```