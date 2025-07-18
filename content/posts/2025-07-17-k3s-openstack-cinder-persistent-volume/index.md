---
title: เชื่อม K3s เข้ากับ OpenStack Block Storage (Cinder) เพื่อสร้าง Persistent
  Volume
slug: k3s-openstack-cinder-persistent-volume
date: 2025-07-17 21:16:00 +0700
description: เชื่อมต่อ K3s เข้ากับ Block Storage ของ OpenStack (Cinder)
  เพื่อให้สามารถจัดการ Persistent Volume ได้อย่างมีประสิทธิภาพ
  ลดความเสี่ยงจากการสูญหายของข้อมูล และเพิ่มความทนทานให้กับระบบ
comments: true
categories: เทคโนโลยี
tags:
  - kubernetes
  - k3s
  - openstack
image: images/k3s-openstack-csi.png
---
จากบล็อกก่อนหน้านี้ ผู้เขียนได้แนะนำการเชื่อม K3s เข้ากับ OpenStack Load Balancer ไปแล้ว ในบทความนี้จะมาลุยต่อด้วยการเชื่อม K3s เข้ากับ Block Storage (Cinder) ของ OpenStack

## คำเตือน (อีกครั้ง)

ก่อนจะไปต่อ ผู้เขียนขอย้ำอีกครั้งว่า โดยทั่วไปแล้วการใช้งาน Kubernetes ควรเลือกใช้บริการ Managed Kubernetes จาก Cloud Provider จะดีที่สุด เพราะสะดวกและมีความเสถียรสูงกว่าการติดตั้งและตั้งค่าคลัสเตอร์ Kubernetes ด้วยตัวเอง ยกเว้นกรณีต่อไปนี้

* Cloud Provider ที่ใช้อยู่ไม่มีบริการ Managed Kubernetes
* มีทีมงานที่เชี่ยวชาญด้าน Kubernetes โดยเฉพาะ
* ใช้สำหรับสร้างระบบทดสอบ (Testing) หรือระบบที่ไม่ค่อยมีความสำคัญ (Non-critical)
* ต้องการติดตั้งเพื่อศึกษาหาความรู้

## เป้าหมาย

เป้าหมายของบทความนี้คือ ทำให้ K3s สามารถจัดการ Block Storage ของ OpenStack (ที่มีชื่อบริการว่า Cinder) ได้โดยตรง ผลลัพธ์ที่ต้องการคือ เมื่อผู้ใช้สร้าง PersistentVolumeClaim (PVC) ใน K3s ระบบจะต้องไปสร้าง Block Storage Volume บน OpenStack ให้โดยอัตโนมัติ และเมื่อมีการขยายขนาดของ PVC ระบบก็จะต้องไปขยายขนาดของ Block Storage ด้วยเช่นกัน

## ทำไมต้องเชื่อม Kubernetes กับ Block Storage ของ Cloud?

หากไม่ทำการเชื่อมต่อนี้ เวลาที่ต้องการใช้งาน Persistent Volume บน K3s เราจะถูกจำกัดให้ใช้ได้แค่ local volume หรือ Local Path Provisioner ซึ่งเป็น Storage Provisioner พื้นฐานที่ติดตั้งมาพร้อมกับ K3s อยู่แล้ว

ปัญหาหลักของ Local Path Provisioner คือข้อมูลจะถูกเก็บไว้บนพื้นที่ของโนด (Node) ที่ Pod ถูกสร้างขึ้นมาเท่านั้น ซึ่งหมายความว่าข้อมูลจะผูกติดอยู่กับโนดนั้น ๆ หากโนดดังกล่าวเกิดล่มหรือมีปัญหา ข้อมูลที่เก็บอยู่ก็จะไม่สามารถใช้งานได้จนกว่าจะกู้โนดนั้นกลับมาได้สำเร็จ วิธีนี้จึงทำให้เกิด Single Point of Failure (SPoF)

