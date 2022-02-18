# Kubernetes Meetup Tokyo 48, めっちゃわかる! metadata.managedFields

- [Kubernetes Meetup Tokyo #48 - connpass](https://k8sjp.connpass.com/event/237734/)

## 事前準備

Flux2 が Server Side Apply でマニフェストを適用を再現。この時点では `test-data` ラベルが存在している。

```
$ kubectl apply -f 1-configmap.yaml --server-side --field-manager flux-controller
configmap/config serverside-applied

kubectl get cm config -o yaml --show-managed-fields
apiVersion: v1
kind: ConfigMap
metadata:
  creationTimestamp: "2022-02-18T01:38:19Z"
  labels:
    test-data: value1
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:labels:
          f:test-data: {}
    manager: flux-controller
    operation: Apply
    time: "2022-02-18T01:38:19Z"
  name: config
  namespace: default
  resourceVersion: "3839182"
  uid: 57278dfe-fb30-4386-ba67-65461c5e8fae
```

## 1. 開発者が Flux2 による適用を停止し、手元でマニフェストを変更し適用。

ここでは Flux2 役を人間がやっているので、停止の処理はスキップ。停止することで勝手に再度適用されないということ。

開発者がローカルでマニフェストを修正し、適用する。修正済みのマニフェストファイルは `2-configmap.yaml`。

```
$ diff -u 1-configmap.yaml 2-configmap.yaml
--- 1-configmap.yaml    2022-02-18 10:36:39.197624382 +0900
+++ 2-configmap.yaml    2022-02-18 10:39:11.703819006 +0900
@@ -3,4 +3,4 @@
 metadata:
   name: config
   labels:
-    test-data: value1
+    test-data: value2

$ kubectl apply -f 2-configmap.yaml
Warning: resource configmaps/config is missing the kubectl.kubernetes.io/last-applied-configuration annotation which is required by kubectl apply. kubectl apply should only be used on resources created declaratively by either kubectl create --save-config or kubectl apply. The missing annotation will be patched automatically.
configmap/config configured

$ kubectl get cm config -o yaml --show-managed-fields
apiVersion: v1
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"ConfigMap","metadata":{"annotations":{},"labels":{"test-data":"value2"},"name":"config","namespace":"default"}}
  creationTimestamp: "2022-02-18T01:38:19Z"
  labels:
    test-data: value2
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:kubectl.kubernetes.io/last-applied-configuration: {}
        f:labels:
          f:test-data: {}
    manager: kubectl-client-side-apply
    operation: Update
    time: "2022-02-18T01:39:44Z"
  name: config
  namespace: default
  resourceVersion: "3839331"
  uid: 57278dfe-fb30-4386-ba67-65461c5e8fae
```

## 2. 開発者は手元のマニフェストでうまくいくことがわかったのでリポジトリに変更をコミット

ここではそれをしたということにする。

## 3. 開発者は Flux2 により適用を再開

再開されたことで Flux2 が SSA でマニフェストを適用する。

```
$ kubectl apply -f 2-configmap.yaml --server-side --field-manager flux-controller
configmap/config serverside-applied

$ kubectl get cm config -o yaml --show-managed-fields
apiVersion: v1
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"ConfigMap","metadata":{"annotations":{},"labels":{"test-data":"value2"},"name":"config","namespace":"default"}}
  creationTimestamp: "2022-02-18T01:38:19Z"
  labels:
    test-data: value2
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:labels:
          f:test-data: {}
    manager: flux-controller
    operation: Apply
    time: "2022-02-18T01:41:18Z"
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:kubectl.kubernetes.io/last-applied-configuration: {}
        f:labels:
          f:test-data: {}
    manager: kubectl-client-side-apply
    operation: Update
    time: "2022-02-18T01:39:44Z"
  name: config
  namespace: default
  resourceVersion: "3839492"
  uid: 57278dfe-fb30-4386-ba67-65461c5e8fae
```

## 4. 開発者は `test-data` ラベルがいらなくなったので削除し、変更をリポジトリにコミット

変更をコミットしたことで Flux2 がマニフェストを再度適用する。

しかし、削除したはずのラベルが残ったままであることが確認できる。

```
$ diff -u 2-configmap.yaml 3-configmap.yaml
--- 2-configmap.yaml    2022-02-18 10:39:11.703819006 +0900
+++ 3-configmap.yaml    2022-02-18 10:42:34.234679373 +0900
@@ -3,4 +3,3 @@
 metadata:
   name: config
   labels:
-    test-data: value2

$ kubectl apply -f 3-configmap.yaml --server-side --field-manager flux-controller
configmap/config serverside-applied

$ kubectl get cm config -o yaml --show-managed-fields
apiVersion: v1
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"ConfigMap","metadata":{"annotations":{},"labels":{"test-data":"value2"},"name":"config","namespace":"default"}}
  creationTimestamp: "2022-02-18T01:38:19Z"
  labels:
    test-data: value2
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:labels: {}
    manager: flux-controller
    operation: Apply
    time: "2022-02-18T01:43:03Z"
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:kubectl.kubernetes.io/last-applied-configuration: {}
        f:labels:
          f:test-data: {}
    manager: kubectl-client-side-apply
    operation: Update
    time: "2022-02-18T01:39:44Z"
  name: config
  namespace: default
  resourceVersion: "3839668"
  uid: 57278dfe-fb30-4386-ba67-65461c5e8fae
```

## 5. ここでの解決策

`metadata.managedFields` に注目すると、Flux2 (`flux-controller` manager) は、`test-data` ラベルの所有権をもっていないことがわかる。

しかし、`kubectl-client-side-apply` manager が `test-data` ラベルの所有権を持っている。そのため、Flux2 が `test-data` ラベルのないマニフェストを適用しても、Flux2 がこのフィールドの所有権を持っていないので無視される。

```
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:labels: {}
    manager: flux-controller
    operation: Apply
    time: "2022-02-18T01:43:03Z"
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:kubectl.kubernetes.io/last-applied-configuration: {}
        f:labels:
          f:test-data: {}
    manager: kubectl-client-side-apply
    operation: Update
    time: "2022-02-18T01:39:44Z"
```

つまり、ここでの解決策は、`kubectl-client-side-apply` manager が `test-data` ラベルのないマニフェストを適用し、そのフィールドの所有権を放棄することでフィールドが削除される。

```
$ kubectl apply -f 3-configmap.yaml
configmap/config serverside-applied
```

このあと Flux2 の SSA による適用が自動的に実行される。

```
$ kubectl apply -f 3-configmap.yaml --server-side --field-manager flux-controller
configmap/config serverside-applied
```

オブジェクトの状態を確認すると、無事 `test-data` ラベルが削除されていることがわかる。

```
$ kubectl get cm config -o yaml --show-managed-fields
apiVersion: v1
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"ConfigMap","metadata":{"annotations":{},"labels":null,"name":"config","namespace":"default"}}
  creationTimestamp: "2022-02-18T01:38:19Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:labels: {}
    manager: flux-controller
    operation: Apply
    time: "2022-02-18T01:44:21Z"
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:kubectl.kubernetes.io/last-applied-configuration: {}
    manager: kubectl-client-side-apply
    operation: Update
    time: "2022-02-18T01:44:00Z"
  name: config
  namespace: default
  resourceVersion: "3839802"
  uid: 57278dfe-fb30-4386-ba67-65461c5e8fae
```
