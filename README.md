# **Learn OS on Weekends**

## **メモリ管理**

### **動的メモリ割り当て**

メインメモリには１バイトごとに１つの物理アドレスが割り当てられる。またプロセッサがアクセスできる物理アドレスの範囲を物理アドレス空間と呼ぶ。コンピュータでは、必要に応じてメモリ領域を動的に確保したり解放したりすることが一般的に行われる。このような処理は動的メモリ割り当てと呼ばれる。

### **動的メモリ割り当ての実装**

物理メモリを動的に割り当てられるようにしたい。ただし、`malloc`関数のようにバイト単位で割り当てるのではなく、ページと呼ばれるまとまった単位で割り当てる。ページサイズは4KB（4096B）が一般的である（`os/common.h`に`PAGE_SIZE`が定義されている）。具体的には、ページ数を引数として渡されると、そのページ分の割り当て可能なメモリ領域の先頭アドレスを返す関数を実装すること。このとき、渡すメモリ領域はゼロクリアしてから返すことが一般的である。`memset`関数が用意されているので、それを使用する。動的に割り当て可能なメモリ領域はリンカスクリプトで定義されている`__free_ram`から`__free_ram_end`までの領域とする。最後に実際に動的にメモリを割り当て、その物理アドレスを表示し、正しく動作することを確かめること。

今回は使い終わったメモリページを解放する処理は実装しない。自作OSを長時間実行することはないと思うので、メモリリークが発生しても問題ないためである。今回実装する割り当てアルゴリズムのことをLinearアロケータ（Bumpアロケータ）と呼び、解放処理が必要ない場面で実際に使われている。

### **仮想メモリ**

仮想メモリは、プロセスが使用するアドレスと実際の物理アドレスを分離することで、メモリを仮想化する技術である。仮想記憶では、プロセスごとに専用の独立したアドレス空間を割り当てる。プロセスから見えるアドレスは物理アドレスではなく、仮想アドレスである。プロセスがメモリにアクセスするときは、仮想アドレスを物理アドレスに変換した上で物理メモリにアクセスする。

仮想メモリを用いることで、他のプロセスに割り当てられたメモリに影響されずに各プロセスに対して連続したアドレス空間を提供でき、また各プロセスのアドレス空間を分離することで互いに隔離し、システムの安定性を高めることができる。

### **ページング方式**

仮想アドレスと物理アドレスとの対応関係（マッピング）をバイト単位で指定できるようにすると、アドレス変換のための情報量が膨大になってしまうため、ある程度の大きさのメモリブロックを単位としてマッピングを行う。現在は固定長のメモリブロックを単位とするページング方式が主流である。

ページング方式では、仮想アドレス空間と物理アドレス空間を固定長（ページサイズ）のブロックに分割する。仮想アドレス空間のブロックを仮想ページ（ページ）、物理アドレス空間のブロックを物理ページ（ページフレーム）と呼ぶ。ページにはページ番号がついており、仮想アドレスや物理アドレスは先頭部分にそのアドレスが属するページ番号を持ち、ページ内オフセットがそれに続く構成になっている。

仮想アドレスから物理アドレスへの変換は、ページを物理ページにマッピングすることで行われる。プロセスに割り当てるメモリは物理アドレス空間上では連続している必要はないため、物理メモリの外部断片化が発生しない。

カーネルは空いている物理ページを連結リストやビットマップなどで管理する。カーネルは新しくプロセスを実行する場合などにプロセスに割り当てるページが必要になると、空いている物理ページを割り当てる。また、プロセスが終了した場合など、ページが不要になれば対応する物理ページを回収する。

### **ページテーブル**

仮想アドレスから物理アドレスへの変換にはページテーブルと呼ばれるデータ構造を使用する。ページテーブルは仮想ページ番号でインデックスする配列であり、物理メモリ上に配置される。ページテーブルの要素をページテーブルエントリと呼び、１つのページテーブルエントリが１つの仮想ページに対応し、マッピングされている物理ページの番号や各種のフラグを格納している。

仮想アドレスを物理アドレスに変換する作業は、MMU（memory management unit）と呼ばれるハードウェアがページテーブルを参照することで動的に行われる（動的アドレス変換）。現代の一般的なプロセッサはMMUを内蔵している。

MMUには、使用するページテーブルの物理アドレスを保持するレジスタ（ページテーブルベースレジスタ）があり、異なるページテーブルを指すように変更することで、プロセッサが使用する仮想アドレス空間を切り替えることができる。

