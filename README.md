### ステップバイステップ手順

このガイドでは、Ubuntu 22.04 LTS を使用し、Docker 上で IP アドレススプーフィングのシミュレーションを行う。手順の提示

1. **Linux ターミナルを開きます。**
2. **Docker Desktop を起動します。**

   Docker がインストールされていない場合は、先に Docker をインストールする

3. **Docker ネットワークと 3 つのコンテナを作成します。**

   これらのコマンドを使用してネットワークを作成し、各コンテナを設定します:

   ```bash
   docker network create --subnet=192.168.100.0/24 mynet

   docker run -itd --name container1 --network mynet --ip 192.168.100.10 ubuntu /bin/bash

   docker run -itd --name container2 --network mynet --ip 192.168.100.20 ubuntu /bin/bash

   docker run -itd --name container3 --network mynet --ip 192.168.100.30 --dns 8.8.8.8 --dns 8.8.4.4 ubuntu /bin/bash

   (if you want to confirm the dns settings on your circumstances, you should run this command; docker exec -it container3 cat /etc/resolv.conf)
   ```

4. **各コンテナで ping コマンドが使用可能にするために必要なパッケージをインストール**

   docker exec -it container1 /bin/bash -c "apt update && apt install -y iputils-ping"

   docker exec -it container2 /bin/bash -c "apt update && apt install -y iputils-ping"

   docker exec -it container3 /bin/bash -c "apt update && apt install -y iputils-ping"

5. **別のターミナルを開いてネットワークトラフィックをキャプチャ、他のステップはすべて最初のターミナルセッションで行う**

   sudo apt-get update

   sudo apt-get install tcpdump

   sudo tcpdump -i docker0 -nn -v icmp

**必要に応じて、ファイアウォールのルールやネットワークの設定を確認**

    (必要であれば)FW ルールの確認

    sudo iptables -L

    (必要であれば)ネットワークルールの確認

    docker network inspect mynet

7. **ネットワークの確認をするために各コンテナに arp ツールをインストール+設定内容御確認の出力**

   docker exec -it container1 /bin/bash -c "apt update && apt install net-tools && arp -n"

   docker exec -it container2 /bin/bash -c "apt update && apt install net-tools && arp -n"

   docker exec -it container3 /bin/bash -c "apt update && apt install net-tools && arp -n"

8. **container1 container2 間で ping 通信が可能であることを確認**
   docker exec -it container1 ping -c 4 192.168.100.20
9. **container3 で container1 に対して container2 として spoofing するための環境構築**

   docker exec -it container3 ifconfig eth0 down

   docker exec -it container3 macchanger -m 02:42:c0:a8:64:14 eth0

   docker exec -it container3 ifconfig eth0 up

10. **container3 でスプーフィングをするために必要なツールをインストール＋環境構築**
    [環境構築]

    docker exec -it container3 apt-get update

    docker exec -it container3 apt-get install macchanger

    [mac アドレスの変更]

    docker exec -it container3 ifconfig eth0 down

    docker exec -it container3 macchanger -m 02:42:c0:a8:64:14 eth0 // -m のあとのアドレスは container2 のものと同一、
    ネットワーク内を監視するツールからは情報が筒抜けだが、container1 からは判別不可

    docker exec -it container3 ifconfig eth0 up

11. **偽装した mac アドレスで ip アドレススプーフィングを実施**
    docker exec -it container3 hping3 -c 4 -1 -a 192.168.100.20 192.168.100.10