จริง ๆ ก็มีทางแก้ปัญหานี้โดยไม่ทำให้เกิด SPoF เช่น การติดตั้ง Software-Defined Storage อย่าง Rook-Ceph เพื่อสร้าง Storage Cluster ของตัวเองขึ้นมา แต่วิธีนี้ก็มาพร้อมกับความซับซ้อนในการติดตั้งและดูแลรักษา แถมยังเปลืองค่าใช้จ่ายด้าน Block Storage เพิ่มขึ้นอีก ถึงแม้จะใช้เทคนิค erasure code เพื่อลดการใช้พื้นที่แล้วก็ตาม

นอกจากนี้ยังอาจเป็นการเพิ่ม Overhead ถึงสองต่อ เพราะ Cloud Provider ที่ใช้ OpenStack เป็นพื้นฐานส่วนใหญ่มักจะใช้ Ceph เป็น Backend Storage อยู่แล้ว การที่เราติดตั้ง Rook-Ceph บนนั้นอีกชั้นจึงไม่ต่างอะไรกับการใช้ "Ceph ซ้อน Ceph" ซึ่งไม่ใช่วิธีที่มีประสิทธิภาพนัก

ดังนั้น การเชื่อม Kubernetes เข้ากับ Block Storage ของ OpenStack โดยตรงจึงเป็นทางออกที่เรียบง่ายและมีประสิทธิภาพสูงสุดสำหรับกรณีนี้

## ขั้นตอนการติดตั้งและตั้งค่า

เพื่อทำให้ K3s คุยกับ Cinder ได้ เราจะติดตั้ง Cinder CSI Driver โดยผู้เขียนจะอ้างอิงวิธีการติดตั้งจาก[เอกสารทางการของ cloud-provider-openstack](https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/cinder-csi-plugin/using-cinder-csi-plugin.md) ซึ่งมีขั้นตอนดังต่อไปนี้

**หมายเหตุ:** หากมี Secret cloud-config ที่ทำไว้ตอนเชื่อม K3s เข้ากับ OpenStack Load Balancer อยู่แล้ว ให้ข้ามไปขั้นตอนที่ 3 ได้เลย

### **ขั้นตอนที่ 1: สร้างไฟล์ตั้งค่า cloud.conf**

สร้างไฟล์ `cloud.conf` ที่มีข้อมูลสำหรับเชื่อมต่อ OpenStack ตามตัวอย่างด้านล่าง

```ini
[Global]
auth-url=https://keystone.example.com/v3
username=your_username
password=your_password
region=TH-BKK
tenant-id=9f6dbf311397409a92cbbc761c7f8865
domain-id=c6b00adf4ed04fc5a958121fadb0e401
```

โดยระบุค่าต่าง ๆ ให้ถูกต้อง ดังนี้

* `auth-url` ดูได้จากไฟล์ OpenStack RC ที่ได้ดาวน์โหลดมาจาก UI หรือใช้คำสั่ง `openstack versions show`
* `username` และ `password` เป็นชุดเดียวกับที่ใช้เข้าหน้า UI
* `region` ดูได้จากไฟล์ OpenStack RC ที่ได้ดาวน์โหลดมาจาก UI หรือใช้คำสั่ง `openstack project list`
* `tenant-id` หรือ `tenant-name` ดูได้จากไฟล์ OpenStack RC ที่ได้ดาวน์โหลดมาจาก UI หรือใช้คำสั่ง `openstack project list`
* `domain-id` หรือ `domain-name` ดูได้จากไฟล์ OpenStack RC ที่ได้ดาวน์โหลดมาจาก UI หรือใช้คำสั่ง `openstack project show <tenant-id/tenant-name>`

### ขั้นตอนที่ 2: สร้าง Secret ใน Kubernetes

เมื่อได้ไฟล์ cloud.conf มาแล้ว เราจะนำไฟล์นี้ไปสร้างเป็น Secret ใน Kubernetes เพื่อให้ Cinder CSI Driver นำไปใช้งานได้

```bash
sudo kubectl create secret generic cloud-config --from-file=cloud.conf -n kube-system
```

### ขั้นตอนที่ 3: ติดตั้ง Cinder CSI Driver Manifests