OSはプロセスごとにページテーブルを用意し、コンテキストスイッチによって実行するプロセスを切り替えるとき、ページテーブルベースレジスタの値を、次に実行するプロセスのページテーブルの先頭アドレスに設定し直す。

具体的には下記の流れでアドレス変換が行われる。

1. 仮想アドレスから仮想ページ番号とページ内オフセットを求める
2. ページテーブルベースレジスタからページテーブルを参照し、仮想ページ番号をもとに、ページテーブルエントリを取得する
3. ページテーブルエントリから物理ページ番号を取得する
4. 物理ページ番号とページ内オフセット（１で取得したもの）を組み合わせて物理アドレスを求める（物理ページ番号 * ページサイズ + ページ内オフセット）

### **多段ページテーブル**

仮想アドレス空間が広くなると、その分、必要なページテーブルのサイズも大きくなる。例えば仮想アドレス空間が256TB、ページサイズが4KB、ページテーブルエントリが8Bの場合、ページテーブルとして、256 / 4 * 8 = 512GB必要になる。しかし、一般的にプロセスは仮想アドレス空間のごく一部しか使用しないため、ほとんどのページテーブルエントリは無駄となる。

そこで考え出されたのが、ページテーブルを階層化した多段ページテーブルである。多段ページテーブルでは、最下位のページテーブルは１段しかない場合のページテーブルと同様のページテーブルエントリを保持するが、最下位以外のページテーブルでは、各エントリは１つ下のページテーブルの物理アドレスを保持する。

多段ページテーブルでは、実際に使用する仮想アドレスの範囲に対応するページテーブルのみを用意すれば良いため、物理メモリを節約することができる。

### **TLB**

ページテーブルを用いたアドレス変換では、メモリアクセスのたびにページテーブルの段数分のメモリ参照が追加で必要となるため、単純な実装ではただでさえ遅いメモリへのアクセスが非常に遅くなってしまう。このため、MMUは仮想アドレスから物理アドレスへのアドレス変換を高速化するためのTLB（トランスレーションルックアサイドバッファ）と呼ばれるハードウェアを備えている。

TLBはページテーブルエントリ専用のキャッシュメモリであり、連想メモリ（CAM: content addressable memory）と呼ばれる、高速に任意のエントリを検索できる特殊なメモリで構成されている。MMUは、仮想アドレスに対応するページテーブルエントリがTLB内に存在（TLBヒット）すれば、TLB内のページテーブルエントリを使用する。このときはページテーブルを参照する必要がないため、アドレス変換を高速に実行できる。一方、存在しなければ（TLBミス）、ページテーブルを参照してページテーブルエントリを取得し、取得したエントリをTLBに格納する。

一般的なプロセッサではTLBのエントリ数は数千個程度であり、TLBは仮想アドレスをキーとしてキャッシュされたページテーブルエントリを検索する。プロセスが異なれば同じ仮想アドレスでもページテーブルエントリは異なるため、プロセス間でコンテキストスイッチする場合、OSはTLBをクリアしなければならない。これをTLBフラッシュと呼び、そのための機械語命令がある。

### **多段ページテーブルの実装**

ここではSv32方式のページテーブルを構築したい。Sv32とはRISC-Vのページング機構である。２段構造のページテーブルになっており、32ビットの仮想アドレスのうち、先頭の10ビットは１段目のページテーブルのインデックス、続く10ビットは２段目のインデックス、残りの12ビットはページ内オフセットを保持する。１段目のページテーブルエントリは２段目のページテーブルの物理アドレスを保持し、２段目のページテーブルエントリは物理ページの物理アドレスを保持する。

ページテーブルエントリには物理アドレスだけでなく、各種のフラグも保持する。RISC-Vのページテーブルエントリは32ビットで、上位22ビットが物理ページ番号、下位10ビットがフラグである。フラグは`V`（有効化フラグ）、`R`（読み込みフラグ）、`W`（書き込みフラグ）、`X`（実行可能フラグ）、`U`（ユーザーモードフラグ）がある。`V`が0の場合はそのページテーブルエントリはマッピングされておらず、無効である。`R`、`W`、`X`が0の場合はそれぞれ読み込み、書き込み、実行が禁止される。`U`が0の場合はユーザーモードの場合にアクセスが禁止される。これらのフラグは`os/kernel.h`に定義されている。

ページテーブルは下記の流れで構築される。こうすることで、仮想ページと物理ページのマッピングを行うことができる。

