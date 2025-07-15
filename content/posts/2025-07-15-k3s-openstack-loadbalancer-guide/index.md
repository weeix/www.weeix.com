---
title: "เชื่อม K3s กับ OpenStack Load Balancer: คู่มือฉบับลุยจริง เจ็บจริง"
slug: k3s-openstack-loadbalancer-guide
date: 2025-07-15 21:49:00 +0700
description: ติดตั้ง k3s บน OpenStack แล้วเจอปัญหา Service type:LoadBalancer
  ไม่ทำงาน? บทความนี้จะพาคุณลุยทุกขั้นตอนการเชื่อมต่อ k3s กับ OpenStack Load
  Balancer ผ่าน Cloud Controller Manager
  พร้อมแชร์ปัญหาจริงและวิธีแก้เฉพาะหน้าที่คุณอาจต้องเจอ!
comments: true
categories: เทคโนโลยี
tags:
  - kubernetes
  - k3s
  - openstack
  - cloud-controller-manager
image: images/k3s-openstack-ccm.png
---
บทความนี้จะมาแชร์ประสบการณ์การเชื่อมต่อ K3s ซึ่งเป็น Kubernetes distribution ขนาดเล็ก เข้ากับ Load Balancer ของ Public Cloud ที่พัฒนาต่อยอดมาจาก OpenStack

เรื่องของเรื่องคือผู้เขียนไปได้ไปเครดิตฟรีจาก Public Cloud เจ้าหนึ่งมาใช้งาน แล้วก็เกิดความคิดที่อยากจะลองติดตั้งคลัสเตอร์ Kubernetes ดู โดยที่เลือก K3s เป็นเพราะติดตั้งง่ายดี ใช้คำสั่งแค่บรรทัดเดียวก็พร้อมใช้งานแล้ว

```
curl -sfL https://get.k3s.io | sh -
```

ง่ายขนาดนี้ก็ไปลุยกันเลย แต่ก่อนหน้านั้น...

## คำเตือน! ก่อนจะลงมือทำตาม

การติดตั้งคลัสเตอร์ Kubernetes ด้วยตัวเองบน Cloud (หรือจะบนเครื่องแม่ข่ายก็เหมือนกัน) มีความซับซ้อนและต้องดูแลรักษาในระยะยาว ผู้เขียนจึงอยากแนะนำว่าควรใช้บริการ Managed Kubernetes (เช่น EKS, AKS, GKE) ของ Cloud Provider นั้น ๆ จะดีที่สุด เพราะสะดวกและมีเสถียรภาพสูงกว่ามาก

อย่างไรก็ตาม การติดตั้งด้วยตัวเองก็ยังเป็นทางเลือกที่น่าสนใจในกรณีต่อไปนี้:

* Cloud Provider ที่คุณใช้ ไม่มี บริการ Managed Kubernetes
* องค์กรของคุณมีทีมงานที่เชี่ยวชาญด้าน Kubernetes และพร้อมที่จะดูแลรักษาระบบเอง
* คุณต้องการสร้างคลัสเตอร์สำหรับ ระบบทดสอบ (Testing/Development) หรือระบบที่ไม่สำคัญมากนัก
* คุณต้องการติดตั้งเพื่อศึกษาหาความรู้ และทำความเข้าใจการทำงานของ Kubernetes ให้ลึกซึ้งยิ่งขึ้น

หากเข้าข่ายข้อใดข้อหนึ่งข้างต้น... ไปต่อกันเลย!

## เป้าหมายของเราคืออะไร?

เป้าหมายหลักของภารกิจนี้คือการทำให้คลัสเตอร์ Kubernetes ของเราสามารถ "คุย" กับระบบของ OpenStack เพื่อจัดการ Load Balancer ได้โดยอัตโนมัติ พูดง่าย ๆ คือ เมื่อเราสร้าง Service ใน Kubernetes โดยระบุชนิดเป็น type: LoadBalancer เราต้องการให้เกิดสิ่งต่อไปนี้:

