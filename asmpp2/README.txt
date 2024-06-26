-----------------------------------------------------
	これは SC/MP-II CPU 用のC風プリプロセッサです
-----------------------------------------------------

概要
  C風の記法で書かれたソースを入力して 
  SC/MP-II CPU 用の asm ソースを出力します。

-----------------------------------------------------
アセンブル方法:
    http://john.ccac.rwth-aachen.de:8000/as/

上記サイトにある、大抵の8bit CPUをサポートしているマクロアセンブラ
を導入し、このMakefileに書かれている通りに asl コマンドでアセンブルしてください。
アセンブル出力は *.p というファイルに出力されますので、それを
p2bin もしくは p2hex という変換ツールによってROMバイナリーかHEXに変換してください。

-----------------------------------------------------
文法
  C言語にちょっとだけ似ています。

・コメントの書き方や、関数の書き方は C言語風です。

・関数引数や戻り値は、直接記述できませんので、レジスタ
  渡しや、ワークエリアでの値伝播で行います。

・使用できるプリプロセッサ構文

  コメントアウト
#if 0
...
#endif

  アセンブラソース一括挿入
#asm
...
#endasm

  ソースファイルインクルード
#include "file.src"

  アセンブラソースへパススルー
#<アセンブラで記述する１行そのまま>

・使用できるレジスタ名
  A  アキュムレータ(8bit)
  E  拡張レジスタ(8bit)
  PC プログラムカウンタ(16bit)
  P1 インデックスレジスタ(16bit)
  P2 インデックスレジスタ(16bit)
  P3 インデックスレジスタ(16bit)

・制御構文
  if () {  }
  while() {  }
  do {  } while();

・注意
  条件比較で A レジスタの値に対する比較
　if ( a == p2[0] ) などすべてにおいて、 比較命令がないので
  減算命令で代用しています。

　なので、比較後、常に A reg が破壊されます。(減算結果が入ります)

  ===> 次善の策として、  条件比較でEregを左辺に書くと、LD A,E 命令を先に挿入
       するようにしました。

  すなわち、case文のようなifを書く場合、常にAregは壊れますが、
'''
    e=a;
    if(e=='A') { ... } 
    if(e=='E') { ... } 
    if(e=='Z') { ... } 
'''
  のように書けます。

  EAレジスタとの比較は、まだ未実装ですが、もちろん比較後、EAレジスタともに壊れますし
　減算結果も破壊される予定です。

・バグ等
  算術演算、論理演算のオペランドが数値定数などの場合の、即値化が正しく動いていないので
　今のところ、即値の先頭に # を書く必要があるかもしれません。(一部修正済)


-----------------------------------------------------
* SC/MP-II コーディング上のルールについて
-----------------------------------------------------

* P1 = グローバル（ダイレクトアドレス）ポインタ

・SC/MP-II CPUは 8bit CPUなのに16bitポインタを3本（正確にはP0=PCとして4本）
  持つユニークなCPUです。

・が、ポインタ相対アドレッシング以外にメモリーにアクセスする手段がないのと
  一般的なCPUには必ず存在する CALL / RET 命令や、スタックポインタがありません。

・なので、やや特別なコーディング上のルールを適用します。

  --- 以下に説明

・ポインタ P1 は 0xff80 に固定します。
  ff80 でなくても、RAMを256byte以上持つシステムであれば、そこのRAMの中心 'xx80' 番地
  を指すことで、同様の効果となります。
　説明の都合上、 0xff80 に固定して説明します。

・ワークエリアは、128byte確保します。
  0xff80 ～ 0xffff の 128byte です。
  また、0xff80 番地は 8bit SP(スタックポインタ) として使用します。

  0xff00 ～ 0xff7f の 128byte は、スタック領域です。
　スタックレベルはせいぜい16レベルあれば間に合いますので、0xff00 ～ 0xff5f まで
  をその他ワークエリアに使用することができます。

・いずれにしても、これらのワークエリア、スタックエリアの参照は P1 ポインタが使用されます。
　（P1 の値は固定です）
  P1 を一時的に他の用途に使用したあとは、再度、固定番地をロードしてください。



* P3 = syscall サブルーチン

・一般的なCPUには必ず存在する CALL / RET 命令や、スタックポインタがありません。
  これを補完するために、P3 ポインタにも固定番地を入れて、CALL / RET 命令を作成します。

やりかたは、moni2.m にもあります。
--
 CALL / RET の実装

 P3=SYSCALL

   XPPC P3
   DB HIGH,LOW


SYSCALL-1:
   XPPC P3
SYSCALL:
   P3をPUSH
   HIGH,LOWをP3に取得
   JMP SYSCALL-1 ==> XPPC P3により HIGH,LOWに分岐

   HIGH=0のときは
　 P3をPOP
   JMP SYSCALL-1 ==> XPPC P3により 呼び出し元に戻る

   AとEが壊れるので、保存、復帰する.
--

* asmpp固有の表記方法について

・メモリーアクセスは、P1 ポインタのオフセット(+-128)を equ 定義したシンボル名
  を 0xff80～0xffff のワークエリアを指すダイレクトアドレスとして使用します。

例）
#asm
// EQU定義.
r1 = 0x30;  // ff80+30=ffb0番地になる。
r2 = 0x31;  // ff80+31=ffb1番地になる。
#endasm
   
// ソース、ディスティネーションのどちらかが Areg だった場合
   a = r1;  //   LD r1(P1)
   r1 = a;  //   ST r1(P1)

// ソース、ディスティネーションの両方がメモリーだった場合
   r1 = r2; //   LD r2(P1) , ST r1(P1)
   注意点として、 Areg は破壊されます。


* 関数定義、関数呼び出し

・asmpp には、型というものが存在しません。（あえていうなら 8bit Areg型のみ）

・なので、関数の戻り値や引数の定義はありません。
　勝手にレジスタやワークを関数の戻り値や引数に使用します。

// 関数の例
subroutine1()
{
   ・・・
// 最後に RET 命令マクロ (XPPC3 , DB(0)) が挿入されます。
}

・関数呼び出しは、関数名() で行います。 ()の中には何も書けません。

・()の中に何かを書いた場合は、それはアセンブラのマクロ＋引数、
  もしくはアセンブラの通常の命令語＋オペランドになります。

//例
    nop;
    ldi(0x20);
    xppc(p3);
    call(func1);
//オペランドや引数が無い場合は、()を書かないことでそのままasmに渡されます


