# Pirate Radio 起動スクリプトの修正(Pimoroni pHAT BEAT)

Raspberry Pi Zero WH と pHAT BEAT で作る Pimoroni の「Pirate Radio」(インターネットラジオ)が、突然鳴らなくなった問題の復旧と、その後に行った起動スクリプトの修正を記録したものです。

- 復旧編(原因の解析と手順): https://note.com/todopapa/n/n896d5e9e6ac6
- OLED増設編: https://note.com/todopapa/n/nda2a3ca7b67c

## 確認した環境

- Raspberry Pi Zero WH + pHAT BEAT
- Raspbian GNU/Linux 11 (bullseye)

他の環境での動作は確認していません。

## このリポジトリの内容

ラズパイ上の配置と同じ構成で、修正後のファイルを入れています。

```
usr/bin/vlcd              VLC本体の起動スクリプト(ネットワーク待ちを追加)
etc/init.d/vlcd           VLCの起動ラッパ(runuser を使用)
etc/init.d/phatbeatd      ボタン制御デーモンの起動ラッパ(runuser を使用)
```

## 修正の内容

### 1. `su -c` を `runuser` に置き換える

`/etc/init.d/phatbeatd` の起動処理(修正後)

```sh
runuser -u root -- "$DAEMON"
```

`/etc/init.d/vlcd` の起動処理(修正後)

```sh
runuser -u pi -- "$DAEMON"
```

### 2. Wi-Fiが使えるまで待つ

起動直後にWi-Fiがまだつながっていないことがあり、固定の待ち時間では不安定でした。`/usr/bin/vlcd` の1行目(`#!/usr/bin/env bash`)の直後に、DNSが引けるまで(最大60秒)待つ処理を追加しています。

```bash
# Wi-Fi・DHCP・DNSが使えるまで待つ(最大60秒)
for i in $(seq 1 60); do
    if getent hosts www.google.com >/dev/null 2>&1; then
        echo "network ready after ${SECONDS}s"
        break
    fi
    sleep 1
done
```

接続が早いときはすぐ次へ進みます。60秒たっても接続できないときは、そのままVLCを起動します。

## 適用手順

編集する前に、バックアップを取ります。

```
sudo cp /usr/bin/vlcd /usr/bin/vlcd.bak
sudo cp /etc/init.d/vlcd /etc/init.d/vlcd.bak
sudo cp /etc/init.d/phatbeatd /etc/init.d/phatbeatd.bak
```

上の修正を手で入れるか、このリポジトリのファイルをコピーします。**再生リストや音量の設定を自分で変えている場合は、全文をコピーせず、修正部分だけを入れてください。**

```
sudo cp usr/bin/vlcd /usr/bin/vlcd
sudo cp etc/init.d/vlcd /etc/init.d/vlcd
sudo cp etc/init.d/phatbeatd /etc/init.d/phatbeatd
sudo chmod +x /usr/bin/vlcd /etc/init.d/vlcd /etc/init.d/phatbeatd
```

構文を確認して、再起動します。

```
bash -n /usr/bin/vlcd && echo OK
sudo systemctl daemon-reload
sudo reboot
```

## 動作確認

```
journalctl -b | grep -i "network ready"
systemctl status vlcd --no-pager
```

- `network ready after 〇s` が出ていれば、待ちの処理が動いています。
- `vlcd` が `active (running)` で、音が鳴り、ボタンが効けば完了です。

確認できたこと: ネットワークがすぐ使える起動、接続まで待ちが発生した起動、どちらでも音が鳴りました。

## 注意

- VLCのリモート操作(ポート9294はパスワードなし、ポート8080はパスワード `raspberry`)が、すべてのネットワークに向けて開いています。家庭内LANでの利用を前提としてください。
- これらのスクリプトは、Pimoroni の Pirate Radio / phat-beat のものを改変したものです。出典は https://github.com/pimoroni/phat-beat です。元のライセンスに従ってください。
- 自己責任でお使いください。
