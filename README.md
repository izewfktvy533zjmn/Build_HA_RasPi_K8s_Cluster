# 高可用性ラズパイKubernetesクラスターの構築

## 概要
このリポジトリはRaspberry Piで高可用性Kubernetesクラスターを構築するための手順とビルドスクリプトをまとめたものです。  

<img src="./images/kubernetes_cluster_on_raspberrypi.jpg" width=100% alt="High Availability Kubernetes Cluster on Raspberry Pi">





## アーキテクチャー
高可用性ラズパイKubernetesクラスターのアーキテクチャを下記に示します。  
本アーキテクチャは大きく **3つのコンポーネント** で構成されています。  
アクティブ/アクティブ構成で冗長化された **アクティブ/アクティブ冗長化構成Kubernetesマスターノード群** と **Kubernetesワーカーノード群** で構築されたクラスター、そのクラスターのkube-apiserverに対するAPIリクエストを複数のマスターノードに分散させるためのアクティブ/スタンバイ構成で冗長化された **アクティブ/スタンバイ冗長化構成ロードバランサ** です。  

<img src="./images/architecture.png" width=100% alt="High Availability Kubernetes Cluster Architecture"><br>



### Kubernetesワーカーノード群
Kubernetesワーカーノード群はKubernetesマスターノード群によってスケジューリングされたPodを動作させます。  

アクティブ/アクティブ冗長化構成Kubernetesマスターノード群の構築、アクティブ/スタンバイ冗長化構成ロードバランサの構築が完了したらワーカーノードをクラスターに参加させ、ワーカーノード群を形成します。  