```
【結果】
・11 のコマンドを実行したターミナルで、
$ docker exec -it container3 hping3 -c 4 -1 -a 192.168.100.20 192.168.100.10
HPING 192.168.100.10 (eth0 192.168.100.10): icmp mode set, 28 headers + 0 data bytes
len=28 ip=192.168.100.10 ttl=64 id=36597 icmp_seq=0 rtt=9.4 ms
len=28 ip=192.168.100.10 ttl=64 id=36697 icmp_seq=1 rtt=4.2 ms
len=28 ip=192.168.100.10 ttl=64 id=36701 icmp_seq=2 rtt=13.8 ms
len=28 ip=192.168.100.10 ttl=64 id=36730 icmp_seq=3 rtt=13.4 ms

--- 192.168.100.10 hping statistic ---
4 packets transmitted, 4 packets received, 0% packet loss
と出力されれば成功

・もう一方の監視ターミナルにおいては以下の通り
$ docker exec -it container1 tcpdump -i eth0 icmp
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
05:19:00.422650 IP container2.mynet > e5f80ef589c0: ICMP echo request, id 61187, seq 0, length 8
05:19:00.422881 IP e5f80ef589c0 > container2.mynet: ICMP echo reply, id 61187, seq 0, length 8
05:19:00.426533 IP container3.mynet > e5f80ef589c0: ICMP redirect container2.mynet to host container2.mynet, length 36
05:19:01.427148 IP container2.mynet > e5f80ef589c0: ICMP echo request, id 61187, seq 256, length 8
05:19:01.427167 IP e5f80ef589c0 > container2.mynet: ICMP echo reply, id 61187, seq 256, length 8
05:19:01.427190 IP container3.mynet > e5f80ef589c0: ICMP redirect container2.mynet to host container2.mynet, length 36
05:19:02.427793 IP container2.mynet > e5f80ef589c0: ICMP echo request, id 61187, seq 512, length 8
05:19:02.427813 IP e5f80ef589c0 > container2.mynet: ICMP echo reply, id 61187, seq 512, length 8
05:19:02.427970 IP container3.mynet > e5f80ef589c0: ICMP redirect container2.mynet to host container2.mynet, length 36
05:19:03.428350 IP container2.mynet > e5f80ef589c0: ICMP echo request, id 61187, seq 768, length 8
05:19:03.428370 IP e5f80ef589c0 > container2.mynet: ICMP echo reply, id 61187, seq 768, length 8
05:19:03.428394 IP container3.mynet > e5f80ef589c0: ICMP redirect container2.mynet to host container2.mynet, length 36
^C
12 packets captured
12 packets received by filter
0 packets dropped by kernel
```

**補足**

1. 基本的に内部ネットワークに干渉する処理において、その権限を内部ネットワーク内の各デバイスに与えず(権限をさいしょうげんにする原則)、また、外部からの通信の場合はそれに伴う処理の権限などにおいて必要最小限以上のものを付与しないことで基本的には防げる、つまり、システム上重要な処理は外部から行うべきでなく、仮に行う場合は、IP アドレスや MAC アドレスのみで認証をしないことで偽装を防ぐことができる

**内部ネットワークに侵入されたり攻撃を受けたときの対処法**

1. ネットワークトラフィックの解析:
   MAC アドレス: ネットワーク上のデバイスが MAC アドレスを調査し、ARP キャッシュを確認すると、実際の物理的な送信元が明らかになる可能性があります。もし container1 が ARP キャッシュに container2 の IP アドレスと関連付けられた異なる MAC アドレスを持っている場合、スプーフィングが疑われる可能性があります。

ネットワークのレイテンシとパフォーマンス: 通常、同じネットワーク内の異なるデバイス間での通信レイテンシは一貫しています。異常な遅延やパフォーマンスの変動が観察されると、通信の起源を疑う理由になり得ます。

2. ICMP リダイレクトメッセージ:
   ICMP redirect メッセージは、ルーターやゲートウェイが提案するルート変更を示しますが、通常は同一のネットワーク内のデバイス間での通信には使用されません。この種のメッセージが頻繁に発生すると、何らかの異常があると見なされる可能性があります。
3. セキュリティツールと監視:
   侵入検知システム (IDS) と侵入防止システム (IPS): ネットワーク上で IDS/IPS ソリューションが稼働している場合、異常なトラフィックパターンや既知の攻撃シグネチャに基づいてスプーフィングの試みを検出できる可能性があります。
