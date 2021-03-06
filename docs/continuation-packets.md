# 継続パケット拡張

## 概要

継続パケット拡張は PCP プロトコルの拡張で、チャンネルパケットに継続/非
継続の追加情報を付与して、ヘッドパケットの次にそのパケットからストリー
ムを開始していいかどうかの判断ができるようにするものです。

この拡張は、2 つの問題を解決しようとしています。1 つは、現状ではダイレ
クト接続で、FLV などのフォーマットでは不正なストリームが復元されてしま
う可能性があるということです。

PeerCast プロトコルではチャンネルパケットの大きさには 16 KB の制限があ
るのですが、旧来からの WMV ではエンコーダーがパケットが 8KB を超えない
ようにエンコードするので、WMV のパケットを PCP のパケットにそのまま格
納できたのでした。

いっぽう、FLV ではパケット(FLVタグ)のサイズは何 MB でもよいようになっ
ており、キーフレームをエンコードしたタグは特に大きくなるので、分割して
複数の PCP パケットに格納しています。

ダイレクト接続では、ヘッドパケットを送った後はどのデータパケットを送っ
てもよいと仮定しているので、例えばヘッドパケットの後に分割されたパケッ
トの 2 つ目が続くとストリームが正しい構造を持たなくなります。

プレーヤーはヘッダーさえ読み込めればストリームのデータに少しエラーがあっ
ても、次のパケットの先頭を探してくれるので、実際には大きな問題にはなら
ないのですが、フォーマットやプレーヤーによってはしばしばエラー復帰がう
まく働かずに再生に失敗することがあります。

元のパケットを分割して作った PCP パケットの先頭以外を「継続パケット」
と印を付けて、ダイレクト接続の再生時にこれらのパケットをヘッドパケット
の次に続けないようにすれば、常に正しいストリームが復元できると期待でき
ます。

2つ目の問題は、ストリームがキーフレームから始まるとは限らないというこ
とです。プレーヤーは動画ストリームがキーフレームで始まることを期待して
います。

これもキーフレームから始まらないからと言って、再生できないわけではない
のですが、プレーヤーによっては灰色の画面にもやもやと物の輪郭が表示され
るような始まり方をしたり、キーフレームまで画面の描画をしないプレーヤー
でも、音声が先に始まって画面は何秒も真っ暗のままといったことが起こりま
す。

「継続パケット」のフラグをキーフレームパケット以外のすべてに付ければ、
再生が必ずキーフレームから始まるようにできます。

## プロトコルの拡張

PCP はチャンネルパケットを `PCP_CHAN_PKT_DATA` アトム (アトム識別ID は
"data") に入れて転送します。`PCP_CHAN_PKT_DATA` は `PCP_CHAN_PKT(pkt)`
に入っており、後者はトップレベルの `PCP_CHAN(chan)` に入ります。以下は
実際の chan アトムの例です。

    chan {
        id bytes(53 11 A5 9E ED 27 B0 16 02 0E 70 4A 42 CE C5 31)
        pkt {
            type id4("data")
            pos  int(1080907)
            data bytes(...)
        }
    }

PeerCast YT では `PCP_CHAN_PKT_CONTINUATION(cont)` アトムを導入して、
継続パケットに印を付けるようにしました。アトムは 1 バイトのデータ部を
持っていて、これが 1 ならば cont が入っている pkt が継続パケットである
ことを示します。

    chan {
        id bytes(53 11 A5 9E ED 27 B0 16 02 0E 70 4A 42 CE C5 31)
        pkt {
            type id4("data")
            pos  int(1096267)
            cont char(1)
            data bytes(...)
        }
    }

0 で非継続パケット(その位置からのストリームデータ部分の開始が可能であ
る)を表わすこともできるのですが、後方互換性の関係上、cont アトムの無い
パケットは非継続として扱わなければならないので、実際にはデータ部 0 の
cont アトムを送るかわりに cont アトムを省略しています。

## 制限

PeerCast はもともとの仕様にないアトムは下流へ流さないようになっている
ので、継続情報はこの拡張に対応していないクライアントのノードを通過でき
ません。