Kubernetesワーカーノード群の構築に関しましては高可用性Kubernetesクラスターの構築に直接的に関与しないため、ワーカーノードをクラスターに参加させる方法、クラスターの動作検証の方法などについては言及しません。  
ワーカノードをクラスターに参加させる方法に関しましては[ワーカーノードの設定](https://github.com/izewfktvy533zjmn/Build_RasPi_Kubernetes_Cluster/blob/master/README.md#%E3%83%AF%E3%83%BC%E3%82%AB%E3%83%BC%E3%83%8E%E3%83%BC%E3%83%89%E3%81%AE%E8%A8%AD%E5%AE%9A)をクラスターの動作検証の方法に関しましては[Kubernetesクラスターの動作検証方法](https://github.com/izewfktvy533zjmn/Build_RasPi_Kubernetes_Cluster/blob/master/README.md#kubernetes%E3%82%AF%E3%83%A9%E3%82%B9%E3%82%BF%E3%83%BC%E3%81%AE%E5%8B%95%E4%BD%9C%E6%A4%9C%E8%A8%BC%E6%96%B9%E6%B3%95)を参考にして下さい。  

<center><img src="./images/kubernetes_woker_nodes.png" width=50% alt="Redundant Kubernetes Master Nodes"></center><br>



### アクティブ/アクティブ冗長化構成Kubernetesマスターノード群
Kubernetesマスターノードはクラスターの操作機能の提供(kube-apiserver)、クラスターに関する情報の管理(etcd)、PodのスケジューリングやPodを適切なワーカーノードに再割り当てるための管理(kube-scheduler)、ワーカーノードの状態監視(kube-controller-manager)といったKubernetesクラスター全体の管理を行います。  

シングルコントロールプレーンクラスターでは1台のマスターノードが故障した場合、クラスターの管理を継続させることができません。  
システムを構成する部分のうち、そこに障害が発生するとシステム全体が停止してしまう部分を **単一障害点 (SPOF : Single Point of Failure)** と呼びますが、シングルコントロールプレーンクラスターではこの1台のマスターノードが単一障害点となります。  
そこで、コントロールプレーンを複数用意することでマスターノードに対して冗長化構成をとり、あるマスターノードが故障した場合でも他のマスターノードがクラスターの管理を継続することを可能にすることで単一障害点をなくし、高可用性クラスターを実現させることができます。  

本アーキテクチャでのKubernetesマスターノード群は **アクティブ/アクティブ構成** を採ることで冗長化させます。  
アクティブ/アクティブ構成とは、同じ機能を持つシステムコンポーネントを複数用意し、これらのコンポーネントを常に同時に稼動させるようなシステム構成をとる冗長化手法の一つです。  
このような冗長化手法を採ることで、各コンポーネントに対する負荷分散がもたらすシステムパフォーマンスの向上や、あるコンポーネントが機能を継続することができなくなった場合でも他の同じ機能を持つコンポーネントが代わりに機能を提供することができるため、システムの可用性を高めることができます。  

<img src="./images/redundant_kubernetes_master_nodes.png" width=100% alt="Redundant Kubernetes Master Nodes"><br>

Kubernetesマスターノード群に対してアクティブ/アクティブ構成をとる場合、下記の項目に関して対処する必要があります。

1. クラスターに関する情報をマスターノード間で共有する必要がある
2. クラスターの操作に関するAPIリクエストを各マスターノードのkube-apiserverに振り分ける必要がある

本アーキテクチャでは上記の項目に関する対処方法に対して、以下の方法を採っています。  

1.に関しては、マスターノード同士がetcdクラスター上に情報を保存することでKubernetesクラスターに関する情報を共有させます。  
etcdクラスターを構築する方法として、[Kubernetes公式ドキュメントに記載されている方法](https://kubernetes.io/ja/docs/setup/production-environment/tools/kubeadm/high-availability/)があります。  
本Kubernetesクラスターの構築にあたって、公式ドキュメントに記載されているetcdクラスターを構築する方法からetcdのメンバーとコントロールプレーンを結合した構成である積み重なったコントロールプレーンを使う方法を採用しています。  

2.に関しては、負荷分散によってAPIリクエストの振り分けを行うことができるためロードバランサを導入することで対処することが可能ですが、高可用性を実現させるためにはこの導入するロードバランサに対していくつかの制約があります。

- マスターノードが正常に稼働しているか確認することができる監視機能がなければならない
- 障害が発生したマスターノードに対しては処理を振り分けないようにしなければならない
- ロードバランサ自体が単一障害点とならないように冗長化しなければならない

本アーキテクチャでは、上記の制約を満たすために **アクティブ/スタンバイ冗長化構成ロードバランサ** を導入することで対処しています。



### アクティブ/スタンバイ冗長化構成ロードバランサ
**アクティブ/スタンバイ冗長化構成ロードバランサ** はアクティブ/アクティブ冗長化構成Kubernetesマスターノード群に対するクラスター操作のAPIリクエストを各マスターノードのkube-apiserverに振り分ける役割を担っています。  

本アーキテクチャにおける高可用性を実現させるための制約を満たすために、負荷分散機能やHTTPリバースプロキシとしての機能、サーバに対するの監視機能、ヘルスチェック機能による非稼働中のサーバに対するリクエスト振り分けの停止などを提供するオープンソースのTCP/HTTPプロキシソフトウェアである[**HAProxy**](http://www.haproxy.org/)を利用することでロードバランサを構築します。  

また、本アーキテクチャにおいて、高可用性を実現させるためには **"ロードバランサ自体が単一障害点とならないように冗長化しなければならない"** という制約がありました。  
冗長化手法に関しては、[アクティブ/アクティブ冗長化構成Kubernetesマスターノード群](#アクティブ/アクティブ冗長化構成Kubernetesマスターノード群)にで **アクティブ/アクティブ構成** という手法を説明しました。  
これに対して、導入するロードバランサでは **アクティブ/スタンバイ構成** という冗長化手法を採っています。  
アクティブ/スタンバイ構成とは、同じ機能を持つシステムコンポーネントを複数用意し、ある一つのコンポーネントだけを稼働状態とし、他のコンポーネントは待機状態にしておくようなシステム構成をとる冗長化手法の一つです。  
このような冗長化手法は、何らかの原因によって稼動状態のシステムコンポーネントに障害が発生したとき、待機状態のシステムコンポーネントを稼動状態へと切り替えることによってシステムの稼働を継続させることができるため、システムの可用性を高めることができます。

本アーキテクチャでは、仮想IPアドレス(VIP)の引き継ぎ機能やVRRPによるサーバの死活監視機能、サービスのヘルスチェック機能などを提供するソフトウェアである[**Keepalived**](https://www.keepalived.org/)を利用することで **アクティブ/スタンバイ冗長化構成ロードバランサ** を構築します。

<img src="./images/redundant_load_balancers.png" width=100% alt="Redundant Load Balancers"><br>





## ネットワーク構成
高可用性ラズパイKubernetesクラスターの構築例を示すにあたり、下記に示すネットワーク構成を想定しました。  
クラスターの操作に関するAPIリクエストを受け取るアクティブ/スタンバイ冗長化構成ロードバランサのIPアドレスとして、仮想IPアドレスである **192.168.3.240/24** を割り当てることにします。  
アクティブ/スタンバイ冗長化構成ロードバランサとして構築されている2台のロードバランサに対しては **192.168.3.241/24** と **192.168.3.242/24** を割り当てることにします。  
アクティブ/アクティブ冗長化構成Kubernetesマスターノード群として構築されているの3台のマスターノードに対してはそれぞれ **192.168.3.251/24** と **192.168.3.252/24** 、**192.168.3.253/24** を割り当てることにします。  

<img src="./images/network_architecture.png" width=100% alt="High Availability Kubernetes Cluster Network Architecture">





## 要求
- マスターノード用Raspberry Pi：3台以上 (奇数台にする必要有り)
- ワーカーノード用Raspberry Pi：1台以上
- ロードバランサ用Raspberry Pi：2台以上
- Raspberry Piがクラスターとして稼働するための最低限の周辺機器で構成されており、初期設定済みであること
- 全てのRaspberry Piに対して、SSHでpiユーザにログインできること
- マスターノードは他の全てのマスターノードに対し、公開鍵認証でSSH接続できること
- kubeadmとkubelet、kubectlがマスターノードとワーカーノードにインストールされており、Kubernetesクラスターを構築する環境が整っていること

_**\* Raspberry Pi上にKubernetesクラスターを構築する環境を整える場合、[ラズパイKubernetesシングルコントロールプレーンクラスターの構築](https://github.com/izewfktvy533zjmn/Build_RasPi_Kubernetes_Cluster)を参考にすることを推奨します。**_





## 環境
### マスターノード
- Raspberry Pi 4 Model B
  - Docker v19.03.8
  - Kubernetes v1.13.5



### ワーカーノード
- Raspberry Pi 3 Model B
  - Docker v18.06.3
  - Kubernetes v1.13.5


- Raspberry Pi 4 Model B
  - Docker v19.03.8
  - Kubernetes v1.13.5



### ロードバランサ
- Raspberry Pi 4 Model B
  - Keepalived v2.0.10
  - HAProxy v1.8.19





## アクティブ/スタンバイ冗長化構成ロードバランサの構築

<img src="./images/redundant_load_balancers_on_network.png" width=100% alt="Redundant Load Balancers on Network">





## アクティブ/アクティブ冗長化構成Kubernetesマスターノード群の構築

<img src="./images/redundant_kubernetes_master_nodes_on_network.png" width=100% alt="Redundant Kubernetes Master Nodes on Network">