1. 仮想アドレスの最初の10ビット（`vpn1`）を１段目のページテーブル（`table1`）のインデックスとして使用する。続く10ビット（`vpn0`）を２段目のページテーブル（`table0`）のインデックスとして使用する。残りの12ビット（`offset`）をページ内オフセットとして使用する
2. 上位22ビットに`table0`の物理ページ番号、下位10ビットに有効化された`V`フラグを格納した値を`table1[vpn1]`に格納する
3. 上位22ビットにマップする物理ページの番号、下位10ビットに`V`フラグを含む各種フラグを格納した値を`table0[vpn0]`に格納する

`os/common.h`に`is_aligned`というアライメントされているかどうかを確かめるためのマクロが定義されているので必要に応じて使用すること。

下位12ビット（ページ内オフセット）が異なる仮想アドレス同士は同じページテーブルエントリを共有することになる。また、真ん中の10ビットが異なる仮想アドレス同士も同じ２段目のページテーブルを共有することになる。つまり、近い仮想アドレス同士は同じページテーブルエントリやページテーブルに存在することになる。プログラムには[参照局所性](https://ja.wikipedia.org/wiki/参照の局所性)という特性があり、ある時点で参照されたリソースは近い将来にも再び参照される可能性が高く（時間的局所性）、あるリソースが参照されたとき、その近くにあるリソースも参照される可能性が高い（空間的局所性）ため、このようなページング機構になっているおかげで、ページテーブルのサイズを小さく抑えられ、またTLBヒット率が高くなる。

### **仮想アドレス空間**

仮想アドレス空間の構成はOSによって異なるが、低位領域をユーザープロセス用とし、高位領域をカーネル用とするのが一般的である。前者をユーザー空間、後者をカーネル空間と呼ぶ。

## **プロセス管理**

### **プロセスとは？**

OSでは実行中のプログラムのことをプロセスと呼ぶ。OSはプロセス単位で保護、リソース割り当て、権限管理を行う。

ユーザーは信頼度の低いプログラムを動かす可能性があり、そのような場合でもシステムを安定動作させるために、OSはそれぞれのプロセスが他のプロセスやOSそのものに影響を与えないように隔離し、保護する。これにより、バグがあるプログラムを動かしてもたいていの場合はそのプロセスだけがクラッシュ（異常終了）するだけですむ。

また、プロセスが動作するには機械語を実行するためのプロセッサと、プログラムやデータを配置するためのメモリが必要であり、OSは各プロセスに対して適切にこれらのリソースを割り当てる。

また、OSにはさまざまなオブジェクト（ファイル、デバイス、ソケット、共有メモリなど）が存在するが、OSはそれぞれのオブジェクトに対して操作を行う権限をプロセスごとに管理し、プロセスが許可されていない操作を行おうとすると、OSはプロセスの実行を停止させたり、ユーザーに許可を求めたりする。

### **PCB**

OSはプロセスを管理するためにそれぞれのプロセスに対してカーネル内にPCB（プロセス制御ブロック）と呼ばれるデータ構造を割り当てる。プロセッサごとに現在実行中のプロセスのPCBへのポインタを持つ。PCBには一般的に下記の内容が含まれる。

- PID（プロセスID）
- プロセスを実行しているユーザに関する情報
- プロセスが使用しているメモリ領域に関する情報
- プロセスが使用しているリソース（オープンしているファイルなど）に関する情報
- プロセスが保持する権限に関する情報
- アカウンティング情報（使用したCPU時間やメモリ使用量、入出力の回数などに関する統計情報）

### **PCBの実装**

PID、プロセスの状態、コンテキストスイッチ時のスタックポインタ、ページテーブルのポインタ、カーネルスタックを持つPCBを定義する。プロセッサがカーネル内のコードを実行しているときは、ユーザープロセスのスタック（ユーザースタック）とは別のスタック（カーネルスタック）を使用する。カーネルスタックはカーネル領域に存在する。カーネルスタックにはコンテキストスイッチ時のCPUレジスタ、どこから呼ばれたのか（関数の戻り先）、各関数でのローカル変数などが入っており、カーネルスタックをプロセスごとに用意することで、別の実行コンテキストを持ち、コンテキストスイッチで状態の保存と復元ができるようになる。

ちなみにプロセスの状態は`os/kernel.h`で、`PROC_UNUSED`（未使用）と`PROC_RUNNABLE`（実行可能）が定義されている。

### **スレッド**

例えばC言語のプログラムはmain関数を起点として、ループを回ったり条件分岐したり関数を呼び出したりしながら動作する。このようなプロセス内のプログラムの実行の流れをスレッドと呼ぶ。スレッドはプロセッサが機械語を実行する流れであるハードウェアスレッドを指す場合と、ここで述べたようなプロセス内のプログラム実行の流れを指す場合がある。そのため後者はユーザースレッドまたはソフトウェアスレッドとも呼ばれる。

同一プロセス内のスレッド同士はアドレス空間を共有し、互いに協調して動作する。これらのスレッドの間には保護は存在しない。あるスレッドがメモリ中のどこかのアドレスに書き込むとその内容は即座に他のスレッドから見える。異なるプロセス間では前述したように仮想メモリ機構によって互いのメモリにアクセスできないように隔離されている。

スレッドは生成時に指定されたアドレス（一般的には関数の先頭アドレス）から実行を開始する。それぞれのスレッドは独立したスタックとプロセッサのレジスタセットを持つ（今回のOSではスレッドは使用しないので、プロセスがスタックを持つ）。スレッドのスタックはアドレスが互いに重ならないように割り当てられる。プロセスの生成にともなって作られる初期スレッドではプロセスのスタック領域を用いるが、他のスレッドではヒープ領域などから割り当てられることが多い。

### **コンテキストスイッチ**

スレッドは自由に生成できるが、コンピュータがある瞬間に実行できるスレッド数はプロセッサ数に制限される。そのためOSはプロセッサを時分割して用いる。つまりプロセッサがあるスレッドを短時間実行したら、次に別のスレッドに切り替えて短時間実行する、というように実行するスレッドを短時間で切り替える。これによってOSはプロセッサの数以上のスレッドを見かけ上、同時（並行）に実行する。

このようにプロセッサが実行するスレッドを切り替える処理をコンテキストスイッチと呼ぶ。コンテキストスイッチはスレッドからは透過であり、それぞれのスレッドからは自分専用のプロセッサがスレッドを実行しているように見える（ユーザースレッドはハードウェアスレッドを仮想化したものと考えることができる）。

### **コンテキストスイッチの実装**

今回のOSではスレッドは使用しないため、プロセスのコンテキストスイッチを実装する。プロセスを作成する関数を実装し、複数のプロセス間でコンテキストスイッチをできるようにしたい。プロセスの最大数は`os/kernel.h`で`PROCS_MAX`が定義されており、この数以上のプロセスは作成できないようにすること。実装したらプロセスを複数作成し、コンテキストスイッチを行い、正しく動作することを確認すること。

コンテキストスイッチの際に、グローバル変数、スレッドは特にプロセス内で使用しないため、`gp`と`tp`は保存する必要はない。またRISC-Vの呼び出し規約（calling convention）により、`t0`~`t6`と`a0`~`a7`は関数呼び出し間で値が保持されることが保証されないレジスタであるため、これらのレジスタも保存する必要はない。`s0`~`s11`は関数内で使用するレジスタであるが、関数呼び出し間で値が保持されることが保証されるため、これらのレジスタの値を保存する必要がある。また関数の実行を正しく再開するために、`ra`も保存する必要がある。

### **スケジューラ**

現代のOSは複数のプログラムを同時に実行できるマルチタスク（マルチプロセス）をサポートしている。スケジューラの役割は、システム全体の効率を最大化し、すべてのプログラムが適切に実行されるようにすることである。マルチスレッドをサポートしない古典的なOSではスケジューリングの対象はプロセスであったが、現代のOSではスケジューリングの対象はスレッドである。スケジューリングの動作には、協調的マルチタスクとプリエンティブマルチタスクの２種類がある。

協調的マルチタスクではそれぞれのプログラムは自発的に他のプログラムにプロセッサを譲る（これをyieldという）操作を行うまで実行を継続する。協調的マルチタスクは単純なので、プリエンプティブマルチタスクに比べて実装が容易であるが、各プログラムは長時間プロセッサを占有しないように定期的にyieldする必要がある。

一方、プリエンプティブマルチタスクでは、プログラムが明示的にyieldしなくても、実行するプログラムがタイムスライス（プリエンプションされずに連続して実行できる時間）を使い切るとカーネルが強制的に切り替える。この強制的な切り替えのことをプリエンプション（横取り）と呼ぶ。プリエンプションされたプログラムはいずれ中断した処理の続きを実行する機会が与えられる。

### **スケジューラの実装**

今回のOSではスレッドを使用しないため、プロセスをスケジューリングの対象とする。コンテキストスイッチの実装では直接コンテキストスイッチを行う関数を呼び出して次に実行するプロセスを指定していたが、この方法ではプロセスの数が増えると次にどのプロセスに切り替えるべきかを決めるのは大変である。そこで、スケジューラを導入し、スケジューラが次に実行するプロセスを決定するようにする。

今回は協調的マルチタスクを行うスケジューラを実装したい。実行可能なプロセスがない場合はidleプロセスを実行するようにすること。プロセスを切り替える際はページテーブルを切り替える必要がある。またプロセスごとに別のカーネルスタックを使うようになったため、トラップハンドラの実装もそれに合わせて変更すること。このトラップハンドラはユーザーモードでトラップが発生した場合にも使用される予定なので、そのことを考慮し、脆弱性に気をつけて実装すること。最後に正しく動作し、ページが正しくマップされていることを確認すること。

`satp`レジスタは前述したページテーブルベースレジスタに相当する。`satp`レジスタにセットする値は、MSB（最上位ビット）がSv32方式を有効にするためのビットであり、下位ビットにはページテーブルの物理ページ番号が格納される。ちなみにブート時には`satp`レジスタは0に初期化されているため、ページングは無効になっており、物理アドレスがそのまま仮想アドレスとして扱われる。

## **ユーザープログラム**

### **ユーザープログラムの実装**

`os/shell.c`ですでに定義されているユーザープログラムを実行できるようにしたい。

`os/run.sh`を実行すると`shell.bin.o`が生成されてるのがわかる。`shell.bin.o`は生バイナリ形式の実行イメージであり、`llvm-nm`コマンドで中身を見てみると、`_binary_shell_bin_start`、`_binary_shell_bin_end`、`_binary_shell_bin_size`というシンボルが定義されていることがわかる。これらのシンボルはそれぞれ、実行イメージの先頭アドレス、終端アドレス、サイズを表している。

```bash
$ llvm-nm shell.bin.o
00010260 D _binary_shell_bin_end
00010260 A _binary_shell_bin_size
00000000 D _binary_shell_bin_start
```

`os/run.sh`ではカーネルのビルド時にClangに`shell.bin.o`を渡しているため、カーネル内でこれらのシンボルは例えば下記のように使用することができる。つまり、`_binary_shell_bin_start`は実行イメージのポインタとして、`_binary_shell_bin_size`は実行イメージのサイズとして使用できるのである。

```c
extern char _binary_shell_bin_start[];
extern char _binary_shell_bin_size[];

void main(void) {
  uint8_t *shell_bin = (uint8_t *) _binary_shell_bin_start;
  printf("shell_bin size = %d\n", (int) _binary_shell_bin_size);
  printf("shell_bin[0] = %x (%d bytes)\n", shell_bin[0]);
}
```

### **システムコール**

ユーザープログラムから自由にプロセッサの設定を変更したり、ハードウェアを直接制御したりできてしまうと、システムの安定性やセキュリティ上重大な問題となるため、カーネルは実行ファイルからロードした機械語プログラムを実行している間、プロセッサをユーザーモードに設定して、ユーザープログラムが問題がある動作を行おうとすると特権違反例外が発生し、カーネルはそのプロセスを強制終了するなどの処置をとる。

しかし、ユーザープログラムもファイルの読み書きやメモリの割り当てなど、リソースを操作したい場面が存在する。そのため、それらの操作をC言語などから呼び出し可能なAPIとして提供する。このAPIをシステムコールと呼ぶ。

ユーザープログラムからシステムコールを呼び出すと、プロセッサのモードをユーザーモードからカーネルモードに切り替え、システムコールの処理を実行した後、カーネルモードからユーザーモードに切り替えてからユーザープログラムに戻る。

システムコールの実行は下記のように行われる。

1. アプリケーションプログラムがシステムコールのラッパー関数を呼び出す
2. ラッパー関数はシステムコール番号とシステムコールへの引数をカーネルに渡すために、レジスタやスタックなどに書き込む
3. ラッパー関数はシステムコールを実行するための特別な機械語命令を実行する。この命令はソフトウェア割り込みを引き起こす
   1. プロセッサをカーネルモードに遷移させる
   2. スタックをカーネルスタックに切り替える
   3. ユーザープロセスの現在のコンテキストをレジスタやカーネルスタックなどに退避させる
   4. カーネル内にあるシステムコールを処理するためのルーチンに制御を移す
4. ２で渡されたシステムコール番号を読み取り、そのシステムコールを処理する関数を呼び出す
5. 処理が終了したらシステムコールの返り値をレジスタなどに書き込み、ユーザーモードに戻るための機械語命令を実行する
6. プロセッサはユーザーモードに遷移し、ユーザープロセスのコンテキストを復元する
7. ラッパー関数はシステムコールの返り値を返す