เข้าไปที่ Repository ของ `cloud-provider-openstack` แล้วดู Branch/Tag ให้ตรงกับเวอร์ชันของ Kubernetes ที่ใช้งานอยู่ ตัวอย่างเช่น หากใช้ Kubernetes เวอร์ชัน 1.32.x ก็ให้เลือก release v1.32.0 จากนั้นใช้คำสั่ง `kubectl apply` กับไฟล์ manifest ทั้งหมด ยกเว้นไฟล์ `csi-secret-cinderplugin.yaml` เพราะเรามี Secret ที่ถูกต้องอยู่แล้ว

```bash
sudo kubectl apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/refs/tags/v1.32.0/manifests/cinder-csi-plugin/cinder-csi-controllerplugin-rbac.yaml
sudo kubectl apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/refs/tags/v1.32.0/manifests/cinder-csi-plugin/cinder-csi-controllerplugin.yaml
sudo kubectl apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/refs/tags/v1.32.0/manifests/cinder-csi-plugin/cinder-csi-nodeplugin-rbac.yaml
sudo kubectl apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/refs/tags/v1.32.0/manifests/cinder-csi-plugin/cinder-csi-nodeplugin.yaml
sudo kubectl apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/refs/tags/v1.32.0/manifests/cinder-csi-plugin/csi-cinder-driver.yaml
```

### ขั้นตอนที่ 4: สร้าง StorageClass

หลังจากติดตั้ง CSI Driver เรียบร้อยแล้ว ขั้นตอนสุดท้ายคือการสร้าง StorageClass เพื่อให้ K3s รู้ว่าจะต้องใช้ Provisioner ตัวไหนในการสร้าง Volume และอนุญาตให้ขยายขนาดได้ในภายหลัง

สร้างไฟล์ `sc.yaml` ขึ้นมาด้วยเนื้อหาดังนี้:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cinder-sc
provisioner: cinder.csi.openstack.org
allowVolumeExpansion: true
```

จากนั้นสั่ง apply:

```bash
sudo kubectl apply -f sc.yaml
```

เพียงเท่านี้ K3s Cluster ของเราก็พร้อมที่จะสร้าง Persistent Volume ผ่าน OpenStack Cinder แล้ว

### ขั้นตอนที่ 4: ทดสอบการทำงาน

สร้างไฟล์ `pvc-busybox.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: busybox-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: cinder-sc
```

สร้างไฟล์ `pod-busybox.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox-pod
spec:
  containers:
    - name: busybox
      image: busybox
      command: ["sleep", "3600"]
      volumeMounts:
        - mountPath: /data
          name: busybox-storage
  volumes:
    - name: busybox-storage
      persistentVolumeClaim:
        claimName: busybox-pvc
```

apply ทั้ง 2 ไฟล์:

```bash
sudo kubectl apply -f pvc-busybox.yaml
sudo kubectl apply -f pod-busybox.yaml
```

สร้างไฟล์ใน Volume:

```bash
sudo kubectl exec -it busybox-pod -- sh -c 'echo hello > /data/hello.txt'
```

ตรวจสอบว่าไฟล์ถูกสร้าง:

```bash
sudo kubectl exec -it busybox-pod -- cat /data/hello.txt
```

ลบ Pod:

```bash
sudo kubectl delete pod busybox-pod
```

apply ไฟล์ pod-busybox.yaml เพื่อสร้าง Pod ขึ้นใหม่:

```bash
sudo kubectl apply -f pod-busybox.yaml
```

ตรวจสอบว่าไฟล์ยังคงอยู่:

```bash
sudo kubectl exec -it busybox-pod-2 -- cat /data/hello.txt
```

### สรุป

การเชื่อมต่อ K3s เข้ากับ OpenStack Block Storage (Cinder) โดยตรงผ่าน CSI Driver เป็นวิธีที่ช่วยให้สามารถจัดการ Persistent Volume ได้อย่างมีประสิทธิภาพและทนทาน ทำให้การบริหารจัดการ Storage สำหรับแอปพลิเคชันบน Kubernetes ของเราเป็นไปอย่างราบรื่นและเหมาะสมกับสถาปัตยกรรมแบบคลาวด์อย่างแท้จริง