1. มีการสร้าง Load Balancer ขึ้นใน OpenStack โดยอัตโนมัติ
2. Load Balancer นั้นถูกตั้งค่าให้กระจาย Traffic ไปยังโนดทั้งหมดในคลัสเตอร์ของเราอย่างถูกต้อง

## ทำไมต้องใช้ Load Balancer ของ Cloud Provider?

ถ้าไม่ทำแบบนี้ได้ไหม? ได้สิ ยังมีอีกหลายวิธี เช่น:

* **ใช้ ServiceLB ที่มากับ K3s:** โดยเมื่อสร้าง Service แล้ว ServiceLB จะเปิดพอร์ตที่เราเลือกบนทุกโนดในคลัสเตอร์ ทำให้เราสามารถใช้เทคนิค DNS Round Robin สร้าง A Record ชี้มายังไอพีของทุกโนดได้ แต่กรณีที่มีโนดล่ม อาจจะทำให้ Service ของเราเข้าได้บ้าง ไม่ได้บ้าง แม้ว่าผู้ให้บริการ DNS บางรายจะมีเทคนิคในการนำไอพีที่ใช้งานไม่ได้ออกจากรอบหมุนเวียนของ Round Robin แต่กว่าจะส่งผลก็ต้องรอให้ DNS Cache หมดอายุก่อนอยู่ดี
* **สร้าง VM ติดตั้ง HAProxy/Nginx เอง:** เราสามารถสร้าง VM แยกขึ้นมาแล้วติดตั้ง Reverse Proxy เช่น HAProxy หรือ Nginx แล้วตั้งค่าให้ชี้ Traffic ไปยัง NodePort ของ Service ด้วยตัวเอง วิธีนี้ทนทานขึ้น แต่ก็ตามมาด้วยความยุ่งยากในการบริหารจัดการที่ต้องทำด้วยมือทั้งหมด

จะเห็นว่าการใช้ Load Balancer ของ Cloud Provider โดยตรงผ่าน Cloud Controller Manager นั้นเป็นวิธีที่ทนทานและเป็นอัตโนมัติที่สุด เรียกได้ว่าเป็นมาตรฐานของ Kubernetes ที่ทำงานบน Cloud เลยก็ว่าได้

## ขั้นตอนการติดตั้งและตั้งค่า

มาถึงส่วนที่ทุกคนรอคอย มาดูขั้นตอนกันเลย

### ขั้นตอนที่ 1: ปรับแต่ง k3s ให้พร้อมคุยกับภายนอก

สิ่งแรกที่ต้องทำคือบอกให้ K3s รู้ว่าเราจะใช้ Cloud Controller จากภายนอกนะ โดยต้องปิด Cloud Controller และ Load Balancer เริ่มต้นของมันออกไปก่อน ถ้าต้องการใช้ Traefik อาจจะเก็บไว้ก็ได้ แต่ผู้เขียนชอบใช้ NGINX Ingress มากกว่า ก็เลยจะปิด Traefik ด้วย

หากคุณติดตั้ง K3s ไปแล้ว ให้แก้ไขไฟล์ Service ของ k3s โดยตรง:

```shell
# แก้ไขไฟล์ /etc/systemd/system/k3s.service
# หรือ /etc/systemd/system/k3s-agent.service สำหรับ Worker Node

# เพิ่ม argument ท้ายบรรทัด ExecStart
ExecStart=/usr/local/bin/k3s server \
    # ... (argument อื่น ๆ) ... \
    '--disable=cloud-controller' \
    '--disable=servicelb' \
    '--disable=traefik' \
    '--cloud-provider=external'
```

จากนั้น Reload Daemon และ Restart Service:

```shell
sudo systemctl daemon-reload
sudo systemctl restart k3s
```

ทำซ้ำแบบนี้กับทุก ๆ โนด

แต่ถ้ากำลังจะติดตั้ง K3s ใหม่ สามารถระบุค่าเหล่านี้ผ่าน Environment Variable ได้เลย ซึ่งจะสะดวกกว่ามาก:

