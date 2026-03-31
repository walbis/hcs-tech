# OpenShift Egress IP vs Huawei Cloud CCE Elastic IP (EIP) Karsilastirmasi

## 1. Tanim ve Temel Kavram

### OpenShift Egress IP
OpenShift Egress IP, OVN-Kubernetes CNI eklentisi tarafindan yonetilen bir ozelliktir. Cluster icerisindeki bir veya daha fazla namespace'e (ya da belirli pod'lara) **sabit bir kaynak IP adresi** atayarak, dis servislere giden trafigin tutarli bir IP adresinden cikmasi saglanir. Bu ozellik ozellikle dis sistemlerin paket filtreleme ile erisimi belirli IP adreslerine sinirladigi senaryolarda kullanilir.

> **Kaynak:** [OpenShift Docs - Configuring Egress IPs (OVN-Kubernetes)](https://docs.openshift.com/container-platform/4.15/networking/ovn_kubernetes_network_provider/configuring-egress-ips-ovn.html)

### Huawei Cloud CCE Elastic IP (EIP)
Elastic IP (EIP), Huawei Cloud'da **statik public IP adresleri ve olceklenebilir bant genisligi** saglayan bir servistir. CCE (Cloud Container Engine) baglaminda, pod'larin veya node'larin internete erisimini saglamak icin kullanilir. Sadece ozel IP adresine sahip kaynaklar dogrudan internete erisamez; EIP baglayarak bu erisim mumkun hale gelir.

> **Kaynak:** [Huawei Cloud - EIP Product Description](https://support.huaweicloud.com/intl/en-us/productdesc-eip/overview_0001.html), [Huawei Cloud - CCE Pod Internet Access](https://support.huaweicloud.com/intl/en-us/usermanual-cce/cce_10_0400.html)

---

## 2. Calisma Mekanizmasi

### OpenShift Egress IP

| Ozellik | Detay |
|---------|-------|
| **Mekanizma** | SNAT (Source NAT) kullanarak pod trafigininin kaynak IP adresini degistirir |
| **Yonetim** | `EgressIP` Custom Resource (CR) ile Kubernetes-native yonetim |
| **Node Secimi** | `k8s.ovn.org/egress-assignable=""` etiketi ile isaretlenen node'lara otomatik atama |
| **Trafik Akisi** | Secilen pod'larin trafigi, egress IP atanmis node uzerinden yonlendirilir |
| **Failover** | Egress IP tasiyan node basarisiz olursa, IP otomatik olarak baska bir uygun node'a tasinir (~2 saniye) |

**Detayli Trafik Akisi (OVN-Kubernetes):**
1. Pod, cluster disi bir hedefe trafik gonderir
2. OVN-Kubernetes, pod'un EgressIP selector'una uygun oldugunu tespit eder
3. OVN logical router **reroute policies** (oncelik 102) trafigi egress node'a yonlendirir
4. Egress node'da OVN northbound database'deki **SNAT kurali** kaynak IP'yi pod IP'den egress IP'ye degistirir
5. Trafik, egress IP kaynak adresi ile node'dan cikar

**OVN SNAT Kaydi Ornegi:**
```
_uuid: 385dd68c-62a2-4394-a3ef-6b86afc3ed43
external_ip: "192.168.126.100"
logical_ip: "10.129.3.232"
type: "snat"
```

> **Kaynak:** [OVN-Kubernetes EgressIP Docs](https://ovn-kubernetes.io/features/cluster-egress-controls/egress-ip/)

### Huawei Cloud CCE EIP

| Ozellik | Detay |
|---------|-------|
| **Mekanizma** | Dogrudan IP baglama veya NAT Gateway uzerinden SNAT |
| **Yonetim** | Huawei Cloud konsolu, API veya CCE annotation'lari ile |
| **Baglama Yontemi** | Node'a EIP baglama (VPC/Tunnel modeli) veya Pod'a dogrudan EIP baglama (Cloud Native 2.0) |
| **Trafik Akisi** | Pod trafigi baglanmis EIP uzerinden veya NAT Gateway SNAT kurallari ile cikar |
| **Failover** | Cloud altyapisi tarafindan yonetilir; EIP kaynak bazinda baglanir |

---

## 3. Konfigurasyion Yontemleri

### OpenShift Egress IP

**Adim 1 - Node Etiketleme:**
```bash
oc label nodes <node_name> k8s.ovn.org/egress-assignable=""
```

**Adim 2 - EgressIP Objesi Olusturma:**
```yaml
apiVersion: k8s.ovn.org/v1
kind: EgressIP
metadata:
  name: egress-project
spec:
  egressIPs:
    - 192.168.127.10
    - 192.168.127.11
  namespaceSelector:
    matchLabels:
      env: production
  podSelector:           # Opsiyonel
    matchLabels:
      app: web
```

**Onemli Alanlar:**
- `spec.egressIPs`: Atanacak IP adresleri dizisi
- `spec.namespaceSelector`: Hedef namespace'leri secen label selector
- `spec.podSelector`: (Opsiyonel) Belirli pod'lari secen label selector

**Node Atama Kurallari (OVN-Kubernetes):**
- Bir egress IP ayni anda **yalnizca bir node'a** atanir
- Egress IP'ler uygun node'lar arasinda **esit dagilimla** dengelenir
- Birden fazla egress IP tanimlanmissa, **tek bir node birden fazla IP barindirmaz** (HA icin)
- Yalnizca **worker node'lar** uygundur - control plane node'lar haric tutulur

**Cloud Provider Node Annotation'i:**
Cloud platformlarinda `cloud.network.openshift.io/egress-ipconfig` annotation'i node basina kapasite ve subnet bilgisi saglar.

**Legacy OpenShift SDN Konfigurasyonu (oc patch):**
```bash
# Namespace'e egress IP atama
oc patch netnamespace <proje_adi> --type=merge \
  -p '{"egressIPs": ["192.168.1.100"]}'

# Node'da izin verilen CIDR araligini tanimlama (otomatik atama)
oc patch hostsubnet <node_adi> --type=merge \
  -p '{"egressCIDRs": ["192.168.1.0/24"]}'

# veya Manuel atama
oc patch hostsubnet <node_adi> --type=merge \
  -p '{"egressIPs": ["192.168.1.100"]}'
```

> **Kaynak:** [OpenShift Docs - EgressIP Object](https://github.com/openshift/openshift-docs/blob/main/modules/nw-egress-ips-object.adoc), [Red Hat Blog - How to Enable Static Egress IP](https://www.redhat.com/en/blog/how-enable-static-egress-ip-red-hat-openshift-container-platform)

### Huawei Cloud CCE EIP

**Yontem 1 - Node'a EIP Baglama (VPC/Tunnel Ag Modeli):**
- Huawei Cloud konsolundan node'un bulundugu ECS'ye EIP baglanir
- Pod'lar, node'un internet baglantisini paylasir
- Erisim formati: `<Node_EIP>:<NodePort>` (NodePort araligi: 30000-32767)

**Yontem 2 - Pod'a EIP Baglama (Cloud Native 2.0 / CCE Turbo):**
- CCE Turbo Cluster gerektirir
- Pod'lar VPC elastic network interface (ENI) kullanir, EIP dogrudan pod'un ag arayuzune baglanir
- NAT veya tunnel encapsulation overhead'i yoktur

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    yangtse.io/pod-with-eip: "true"
    yangtse.io/eip-bandwidth-size: "5"          # Mbit/s, varsayilan: 5
    yangtse.io/eip-network-type: 5_bgp          # Secenekler: 5_bgp, 5_union, 5_sbgp
    yangtse.io/eip-charge-mode: bandwidth       # Secenekler: bandwidth, traffic
    yangtse.io/eip-bandwidth-name: "my-bw"
spec:
  containers:
    - name: app
      image: nginx
```

Mevcut bir EIP baglamak icin:
```yaml
annotations:
  yangtse.io/eip-id: "<mevcut_eip_id>"
```

**Yontem 3 - NAT Gateway + SNAT (Onerilen):**
- Ayni VPC'de NAT Gateway olusturulur
- EIP, NAT Gateway'e baglanir
- SNAT kurallari subnet CIDR blogu bazinda tanimlanir
- Tum pod'lar paylasimli EIP uzerinden internete cikar
- Yuksek esli baglanti sayisini destekler (NAT Gateway basina 20 Gbit/s'e kadar)

**Yontem 4 - LoadBalancer Service ile EIP (ELB):**
```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    kubernetes.io/elb.autocreate: '{"type":"public","bandwidth_size":5,"eip_type":"5_bgp"}'
spec:
  type: LoadBalancer
```

> **Kaynak:** [Huawei Cloud - CCE Pod Internet Access](https://support.huaweicloud.com/intl/en-us/usermanual-cce/cce_10_0400.html), [Huawei Cloud - EIP for Pod in CCE Turbo](https://support.huaweicloud.com/intl/en-us/usermanual-cce/cce_10_0734.html)

---

## 4. Benzerlikler

| # | Ortak Ozellik | Aciklama |
|---|---------------|----------|
| 1 | **SNAT Kullanimi** | Her iki sistem de kaynak IP adresini donusturmek icin SNAT mekanizmasi kullanir |
| 2 | **Sabit Cikis IP'si** | Dis servislere sabit/tutarli bir IP adresi ile erisim saglar |
| 3 | **Node Bazli Yonlendirme** | Trafik belirli node'lar uzerinden yonlendirilir |
| 4 | **Firewall/Whitelist Uyumlulugu** | Dis sistemlerde IP bazli erisim kontrolu (whitelist) icin idealdir |
| 5 | **Olceklenebilirlik** | Birden fazla IP adresi atanabilir; yuk dagitimi mumkundur |
| 6 | **Kubernetes Entegrasyonu** | Her ikisi de Kubernetes/konteyner ortamlarinda calisir |

---

## 5. Farkliliklar

| Kriter | OpenShift Egress IP | Huawei Cloud CCE EIP |
|--------|---------------------|----------------------|
| **Kapsam** | Cluster-internal networking ozelligi | Cloud-level networking servisi |
| **Yonetim Katmani** | Kubernetes CRD (EgressIP objesi) | Cloud konsolu, API + Kubernetes annotation |
| **IP Atama** | Mevcut node IP havuzundan otomatik atama | Cloud'dan bagimsiz EIP satin alma/tahsis |
| **Granularite** | Namespace + Pod label selector ile ince ayar | Node-level veya Pod-level (Cloud Native 2.0) |
| **Failover** | OVN-Kubernetes otomatik IP migrasyonu (node arasi) | Cloud altyapisi tarafindan yonetilir |
| **Maliyet Modeli** | Cluster kaynagi - ek maliyet yok | EIP + bant genisligi icin ayri faturalandirma |
| **Bant Genisligi** | Cluster/node ag kapasitesine bagli | EIP bazinda olceklenebilir bant genisligi (Static/Dynamic/Premium BGP) |
| **Platform Bagimliligi** | OpenShift (OVN-Kubernetes veya OpenShift SDN) | Huawei Cloud CCE (VPC, Tunnel, Cloud Native 2.0 ag modelleri) |
| **NAT Gateway** | Yok - dogrudan node uzerinden SNAT | NAT Gateway + SNAT kurallari ile paylasimli cikis |
| **Pod-Level IP** | Pod selector ile ayni egress IP'yi paylasir | Cloud Native 2.0'da pod basina bagimsiz EIP |
| **IPv6 Destegi** | Dual-stack (IPv4 + IPv6) destekler | EIP tipi ve bolgeye bagli |
| **Multi-Tenant** | Namespace bazinda izolasyon | VPC/Subnet bazinda izolasyon |
| **Public Cloud Limitleri** | AWS/GCP/Azure'da node basina IP limiti vardir | Huawei Cloud EIP kotasina tabidir |

---

## 6. Platform Destegi

### OpenShift Egress IP
- Bare Metal
- VMware vSphere
- AWS, GCP, Azure
- IBM Z/LinuxONE, IBM Power
- Nutanix, OpenStack
- *ROSA platformunda bazi kisitlamalar mevcuttur*

> **Kaynak:** [OpenShift Docs - Egress IP Platform Support](https://github.com/openshift/openshift-docs/blob/main/modules/nw-egress-ips-about.adoc)

### Huawei Cloud CCE EIP
- CCE Managed Cluster (VPC/Tunnel ag modeli - node EIP)
- CCE Turbo Cluster (Cloud Native 2.0 - pod EIP)
- NAT Gateway tum CCE cluster tiplerinde desteklenir

> **Kaynak:** [Huawei Cloud - CCE Pod Internet Access](https://support.huaweicloud.com/intl/en-us/usermanual-cce/cce_10_0400.html)

---

## 7. Sinirlamalar ve Dikkat Edilmesi Gerekenler

### OpenShift Egress IP
- **Control plane node'larinda desteklenmez**
- Uygulama ve ingress pod'lari ayni node'da ise, route kaynakli istekler icin egress IP uygulanmaz
- Egress IP'ler node'un **birincil ag arayuzunde ek IP** olarak uygulanir; ayni subnet'te olmalidir (bare metal secondary NIC harici)
- Linux ag konfigurasyion dosyalarinda tanimlanmamalidir
- Public cloud'larda node basina atanabilecek IP siniri vardir (AWS: 30-50, GCP: 10/node, Azure: 256/NIC)
- Tum trafigi tek bir node'a yonlendirmek performans sorunlarina yol acabilir
- Hatali label selector tum namespace'lerin cikis IP'sini degistirebilir
- **OpenShift SDN:** Namespace'ler arasi egress IP paylasimi desteklenmez; otomatik ve manuel atama ayni node'da karistirilamaz
- **OpenShift SDN:** Egress IP tanimli ancak hicbir node tarafindan barindirılmayan bir namespace'in **cikis trafigi duser (drop)**
- **RHOSP:** Failover sirasinda Neutron reservation port yeniden olusturulur, floating IP iliskisi kopar - manuel yeniden atama gerekir
- Bagimsiz ovn-controller islemesi nedeniyle, pod trafigi egress node'a SNAT kurallarindan once ulasabilir (kisa sureli race condition)

### Huawei Cloud CCE EIP
- Her EIP ayni anda yalnizca **bir kaynaga** baglanabilir
- EIP, yalnizca **ayni bolgedeki** kaynaklara baglanir
- Pod seviyesinde EIP yalnizca **Cloud Native 2.0 (CCE Turbo)** ag modelinde desteklenir
- SNAT kurallari **CIDR blogu bazinda** calisir; pod bazinda granularite saglamaz
- EIP bant genisligi siniri mevcuttur (inbound <=10 Mbit/s ise 10 Mbit/s'ye kadar)
- Pod EIP annotation'lari **pod olusturulduktan sonra degistirilemez** - pod yeniden olusturulmalidir
- Otomatik tahsis edilen EIP manuel silinirse pod'un agi bozulur, pod yeniden olusturulmalidir
- Pod basina **maksimum 1 EIP** atanabilir
- ECS sub-ENI baglama hizi limiti: tenant bazinda **600 pod/dakika**
- BMS ENI baglama suresi: **20-30 saniye**, node basina **3 esli** pod olusturma limiti
- Ek maliyet: EIP + bant genisligi kullanim ucreti

---

## 8. Huawei CCE: NAT Gateway vs Dogrudan EIP Karsilastirmasi

| Kriter | NAT Gateway (SNAT) | Dogrudan EIP (Node/Pod) |
|--------|---------------------|-------------------------|
| **Paylasim Modeli** | Cok sayida pod tek/birden fazla EIP'yi paylasir | Her node/pod kendi EIP'sine sahiptir |
| **Esli Baglanti** | Yuksek - buyuk olcekli esli baglanti icin tasarlanmis | Bireysel EIP bant genisligi ile sinirli |
| **Maks Bant Genisligi** | NAT Gateway basina 20 Gbit/s'e kadar | EIP bazinda bant genisligi limiti |
| **Ag Modeli** | Tumu (Tunnel, VPC, Cloud Native 2.0) | Node EIP: tum modeller; Pod EIP: yalnizca Cloud Native 2.0 |
| **Maliyet Verimliligi** | Cok pod icin tek EIP (ekonomik) | Pod/node basina EIP (olcekte pahali) |
| **Oncelik Kurali** | Hem EIP hem NAT Gateway varsa, trafik EIP uzerinden cikar | - |
| **Kisitlama** | VPC basina bir NAT Gateway; subnet basina bir SNAT kurali | Pod EIP: pod basina maks 1 EIP |

> **Kaynak:** [Huawei Cloud - CCE Pod Internet Access](https://support.huaweicloud.com/intl/en-us/usermanual-cce/cce_10_0400.html)

---

## 9. Huawei CCE: Node EIP vs Pod EIP

| Kriter | Node'a EIP Baglama | Pod'a EIP Baglama |
|--------|-------------------|-------------------|
| **Ag Modeli** | Tumu (VPC, Tunnel, Cloud Native 2.0) | Yalnizca Cloud Native 2.0 (CCE Turbo) |
| **Granularite** | Node seviyesi (tum pod'lar paylasir) | Pod seviyesi (pod basina ozel) |
| **Trafik Yolu** | Node EIP → NodePort → kube-proxy → Pod (olasi cross-node hop) | Dogrudan pod ENI uzerinden (NAT/tunnel yok) |
| **Performans** | Route yonlendirme nedeniyle performans kaybi olabilir | Daha yuksek performans, encapsulation overhead yok |
| **EIP Yasam Dongusu** | ECS konsolunda bagimsiz yonetilir | Otomatik tahsis: pod ile silinir; Mevcut EIP: pod silindikten sonra korunur |
| **Guvenlik** | Tum node internete acilir | Pod bazinda izole; pod bazinda security group mumkun |
| **Annotation** | Yok (ECS/VPC konsolundan yapilandirilir) | `yangtse.io/pod-with-eip`, `yangtse.io/eip-id` vb. |
| **Kullanim Alani** | Hizli NodePort erisimi, test ortamlari | Ozel public IP gerektiren production workload'lar |

> **Kaynak:** [Huawei Cloud - EIP for Pod in CCE Turbo](https://support.huaweicloud.com/intl/en-us/usermanual-cce/cce_10_0734.html)

---

## 10. Teknik Spesifikasyonlar

### OpenShift Egress IP

| Spesifikasyon | Deger |
|---------------|-------|
| **CNI Eklentisi** | OVN-Kubernetes (varsayilan OCP 4.6+) veya OpenShift SDN (legacy) |
| **CRD / API Grubu** | `EgressIP` (`k8s.ovn.org/v1`) |
| **Node Etiketi** | `k8s.ovn.org/egress-assignable=""` |
| **Cloud Annotation** | `cloud.network.openshift.io/egress-ipconfig` |
| **Host CIDR Annotation** | `k8s.ovn.org/host-cidrs` (secondary NIC icin, yalnizca bare metal) |
| **IP Protokolu** | IPv4 + IPv6 (Dual-Stack) |
| **OVN Health Check Portu** | 9107 |
| **Failover Gecikmesi** | ~2 saniye (OCPBUGS-32161 duzeltmesi sonrasi) |
| **P99 Baslangic Gecikmesi (24K EIP)** | 8.4 ms |
| **Olcek Testi** | 24.000 egress IP, 120 node cluster, worker basina 200 EIP |
| **AWS IP Limiti** | Instance tipine gore 30-50 (degisken) |
| **GCP IP Limiti** | Node basina 10 alias, VPC basina ~15.000 |
| **Azure IP Limiti** | NIC basina 256, sanal ag basina 65.536 |
| **Scope** | Cluster-scoped kaynak |
| **Minimum Versiyon** | OCP 4.10 (cloud destegi), OCP 4.6 (OVN-Kubernetes varsayilan) |
| **OpenShift SDN Otomatik Mod** | Namespace basina 1 egress IP |
| **OpenShift SDN Manuel Mod** | Namespace basina birden fazla egress IP |
| **OVN-Kubernetes** | Namespace'ler arasi IP paylasimi destekli |
| **Ikincil NIC Destegi** | Yalnizca bare metal |

> **Kaynak:** [Red Hat Developer Blog - Egress IP Scale Testing](https://developers.redhat.com/blog/2024/06/21/egress-ip-scale-testing-openshift-container-platform)

### Huawei Cloud CCE EIP

| Spesifikasyon | Deger |
|---------------|-------|
| **EIP Tipleri** | `5_bgp` (Dynamic BGP), `5_sbgp` (Static BGP), `5_union`, `5_telcom` |
| **Varsayilan Bant Genisligi** | 5 Mbit/s |
| **Maks Bant Genisligi (LB)** | 1-2000 Mbit/s (bolgeye bagli) |
| **NAT Gateway Maks Bant Genisligi** | 20 Gbit/s |
| **Ucretlendirme Modlari** | Bant genisligine gore, trafige gore |
| **Pod Basina Maks EIP** | 1 |
| **Pod EIP Min Cluster Versiyonlari** | v1.19.16-r20, v1.21.10-r0, v1.23.8-r0, v1.25.3-r0+ |
| **Mevcut EIP Baglama Min Versiyonlari** | v1.23.16-r0, v1.25.11-r0, v1.27.8-r0, v1.28.6-r0, v1.29.2-r0+ |
| **Cloud Native 2.0 Maks Olcek** | 2.000 ECS node, 100.000 pod/cluster |
| **ENI Olusturma Hizi (ECS)** | ~1 saniye |
| **ENI Olusturma Hizi (BMS)** | 20-30 saniye |
| **NodePort Araligi** | 30000-32767 |

> **Kaynak:** [Huawei Cloud - EIP for Pod in CCE Turbo](https://support.huaweicloud.com/intl/en-us/usermanual-cce/cce_10_0734.html)

---

## 11. Kullanim Senaryolari Karsilastirmasi

| Senaryo | OpenShift Egress IP | Huawei CCE EIP |
|---------|---------------------|----------------|
| Dis API'ye sabit IP ile erisim | Namespace/Pod selector ile EgressIP atama | NAT Gateway SNAT veya node EIP |
| Coklu tenant izolasyonu | Her namespace'e farkli egress IP | Her VPC/subnet'e farkli EIP/NAT |
| Yuksek trafik / cok sayida baglanti | Birden fazla egress IP ve node | NAT Gateway + SNAT (onerilen) |
| Pod basina bagimsiz public IP | Desteklenmez (paylasimli egress IP) | CCE Turbo'da pod-level EIP |
| Hybrid/Multi-cloud | OpenShift destekli tum platformlarda | Yalnizca Huawei Cloud |

---

## 12. Failover Davranisi Karsilastirmasi

| Kriter | OpenShift Egress IP | Huawei CCE EIP |
|--------|---------------------|----------------|
| **Mekanizma** | OVN port 9107 uzerinden health check; basarisiz node tespit edilir | Cloud altyapisi tarafindan yonetilir |
| **Failover Suresi** | ~2 saniye | Cloud SLA'ya bagli |
| **Otomatik Yeniden Atama** | Evet - IP baska uygun node'a tasinir | Pod EIP: pod yeniden zamanlandiginda EIP korunur (mevcut EIP); otomatik EIP: yeniden olusturulur |
| **Yeniden Dengeleme** | Node geri geldiginde IP'ler yeniden dengelenir | Uygulanmaz - EIP kaynak bazinda sabit |
| **Birden Fazla IP** | Pod ilk IP'yi kullanir; basarisiz olursa sonrakine gecer | Her pod/node tek EIP; NAT Gateway'de EIP degisikligi gerekmez |

> **Kaynak:** [Red Hat Developer Blog - Egress IP Scale Testing](https://developers.redhat.com/blog/2024/06/21/egress-ip-scale-testing-openshift-container-platform)

---

## 13. Ozet

**OpenShift Egress IP**, Kubernetes-native bir cozum olarak cluster icinde tanimlanan CRD'ler araciligiyla yonetilir ve ozellikle coklu platform destegi, namespace/pod bazinda ince granularite ve otomatik failover sunmasi ile one cikar. Ek bir cloud servisi gerektirmez.

**Huawei Cloud CCE EIP**, cloud-managed bir servis olarak daha esnek bant genisligi yonetimi, NAT Gateway ile olceklenebilir SNAT, ve CCE Turbo'da pod basina bagimsiz public IP atama gibi ozellikler sunar. Ancak ek maliyet ve platform bagimliligini beraberinde getirir.

Her iki cozum de ayni temel ihtiyaci karsilar: **konteyner trafiginiin dis dunyaya sabit ve kontrol edilebilir bir IP adresi ile cikmasi**. Secim, kullanilan platforma, maliyet beklentilerine ve mimari gereksinimlere gore yapilmalidir.

---

## Kaynaklar

### OpenShift Egress IP
1. [OpenShift Documentation - Configuring Egress IPs (OVN-Kubernetes)](https://docs.openshift.com/container-platform/4.15/networking/ovn_kubernetes_network_provider/configuring-egress-ips-ovn.html)
2. [OpenShift Docs Source - nw-egress-ips-about.adoc](https://github.com/openshift/openshift-docs/blob/main/modules/nw-egress-ips-about.adoc)
3. [OpenShift Docs Source - nw-egress-ips-object.adoc](https://github.com/openshift/openshift-docs/blob/main/modules/nw-egress-ips-object.adoc)
4. [OpenShift Docs Source - nw-egress-ips-node.adoc](https://github.com/openshift/openshift-docs/blob/main/modules/nw-egress-ips-node.adoc)
5. [Red Hat Developer Blog - Egress IP Scale Testing in OpenShift](https://developers.redhat.com/blog/2024/06/21/egress-ip-scale-testing-openshift-container-platform)
6. [Red Hat Blog - How to Enable Static Egress IP](https://www.redhat.com/en/blog/how-enable-static-egress-ip-red-hat-openshift-container-platform)
7. [OVN-Kubernetes Upstream Docs - EgressIP](https://ovn-kubernetes.io/features/cluster-egress-controls/egress-ip/)
8. [OVN-Kubernetes EgressIP Source (GitHub)](https://github.com/openshift/ovn-kubernetes/blob/master/docs/features/cluster-egress-controls/egress-ip.md)

### Huawei Cloud CCE EIP
9. [Huawei Cloud - EIP Product Description](https://support.huaweicloud.com/intl/en-us/productdesc-eip/overview_0001.html)
10. [Huawei Cloud - CCE Pod Internet Access](https://support.huaweicloud.com/intl/en-us/usermanual-cce/cce_10_0400.html)
11. [Huawei Cloud - Configuring EIP for Pod in CCE Turbo](https://support.huaweicloud.com/intl/en-us/usermanual-cce/cce_10_0734.html)
12. [Huawei Cloud - Cloud Native Network 2.0](https://support.huaweicloud.com/intl/en-us/usermanual-cce/cce_10_0284.html)
13. [Huawei Cloud - CCE Networking Overview](https://support.huaweicloud.com/intl/en-us/usermanual-cce/cce_10_0249.html)