```shell
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable-cloud-controller --disable=servicelb --disable=traefik --kubelet-arg=cloud-provider=external" sh -
```

หลังจาก Restart หรือติดตั้งใหม่แล้ว จะพบว่าสามารถใช้คำสั่ง `kubectl` ได้ตามปกติ แต่ Pod ทั้งหมดจะมีสถานะเป็น Pending เนื่องจากโนดมี taint `node.cloudprovider.kubernetes.io/uninitialized=true:NoSchedule`

### ขั้นตอนที่ 2: สร้างไฟล์ตั้งค่า cloud.conf

ไฟล์นี้ใช้สำหรับบอก Cloud Controller Manager ว่าจะเชื่อมต่อกับ OpenStack API ได้อย่างไร โดยต้องระบุ URL, Username, Password, Project (Tenant) ID และข้อมูลอื่น ๆ ที่จำเป็น สามารถดูตัวอย่างไฟล์ได้จาก [Official Repository ของ Cloud Provider OpenStack](https://github.com/kubernetes/cloud-provider-openstack/blob/master/manifests/controller-manager/cloud-config) และแก้ไขให้ตรงกับ Public Cloud ที่เราใช้

#### ตัวอย่าง cloud.conf:

```editorconfig
[Global]
auth-url=https://keystone.example.com/v3
username=your_username   
password=your_password
region=TH-BKK   
tenant-id=9f6dbf311397409a92cbbc761c7f8865
domain-id=c6b00adf4ed04fc5a958121fadb0e401

[LoadBalancer]
subnet-id=8b219e59-d293-4ce6-b364-1c9c7eb1a34e
floating-network-id=2aa7a98d-ae38-4844-82d6-9ab5c2460b69
# Uncomment if your cloud provider doesn't set the default value
#flavor-id=fcbb978d-da0b-4ba6-8200-fd9228e6598e
#availability-zone=TH-BKK
# Uncomment if your cloud provider doesn't support creating LBs via API
#internal-lb=true
```

ข้อมูลต่าง ๆ ที่จำเป็นสามารถดูได้โดยใช้คำสั่ง OpenStack Client เช่น

* `openstack versions show`
* `openstack project list`
* `openstack project show`
* `openstack subnet list`
* `openstack network list`
* `openstack loadbalancer availabilityzone list`
* `openstack loadbalancer flavor list`

### ขั้นตอนที่ 3: สร้าง Secret ใน Kubernetes

เมื่อได้ไฟล์ `cloud.conf` มาแล้ว เราจะนำไฟล์นี้ไปสร้างเป็น Secret ใน Kubernetes เพื่อให้ Cloud Controller Manager นำไปใช้งานได้

```shell
sudo kubectl create secret -n kube-system generic cloud-config --from-file=cloud.conf
```

**สำคัญ:** secret จะต้องชื่อ `cloud-config` และมีไฟล์ `cloud.conf` อยู่ด้านใน

### ขั้นตอนที่ 4: ติดตั้ง OpenStack Cloud Controller Manager

เราสามารถติดตั้งได้จาก Manifest หรือ Helm Chart ของ Official Repository ซึ่งในที่นี้จะใช้ Manifest เพราะขี้เกียจติดตั้ง Helm:

```shell
sudo kubectl apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/refs/tags/v1.32.0/manifests/controller-manager/cloud-controller-manager-roles.yaml
sudo kubectl apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/refs/tags/v1.32.0/manifests/controller-manager/cloud-controller-manager-role-bindings.yaml
sudo kubectl apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/refs/tags/v1.32.0/manifests/controller-manager/openstack-cloud-controller-manager-ds.yaml
```

**หมายเหตุ:** ds ในชื่อไฟล์ย่อมาจาก DaemonSet ซึ่งหมายความว่า Controller Manager จะถูกติดตั้งลงบนโนดควบคุม (master node) ทุก ๆ โนด

### ขั้นตอนที่ 5: แก้ไข DaemonSet ให้รู้จัก k3s

**อัพเดต:** เห็นว่ามี [Pull Request ที่แก้ nodeSelector ให้เลือก `node-role.kubernetes.io/control-plane=true` แล้ว](https://github.com/kubernetes/cloud-provider-openstack/pull/2902) ถ้าใครใช้ openstack-cloud-controller-manager อาจจะไม่ต้องทำตามขั้นตอนนี้แล้วก็ได้

หลังจากติดตั้งในขั้นตอนที่แล้ว คุณอาจจะพบว่า... ไม่มี Pod ของ openstack-cloud-controller-manager ถูกสร้างขึ้นมาเลย!

ปัญหานี้เกิดจาก Manifest ของ DaemonSet ที่มีการระบุ nodeSelector ให้ Pod ทำงานบนโนดที่มี Label `node-role.kubernetes.io/master` ที่มีค่าเป็น **ค่าว่าง** ("") เท่านั้น

แต่ K3s กำหนด Label ให้โนดควบคุมเป็น `node-role.kubernetes.io/control-plane=true` ทำให้ Selector ไม่ตรงกัน เราจึงต้องแก้ไข DaemonSet ด้วยคำสั่ง patch (หรือใครสะดวกใช้คำสั่ง edit ก็ได้เหมือนกัน)

```shell
sudo kubectl patch daemonset openstack-cloud-controller-manager -n kube-system --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/nodeSelector/node-role.kubernetes.io~1control-plane", "value": "true"}]'
```

**คำอธิบาย:** `~1` ใน path คือการ escape เครื่องหมาย / ใน JSON Patch เรากำลังบอกให้ Kubernetes เปลี่ยน Selector ไปมองหา Label `node-role.kubernetes.io/control-plane` ที่มีค่าเป็น `true` แทน

หลังจากรันคำสั่งนี้แล้วไม่นาน Pod ของ Controller Manager ก็ควรจะถูกสร้างและอยู่ในสถานะ Running โดยเราสามารถตรวจสอบสถานะของ Pod ได้ด้วยคำสั่ง:

```shell
sudo kubectl get pods -n kube-system -l k8s-app=openstack-cloud-controller-manager -w
```

เมื่อ Pod ทำงานเรียบร้อยแล้ว taint `node.cloudprovider.kubernetes.io/uninitialized` บนโนดก็จะหายไป และ Pod อื่น ๆ ที่เคยมีสถานะ Pending ก็จะเริ่มทำงานได้ตามปกติ

### ขั้นตอนที่ 6: ตรวจสอบ Security Groups

เพื่อให้ Load Balancer ของ OpenStack สามารถส่ง Traffic เข้ามายัง Node ใน Cluster ได้ เราต้องแน่ใจว่า Security Group ของโนดทุกตัวได้เปิด Port ในช่วง NodePort (ปกติจะอยู่ในช่วงพอร์ต 30000-32767) สำหรับ TCP Traffic ที่มาจาก CIDR (Subnet) ของ Load Balancer

### ขั้นตอนที่ 7: ทดสอบการทำงาน

ถึงเวลาทดสอบแล้ว! เราจะลองสร้าง Deployment ง่าย ๆ และเปิดให้เข้าถึงได้ผ่าน Service ชนิด LoadBalancer

1. สร้าง Deployment ของ Nginx: `sudo kubectl create deployment nginx --image=nginx`
2. เปิด Service เป็นชนิด LoadBalancer: `sudo kubectl expose deployment nginx --port=80 --type=LoadBalancer`
3. รอดูผลลัพธ์: `sudo kubectl get service nginx -w`

รอสักครู่... ถ้าทุกอย่างถูกต้อง คุณจะเห็นว่าช่อง EXTERNAL-IP ของ Service nginx จะมี IP Address ปรากฏขึ้นมา ซึ่ง IP นี้คือ IP ของ Load Balancer ที่ถูกสร้างขึ้นใน OpenStack นั่นเอง! แต่ถ้าไม่เจอก็ไม่เป็นไร ลองอ่านหัวข้อด้านล่างดู

## ข้อจำกัดที่พบบน Cloud Provider และวิธีแก้ปัญหาเฉพาะหน้า

ถึงแม้จะทำตามขั้นตอนมาทั้งหมด แต่ผู้เขียนก็ยังเจอปัญหาเฉพาะตัวของ Cloud Provider ที่ใช้งานอยู่ (Nipa Cloud) ซึ่งอาจจะเป็นประโยชน์กับคนที่เจอสถานการณ์คล้าย ๆ กัน

1. **ต้องระบุ Flavor และ Availability Zone:** Load Balancer ที่ถูกสร้างขึ้นมาไม่ทำงาน หากไม่ระบุ flavor-id และ availability-zone ไปใน cloud.conf หรือใน Annotation ของ Service ไม่แน่ใจว่าเป็นเพราะทาง Cloud ลืมตั้งค่า Default ไว้หรือเปล่า
2. **ไม่สามารถสร้าง Floating IP ได้ เพราะ Description ยาวเกินไป:** Cloud Provider จำกัดความยาวของฟิลด์ description ของ Floating IP ไว้ที่ 20 ตัวอักษร แต่ openstack-cloud-controller-manager จะพยายามสร้าง Floating IP โดยระบุ description ที่มีความยาวไม่ต่ำกว่า 67 ตัวอักษร ทำให้เกิดข้อผิดพลาด HTTP 400 Bad Request วนลูปไม่รู้จบใน Log ของ Controller Manager (ทาง Provider รับทราบปัญหานี้แล้ว แต่ยังไม่มีกรอบเวลาแก้ไขที่ชัดเจน)
3. **Attach IP แล้ว Traffic ไม่เข้า:** หากลอง Attach Floating IP ผ่าน API หรือ OpenStack Client ด้วยตัวเอง Controller Manager จะหยุดพยายามสร้าง IP และคำสั่ง `kubectl get service` ก็จะแสดง External IP ถูกต้อง แต่... Traffic ไม่สามารถเข้าได้!

จากข้อจำกัดข้างต้น (ซึ่งอาจจะเป็นความอ่อนประสบการณ์ของผู้เขียนเองก็ได้) จึงใช้วิธีแก้ปัญหาเฉพาะหน้าไปก่อน คือ เพิ่มการตั้งค่า internal-lb=true เข้าไปในไฟล์ cloud.conf เพื่อบอกให้ Controller Manager สร้างแค่ Load Balancer แต่ไม่ต้องพยายามสร้าง Floating IP เมื่อสร้าง Load Balancer เสร็จแล้วก็เข้าไปที่หน้า Web UI ของ Cloud Provider เพื่อ Attach Floating IP ให้กับ Load Balancer ที่ถูกสร้างขึ้นมาด้วยมือ

วิธีนี้ทำให้สามารถสร้าง Load Balancer ได้แบบ (กึ่ง) อัตโนมัติ เพราะต้องเข้าไปจัดการส่วนที่เหลือใน Web UI ด้วยตนเอง

## บทสรุป

การเชื่อมต่อ K3s เข้ากับ OpenStack Load Balancer เป็นประสบการณ์ที่ท้าทายแต่ก็ทำให้เข้าใจสถาปัตยกรรมเบื้องหลังของ Kubernetes บน Cloud มากขึ้น แม้จะต้องเจอกับปัญหาที่ได้กล่าวไปข้างต้นก็ตาม

สุดท้ายนี้ ขอย้ำคำเตือนเดิมอีกครั้งว่า หากไม่จำเป็นจริง ๆ การใช้บริการ Managed Kubernetes ของ Cloud Provider ยังคงเป็นทางเลือกที่ดีและง่ายที่สุด แต่ถ้าคุณเป็นสายลุย ชอบเรียนรู้ และไม่กลัวปัญหา ก็จัดเลย
