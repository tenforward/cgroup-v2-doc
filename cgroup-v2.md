
# Control Group v2

October, 2015		Tejun Heo <tj@kernel.org>

<!--
This is the authoritative documentation on the design, interface and
conventions of cgroup v2.  It describes all userland-visible aspects
of cgroup including core and specific controller behaviors.  All
future changes must be reflected in this document.  Documentation for
v1 is available under Documentation/cgroup-legacy/.
-->
これは cgroup v2 のデザイン、インターフェース、きまりごとの正式な文書です。cgroup コアと特定のコントローラの動きを含む、cgroup のユーザランドから見えるものを説明します。将来的な変更はすべてこの文書に反映させなければなりません。v1 の文書は Documentation/cgroup-legacy/ 以下にあります。

# CONTENTS

```
1. はじめに
  1-1. 用語
  1-2. cgroup とは何か?
2. 基本的な操作
  2-1. マウント
  2-2. プロセスとスレッドの体系化
    2-2-1. プロセス
    2-2-2. スレッド
  2-3. [un]populated 通知
  2-4. コントローラの制御
    2-4-1. 有効化と無効化
    2-4-2. トップダウン制約
    2-4-3. 内部にプロセスを持たない制約
  2-5. 権限委譲
    2-5-1. 権限委譲モデル
    2-5-2. 権限委譲の制約
  2-6. ガイドライン
    2-6-1. 一度だけ組織化してコントロールする
    2-6-2. 名前の衝突を避ける
3. リソース分配のモデル
  3-1. ウェイト
  3-2. 制限
  3-3. 保護
  3-4. 割り当て
4. インターフェースファイル
  4-1. フォーマット
  4-2. 規約
  4-3. コアインターフェースファイル
5. コントローラ
  5-1. CPU
    5-1-1. CPU インターフェースファイル
  5-2. Memory
    5-2-1. メモリインターフェースファイル
    5-2-2. 一般的な使い方
    5-2-3. メモリの所有権
  5-3. IO
    5-3-1. IO インターフェースファイル
    5-3-2. Writeback
P. Information on Kernel Programming
  P-1. Filesystem Support for Writeback
D. Deprecated v1 Core Features
R. Issues with v1 and Rationales for v2
  R-1. Multiple Hierarchies
  R-2. Thread Granularity
  R-3. Competition Between Inner Nodes and Threads
  R-4. Other Interface Issues
  R-5. Controller Issues and Remedies
    R-5-1. Memory
```

<!--
Introduction
-->
はじめに
======

<!--
Terminology
-->
用語
----

<!--
"cgroup" stands for "control group" and is never capitalized.  The
singular form is used to designate the whole feature and also as a
qualifier as in "cgroup controllers".  When explicitly referring to
multiple individual control groups, the plural form "cgroups" is used.
-->
"cgroup" は "control group" を表します。大文字では表記することはありません。単数形は全機能を表したり、"cgroup controllers" のように修飾子として全機能を表すために使います。明確に複数の個別の control groups を示すときに、複数形の "cgroups" を使います。

<!--
What is cgroup?
-->
cgroup とは何か?
---------------

<!--
cgroup is a mechanism to organize processes hierarchically and
distribute system resources along the hierarchy in a controlled and
configurable manner.
-->
cgroup は、階層的にプロセスをまとめるためのメカニズムです。また、管理されており設定可能な方法で、階層にしたがってシステムリソースを分配するためのメカニズムでもあります。

<!--
cgroup is largely composed of two parts - the core and controllers.
cgroup core is primarily responsible for hierarchically organizing
processes.  A cgroup controller is usually responsible for
distributing a specific type of system resource along the hierarchy
although there are utility controllers which serve purposes other than
resource distribution.
-->
cgroup は大きくふたつの部分から成っています。それはコアとコントローラです。cgroup コアは主に階層構造でプロセスをまとめる働きをします。
cgroup コントローラは通常、階層構造で特定のタイプのシステムリソースを分配する働きをします。しかし、リソース分配以外の目的を提供するユーティリティ的なコントローラもあります。

<!--
cgroups form a tree structure and every process in the system belongs
to one and only one cgroup.  All threads of a process belong to the
same cgroup.  On creation, all processes are put in the cgroup that
the parent process belongs to at the time.  A process can be migrated
to another cgroup.  Migration of a process doesn't affect already
existing descendant processes.
-->
cgroups はツリー構造を形成し、システム上のそれぞれのプロセスは、唯一のcgroup に属します。プロセスのすべてのスレッドは同じ cgroup に属します。
生成時、すべてのプロセスはその時点で親プロセスが属している cgroup に置かれます。プロセスを他の cgroup に移動できます。プロセスの移動は、既存の派生プロセスには影響を与えません。

<!--
Following certain structural constraints, controllers may be enabled or
disabled selectively on a cgroup.  All controller behaviors are
hierarchical - if a controller is enabled on a cgroup, it affects all
processes which belong to the cgroups consisting the inclusive
sub-hierarchy of the cgroup.  When a controller is enabled on a nested
cgroup, it always restricts the resource distribution further.  The
restrictions set closer to the root in the hierarchy can not be
overridden from further away.
-->
後述の構造的な制約に従い、コントローラは cgroup 上で有効にするか無効にするかを選択できます。コントローラはすべて、階層的に動作します。あるコントローラが cgroup で有効になった場合、cgroup のサブ階層すべてを含むものから成る cgroups に属するすべてのプロセスに影響します。ネストした cgroup 内でコントローラが有効な場合、さらに常にリソース分配が制限されます。階層のルートに近いところで設定される制限は、より階層が深いところから上書きできません。


基本的な操作
==========

マウント
------

<!--
Unlike v1, cgroup v2 has only single hierarchy.  The cgroup v2
hierarchy can be mounted with the following mount command.
-->
v1 と違い、cgroup v2 は単一の階層構造のみを持ちます。cgroup v2 階層は以下のように mount コマンドでマウントできます。

```
  # mount -t cgroup2 none $MOUNT_POINT
```

<!--
cgroup2 filesystem has the magic number 0x63677270 ("cgrp").  All
controllers which support v2 and are not bound to a v1 hierarchy are
automatically bound to the v2 hierarchy and show up at the root.
Controllers which are not in active use in the v2 hierarchy can be
bound to other hierarchies.  This allows mixing v2 hierarchy with the
legacy v1 multiple hierarchies in a fully backward compatible way.
-->
cgroup2 ファイルシステムのマジックナンバーは 0x63677270 ("cgrp") です。
v2 でサポートされていて、v1 の階層にバインドされていないコントローラがすべて root に出現します。v2 階層内でアクティブに使用していないコントローラは、他の階層にバインドできます。これにより、完全に下位互換性を保ちながら、レガシーな v1 の複数階層構造と同時に v2 階層をミックスで使用できます。

<!--
A controller can be moved across hierarchies only after the controller
is no longer referenced in its current hierarchy.  Because per-cgroup
controller states are destroyed asynchronously and controllers may
have lingering references, a controller may not show up immediately on
the v2 hierarchy after the final umount of the previous hierarchy.
Similarly, a controller should be fully disabled to be moved out of
the unified hierarchy and it may take some time for the disabled
controller to become available for other hierarchies; furthermore, due
to inter-controller dependencies, other controllers may need to be
disabled too.
-->
コントローラは、属する階層構造内での参照がされなくなったあとにだけ、階層をまたがって移動できます。なぜなら、cgroup ごとのコントローラの状態は非同期に削除されるので、コントローラには残っている参照があるかもしれません。そういうわけで、コントローラは、前の階層から最終的に umount されたあとすぐには、v2 の階層には現れないかもしれません。同様にコントローラは、単一階層構造から引っ越すために完全に無効化しなくてはなりません。
そして、無効化されたコントローラが、他の階層で有効になるには少し時間がかかるかもしれません。さらに、コントローラ間の依存関係により、他のコントローラも無効化する必要があるかもしれません。

<!--
While useful for development and manual configurations, moving
controllers dynamically between the v2 and other hierarchies is
strongly discouraged for production use.  It is recommended to decide
the hierarchies and controller associations before starting using the
controllers after system boot.
-->
開発とマニュアルでの設定には便利ですが、コントローラを v2 と他の階層間でダイナミックに移動させることは、プロダクション利用では強く推奨されません。システムブート後にコントローラを使いはじめる前に、階層とコントローラの関連付けを決定しておくことおすすめします。

<!--
During transition to v2, system management software might still
automount the v1 cgroup filesystem and so hijack all controllers
during boot, before manual intervention is possible. To make testing
and experimenting easier, the kernel parameter cgroup_no_v1= allows
disabling controllers in v1 and make them always available in v2.
-->
v2 への移行の間、システム管理ソフトウェアは v1 cgroup ファイルシステムを自動的にマウントし続け、手動で操作ができるようになる前に、ブート時にすべてのコントローラを乗っ取ってしまうかもしれません。簡単にテストや実験を行うには、カーネルパラメータ cgroup_no_v1= を使って、v1 のコントローラを無効化し、それらを v2 で利用できるようにできます。

<!--
cgroup v2 currently supports the following mount options.
-->
cgroup v2 は現時点で次のオプションをサポートします。

<dl>
<dt>nsdelegate</dt>

<!--
        Consider cgroup namespaces as delegation boundaries.  This
        option is system wide and can only be set on mount or modified
        through remount from the init namespace.  The mount option is
        ignored on non-init namespace mounts.  Please refer to the
        Delegation section for details.
-->
<dd>cgroup 名前空間を権限委譲の境界とみなします。このオプションはシステムワイドで、マウント時のみ設定でき、初期の名前空間から再マウントすることで変更できます。このオプションは、初期の名前空間以外からのマウントでは無視されます。詳しくは権限委譲のセクションを参照してください。</dd>
</dl>

<!--
Organizing Processes and Threads
-->
プロセスとスレッドの体系化
--------------------

<!--
Processes
~~~~~~~~~
-->
### プロセス

<!--
Initially, only the root cgroup exists to which all processes belong.
A child cgroup can be created by creating a sub-directory.
-->
最初は、すべてのプロセスが属する root cgroup だけが存在します。子cgroup はサブディレクトリを作ることにより作られます。

```
  # mkdir $CGROUP_NAME
```

<!--
A given cgroup may have multiple child cgroups forming a tree
structure.  Each cgroup has a read-writable interface file
"cgroup.procs".  When read, it lists the PIDs of all processes which
belong to the cgroup one-per-line.  The PIDs are not ordered and the
same PID may show up more than once if the process got moved to
another cgroup and then back or the PID got recycled while reading.
-->
指定した cgroup はツリー構造からなる複数の子 cgroup を持っているかもしれません。cgroup はそれぞれが読み書き可能なインターフェースファイル"cgroup.procs" を持っています。ファイルの読み込み時は、cgroup に属するプロセスの PID すべてを一行にひとつリストします。PID はソートはされていません。そして、プロセスが他の cgroup に移動した後に戻ってきた場合や、読み込んでいる間に PID が再利用された場合、同じ PID が一度以上現れるかもしれません。

<!--
A process can be migrated into a cgroup by writing its PID to the
target cgroup's "cgroup.procs" file.  Only one process can be migrated
on a single write(2) call.  If a process is composed of multiple
threads, writing the PID of any thread migrates all threads of the
process.
-->
プロセスの PID を、ターゲットとなる cgroup の "cgroup.procs" ファイルに書き込むことにより、プロセスを別の cgroup へ移動できます。ひとつのプロセスだけが、単一の write(2) のコールで移動できます。プロセスが複数のスレッドで構成される場合は、任意のスレッドの PID を書き込むことで、プロセスのすべてのスレッドが移動します。

<!--
When a process forks a child process, the new process is born into the
cgroup that the forking process belongs to at the time of the
operation.  After exit, a process stays associated with the cgroup
that it belonged to at the time of exit until it's reaped; however, a
zombie process does not appear in "cgroup.procs" and thus can't be
moved to another cgroup.
-->
プロセスが子プロセスを fork した場合、新しいプロセスは、操作時点でfork するプロセスが属する cgroup 内で生成されます。exit のあと、プロセスが刈り取られるまでは、プロセスは exit 時点に属していた cgroup に関連付けられ続けます。しかし、ゾンビプロセスは "cgroup.procs" 内には現れません。ですので、他の cgroup には移動できません。

<!--
A cgroup which doesn't have any children or live processes can be
destroyed by removing the directory.  Note that a cgroup which doesn't
have any children and is associated only with zombie processes is
considered empty and can be removed.
-->
子 cgroup も実行中のプロセスも持たない cgroup はディレクトリを削除すれば削除できます。子 cgroup を持っておらず、ゾンビプロセスのみが関連付けられている cgroup は空であるとみなせるので、削除できます。

```
  # rmdir $CGROUP_NAME
```

<!--
"/proc/$PID/cgroup" lists a process's cgroup membership.  If legacy
cgroup is in use in the system, this file may contain multiple lines,
one for each hierarchy.  The entry for cgroup v2 is always in the
format "0::$PATH".
-->
"/proc/$PID/cgroup" はプロセスがどの cgroup に属しているかをリストします。システム上で旧 cgroup が使われている場合、このファイルは複数の行が含まれるかもしれません。cgroup v2 のエントリは常に "0::$PATH" というフォーマットです。

```
  # cat /proc/842/cgroup
  ...
  0::/test-cgroup/test-cgroup-nested
```

<!--
If the process becomes a zombie and the cgroup it was associated with
is removed subsequently, " (deleted)" is appended to the path.
-->
プロセスがゾンビになった場合で、関連付けられていた cgroup がその後削除された場合は、" (deleted)" がパスに付け加えられます。

```
  # cat /proc/842/cgroup
  ...
  0::/test-cgroup/test-cgroup-nested (deleted)
```

<!--
Threads
-->
### スレッド

<!--
cgroup v2 supports thread granularity for a subset of controllers to
support use cases requiring hierarchical resource distribution across
the threads of a group of processes.  By default, all threads of a
process belong to the same cgroup, which also serves as the resource
domain to host resource consumptions which are not specific to a
process or thread.  The thread mode allows threads to be spread across
a subtree while still maintaining the common resource domain for them.
-->
cgroup v2 はコントローラのサブセットに対するスレッドの粒度がサポートさ
れています。これは、プロセスグループのスレッドでの階層的なリソース分配
が必要とされているようなユースケースをサポートするためです。デフォルト
では、プロセスのすべてのスレッドは同じcgroupに属します。プロセスもしく
はスレッド固有ではないホストのリソース消費に対するリソースドメインとし
ての役割も果たします。スレッドモードでは、スレッドが共通のリソースドメ
インを維持しながら、サブツリー全体にスレッドを分散できます。

<!--
Controllers which support thread mode are called threaded controllers.
The ones which don't are called domain controllers.
-->
スレッドをサポートするコントローラはスレッド化コントローラと呼ばれます。
そうでないものは、ドメインコントローラと呼ばれます。

<!--
Marking a cgroup threaded makes it join the resource domain of its
parent as a threaded cgroup.  The parent may be another threaded
cgroup whose resource domain is further up in the hierarchy.  The root
of a threaded subtree, that is, the nearest ancestor which is not
threaded, is called threaded domain or thread root interchangeably and
serves as the resource domain for the entire subtree.
-->
cgroup をスレッド化するとマークすると、スレッド cgroup として、その親
のリソースドメインに参加します。親は、リソースドメインが階層のさらに上
にある他のスレッド cgroup でも構いません。スレッド化されたサブツリーの
root、すなわち、スレッド化されていない直近の祖先が、スレッド化ドメイン
もしくはスレッド化 root と呼ばれ、サブツリー全体のリソースドメインとし
て機能します。

<!--
Inside a threaded subtree, threads of a process can be put in
different cgroups and are not subject to the no internal process
constraint - threaded controllers can be enabled on non-leaf cgroups
whether they have threads in them or not.
-->
スレッド化サブツリーの内部では、プロセスのスレッドは異なる cgroup に置
けて、内部プロセスの制約外となります。スレッド化コントローラは、スレッ
ドが存在するかどうかに関わらず、リーフ以外の cgroup で有効化できます。

<!--
As the threaded domain cgroup hosts all the domain resource
consumptions of the subtree, it is considered to have internal
resource consumptions whether there are processes in it or not and
can't have populated child cgroups which aren't threaded.  Because the
root cgroup is not subject to no internal process constraint, it can
serve both as a threaded domain and a parent to domain cgroups.
-->
スレッド化ドメイン cgroup は、サブツリーのドメインのリソース消費のすべ
てをホストしているので、プロセスが存在していてもしていなくても、内部リ
ソース消費を持つとみなされます。root cgroup は内部プロセスの制約があり
ませんので、スレッド化ドメインとドメイン cgroup の親の両方として機能し
ます。

<!--
The current operation mode or type of the cgroup is shown in the
"cgroup.type" file which indicates whether the cgroup is a normal
domain, a domain which is serving as the domain of a threaded subtree,
or a threaded cgroup.
-->
cgroup の現在の操作モードもしくはタイプは "cgroup.type" ファイル内で確
認できます。このファイルは、cgroup が通常のドメインなのか、スレッド化
サブツリーのドメインとして機能するドメインなのか、スレッド化 cgroup と
して機能するドメインなのかを示します。

<!--
On creation, a cgroup is always a domain cgroup and can be made
threaded by writing "threaded" to the "cgroup.type" file.  The
operation is single direction::
-->
作成時、cgroup は常にドメイン cgroup であり、"threaded" という文字列を
"cgroup.type" ファイルに書き込むことでスレッド化することができます。

```
  # echo threaded > cgroup.type
```

<!--
Once threaded, the cgroup can't be made a domain again.  To enable the
thread mode, the following conditions must be met.
-->
一度スレッド化されると、cgroup は再度ドメインにはなれません。スレッド
モードを有効にするには、以下の状態でなければなりません。

<!--
- As the cgroup will join the parent's resource domain.  The parent
  must either be a valid (threaded) domain or a threaded cgroup.
-->
- cgroup は親のリソースドメインにジョインするので、親は有効な（スレッ
  ド化）ドメインもしくはスレッド化 cgroup でなければなりません

<!--
- When the parent is an unthreaded domain, it must not have any domain
  controllers enabled or populated domain children.  The root is
  exempt from this requirement.
-->
- 親がスレッド化ドメインの場合、ドメインコントローラが有効になっている
  もしくは、ドメインの子が populated になっていてはいけません。root は
  この必要条件を満たす必要はありません。

<!--
Topology-wise, a cgroup can be in an invalid state.  Please consider
the following toplogy::
-->
トポロジーの点では、cgroup は無効な状態を取り得ます。以下のトポロジーを考えてみてください。

  A (threaded domain) - B (threaded) - C (domain, just created)

<!--
C is created as a domain but isn't connected to a parent which can
host child domains.  C can't be used until it is turned into a
threaded cgroup.  "cgroup.type" file will report "domain (invalid)" in
these cases.  Operations which fail due to invalid topology use
EOPNOTSUPP as the errno.
-->
C はドメインとして作られますが、子ドメインをホストできる親には接続されていません。C はスレッド化 cgroup に変更されるまでは使えません。この場合、"cgroup.type" ファイルは "domain (invalid)" と報告します。無効なトポロジーによる失敗操作は errno として EOPNOTSUPP を使います。

<!--
A domain cgroup is turned into a threaded domain when one of its child
cgroup becomes threaded or threaded controllers are enabled in the
"cgroup.subtree_control" file while there are processes in the cgroup.
A threaded domain reverts to a normal domain when the conditions
clear.
-->
ドメイン cgroup は、cgroup 内にプロセスがある間に、子 cgroup のひとつがスレッド化になるか、スレッド化コントローラが "cgroup.subtree_control" で有効になると、スレッド化ドメインに変化します。条件がクリアされると、スレッド化ドメインは通常のドメインに戻ります。

<!--
When read, "cgroup.threads" contains the list of the thread IDs of all
threads in the cgroup.  Except that the operations are per-thread
instead of per-process, "cgroup.threads" has the same format and
behaves the same way as "cgroup.procs".  While "cgroup.threads" can be
written to in any cgroup, as it can only move threads inside the same
threaded domain, its operations are confined inside each threaded
subtree.
-->
"cgroup.threads" には、その cgroup 内のすべてのスレッド ID のリストが含まれています。操作がプロセス単位でなくスレッド単位であることを除いて、"cgroup.threads" ファイルは "cgroup.procs" と同じふるまいをし、同じフォーマットを持ちます。任意の cgroup の "cgroup.threads" に書き込めますが、同じスレッドドメイン内でしかスレッドは移動できませんし、操作はスレッド化サブツリー内に限られます。

<!--
The threaded domain cgroup serves as the resource domain for the whole
subtree, and, while the threads can be scattered across the subtree,
all the processes are considered to be in the threaded domain cgroup.
"cgroup.procs" in a threaded domain cgroup contains the PIDs of all
processes in the subtree and is not readable in the subtree proper.
However, "cgroup.procs" can be written to from anywhere in the subtree
to migrate all threads of the matching process to the cgroup.
-->
スレッド化ドメイン cgroup は全サブツリーに対するリソースドメインとして働き、スレッドはサブツリー内に分散する一方、すべてのプロセスはスレッド化ドメイン cgroup 内に存在するとみなされます。スレッド化ドメイン cgroup 内の "cgroup.procs" はサブツリー内のすべてのプロセスの PID を含みます。そしてサブツリー固有のものからは読み取れません。しかし、"cgroup.procs" は、サブツリー内のどこからでも書込みでき、マッチするプロセスの全スレッドを cgroup に移動させられます。

<!--
Only threaded controllers can be enabled in a threaded subtree.  When
a threaded controller is enabled inside a threaded subtree, it only
accounts for and controls resource consumptions associated with the
threads in the cgroup and its descendants.  All consumptions which
aren't tied to a specific thread belong to the threaded domain cgroup.
-->
スレッド化サブツリー内では、スレッド化コントローラのみ有効にできます。スレッド化サブツリー内でスレッド化コントローラが有効化された時、cgroup とその子孫内のスレッドに関連するリソース消費だけが対象となり、コントロールされます。特定のスレッドに結びついていない消費すべてが、スレッド化ドメイン cgroup に属します。

<!--
Because a threaded subtree is exempt from no internal process
constraint, a threaded controller must be able to handle competition
between threads in a non-leaf cgroup and its child cgroups.  Each
threaded controller defines how such competitions are handled.
-->
スレッド化サブツリーは内部プロセスを持たない制約を免除されるため、スレッド化コントローラはリーフ cgroup に属さないスレッドと子 cgroup のスレッド間の競合を扱えなければなりません。スレッド化コントローラはそれぞれ、どのように競合を扱うかを定義しなければいけません。

<!-- 2-3. [Un]populated Notification -->
[un]populated 通知
-----------------

<!--
Each non-root cgroup has a "cgroup.events" file which contains
"populated" field indicating whether the cgroup's sub-hierarchy has
live processes in it.  Its value is 0 if there is no live process in
the cgroup and its descendants; otherwise, 1.  poll and [id]notify
events are triggered when the value changes.  This can be used, for
example, to start a clean-up operation after all processes of a given
sub-hierarchy have exited.  The populated state updates and
notifications are recursive.  Consider the following sub-hierarchy
where the numbers in the parentheses represent the numbers of processes
in each cgroup.
-->
root 以外の cgroup それぞれには "cgroup.events" ファイルがあります。このファイルには、cgroup のどのサブ階層内に生存しているプロセスが存在するかを示す "populated" フィールドを含んでいます。もし生存しているプロセスがその cgroup とその子孫に存在していなければ、この値は 0、そうでない場合は 1 となります。この値が変わった時には、poll と [id]notify イベントがトリガされます。これは、例えば、指定したサブ階層のすべてのプロセスが exit したあとに、クリーンアップ操作を開始するのに使えます。populated のステータスの更新と通知は再帰的に行われます。下記のサブ階層のカッコ内の数字は、それぞれの cgroup のプロセス数を表すとします。

```
  A(4) - B(0) - C(1)
              \ D(0)
```

<!--
A, B and C's "populated" fields would be 1 while D's 0.  After the one
process in C exits, B and C's "populated" fields would flip to "0" and
file modified events will be generated on the "cgroup.events" files of
both cgroups.
-->
A、B、C の "populated" フィールドは 1、D のフィールドは 0 となるでしょう。C 内のプロセスが exit した後、B と C の "populated" フィールドは 0 になります。そして、ファイルが変化したイベントが、両方の cgroup の"cgroup.events" ファイル上で生成されるでしょう。

<!-- 2-4. Controlling Controllers -->
コントローラの制御
--------------

<!-- 2-4-1. Enabling and Disabling -->
### 有効化と無効化

<!--
Each cgroup has a "cgroup.controllers" file which lists all
controllers available for the cgroup to enable.
-->
cgroup にはそれぞれ、"cgroup.controllers" というファイルがあります。このファイルは、その cgroup で有効になっていて、使用できるすべてのコントローラがリストされています。

```
  # cat cgroup.controllers
  cpu io memory
```

<!--
No controller is enabled by default.  Controllers can be enabled and
disabled by writing to the "cgroup.subtree_control" file.
-->
デフォルトでは有効になっているコントローラはありません。コントローラは"cgroup.subtree_control" ファイルに書き込むことで有効化・無効化できます。

```
  # echo "+cpu +memory -io" > cgroup.subtree_control
```

<!--
Only controllers which are listed in "cgroup.controllers" can be
enabled.  When multiple operations are specified as above, either they
all succeed or fail.  If multiple operations on the same controller
are specified, the last one is effective.
-->
"cgroup.controllers" にリストされているコントローラのみが有効化できます。前記のように複数の操作を指定した場合、すべてが成功するか失敗するかのどちらかです。同じコントローラに対して複数の操作をした場合、最後の操作だけが有効です。

<!--
Enabling a controller in a cgroup indicates that the distribution of
the target resource across its immediate children will be controlled.
Consider the following sub-hierarchy.  The enabled controllers are
listed in parentheses.
-->
cgroup 内でコントローラを有効化することは、直近の子 cgroup 全体で、ターゲットとなるリソースの分配がコントロールされるということを意味します。
下記のサブ階層を考えてみましょう。有効化されたコントローラをカッコ内でリストします。

```
  A(cpu,memory) - B(memory) - C()
                            \ D()
```

<!--
As A has "cpu" and "memory" enabled, A will control the distribution
of CPU cycles and memory to its children, in this case, B.  As B has
"memory" enabled but not "CPU", C and D will compete freely on CPU
cycles but their division of memory available to B will be controlled.
-->
A では "cpu" と "memory" を有効にしていますので、A はその子 (この場合は B) に対して CPU サイクルとメモリの分配をコントロールします。B では"memory" を有効にしていますが、"CPU" は有効にしていませんので、C と D は CPU サイクルを自由に競争します。しかし、B でメモリの分割を有効にしているので、メモリはコントロールされます。

<!--
As a controller regulates the distribution of the target resource to
the cgroup's children, enabling it creates the controller's interface
files in the child cgroups.  In the above example, enabling "cpu" on B
would create the "cpu." prefixed controller interface files in C and
D.  Likewise, disabling "memory" from B would remove the "memory."
prefixed controller interface files from C and D.  This means that the
controller interface files - anything which doesn't start with
"cgroup." are owned by the parent rather than the cgroup itself.
-->
コントローラは、子 cgroup に対するターゲットとなるリソースの分配を調節するので、コントローラを有功にすると、子 cgroup 内にコントローラ用のインターフェースファイルを作成します。先の例では、B で "cpu" を有効にすると、C と D 内で "cpu." というプレフィックスが付いたインターフェースファイルができるでしょう。同様に、B で "memory" を無効にすると、C と D 以降では "memory." というプレフィックスが付いたインターフェースファイルは削除されます。これは、コントローラインターフェースファイル("cgroup." で始まらないすべてのファイル) は、その cgroup 自身でなく、親が所有するということを意味します。

<!-- 2-4-2. Top-down Constraint -->
### トップダウン制約

<!--
Resources are distributed top-down and a cgroup can further distribute
a resource only if the resource has been distributed to it from the
parent.  This means that all non-root "cgroup.subtree_control" files
can only contain controllers which are enabled in the parent's
"cgroup.subtree_control" file.  A controller can be enabled only if
the parent has the controller enabled and a controller can't be
disabled if one or more children have it enabled.
-->
リソースはトップダウンで分配されます。そして cgroup はリソースが親から分配されているときだけ、さらにリソースを分配できます。これは、すべてのルート以外の "cgroup.subtree_control" ファイルは、親の"cgroup.subtree_control" ファイルで有効になっているコントローラだけを含むことができるということです。コントローラは親が有効にしているコントローラを持っているときのみ有効にできます。そして、コントローラはひとつ以上の子供が有効にしている場合は無効化できません。



<!-- No Internal Process Constraint -->
### 内部にプロセスを持たない制約

<!--
Non-root cgroups can only distribute resources to their children when
they don't have any processes of their own.  In other words, only
cgroups which don't contain any processes can have controllers enabled
in their "cgroup.subtree_control" files.
-->
ルート以外の cgroup は、自身がプロセスを一切持たないときだけ、子供にリソースを分配できます。言い換えると、いかなるプロセスも含まれていないcgroup のみが、"cgroup.subtree_control" ファイル内でコントローラを有効にできるということです。

<!--
This guarantees that, when a controller is looking at the part of the
hierarchy which has it enabled, processes are always only on the
leaves.  This rules out situations where child cgroups compete against
internal processes of the parent.
-->
これは、あるコントローラが有効になっている階層の一部を見ているとき、プロセスは常にリーフ（葉）部分にのみ含まれることを保証します。これにより、子 cgroup が親の内部プロセスと競合することができなくなります。

<!--
The root cgroup is exempt from this restriction.  Root contains
processes and anonymous resource consumption which can't be associated
with any other cgroups and requires special treatment from most
controllers.  How resource consumption in the root cgroup is governed
is up to each controller.
-->
ルート cgroup はこの制約の適用外です。ルートはプロセスを含み、他のcgroup と関連づけられない匿名のリソース消費を含みます。そして、ほとんどのコントローラから特別な扱いを必要とします。ルート cgroup のリソース消費をどのように管理するかは、それぞれのコントローラに任されています。

<!--
Note that the restriction doesn't get in the way if there is no
enabled controller in the cgroup's "cgroup.subtree_control".  This is
important as otherwise it wouldn't be possible to create children of a
populated cgroup.  To control resource distribution of a cgroup, the
cgroup must create children and transfer all its processes to the
children before enabling controllers in its "cgroup.subtree_control"
file.
-->
この制約は、cgroup の "cgroup.subtree_control" で有効になっているコントローラがない場合に障害になることはありません。プロセスを含む cgroup の子供を作れないので、これは重要です。cgroup のリソース配分をコントロールするために、"cgroup.subtree_control" ファイルでコントローラを有効にする前に、cgroup は子供を作らなければならず、コントロールするプロセスすべてを子供に移動させなければなりません。


<!-- Delegation -->
権限委譲
------

<!-- Model of Delegation -->
### 権限委譲モデル

<!--
A cgroup can be delegated in two ways.  First, to a less privileged
user by granting write access of the directory and its "cgroup.procs",
"cgroup.threads" and "cgroup.subtree_control" files to the user.
Second, if the "nsdelegate" mount option is set, automatically to a
cgroup namespace on namespace creation.
-->
cgroup はふたつの方法で権限委譲できます。ひとつ目は、特権を持たないユーザのために、ディレクトリと、その中の "cgroup.procs"、"cgroup.threads"、"cgroup.subtree_control" ファイルにユーザの書き込み権を与えることによってです。ふたつ目は、"nsdelegate" マウントオプションが設定されている場合、名前空間の作成時に、自動的に cgroup namespace へ移動します。

<!--
Because the resource control interface files in a given directory
control the distribution of the parent's resources, the delegatee
shouldn't be allowed to write to them.  For the first method, this is
achieved by not granting access to these files.  For the second, the
kernel rejects writes to all files other than "cgroup.procs" and
"cgroup.subtree_control" on a namespace root from inside the
namespace.
-->
与えられたディレクトリにあるリソースコントロールファイルは、親のリソースの分配をコントロールするので、委任された側はそれらへの書き込みを許されるべきではありません。最初の方法では、これはこれらのファイルへのアクセスを許可しないことで実現されます。ふたつ目の方法では、名前空間内から名前空間の root にある "cgroup.procs"、"cgroup.subtree_control" 以外のすべてのファイルへのアクセスを拒否することで実現されます。

<!--
The end results are equivalent for both delegation types.  Once
delegated, the user can build sub-hierarchy under the directory,
organize processes inside it as it sees fit and further distribute the
resources it received from the parent.  The limits and other settings
of all resource controllers are hierarchical and regardless of what
happens in the delegated sub-hierarchy, nothing can escape the
resource restrictions imposed by the parent.
-->
最終的な結果は、両方の権限委譲タイプで同じです。一度権限委譲されると、ユーザはディレクトリ以下にサブ階層を作り、その中でプロセスをまとめ、親から受け取ったリソースをすべて分配できます。すべてのリソースコントローラの制限や他の設定は階層的であり、委譲されたサブ階層内で何が起ころうとも、親から受け取ったリソース制限を逃れることはできません。

<!--
Currently, cgroup doesn't impose any restrictions on the number of
cgroups in or nesting depth of a delegated sub-hierarchy; however,
this may be limited explicitly in the future.
-->
現時点では、cgroup は cgroup の数、移譲されたサブ階層のネストの深さについては何も制限されていません。しかし、将来は明確に制限されるかもしれません。


<!-- 2-5-2. Delegation Containment -->
### 権限移譲の制約

<!--
A delegated sub-hierarchy is contained in the sense that processes
can't be moved into or out of the sub-hierarchy by the delegatee.
-->
権限委譲されたサブ階層は、権限委譲された側がサブ階層の外や、外からサブ階層中へプロセスを移動できないという意味を含みます。

<!--
For delegations to a less privileged user, this is achieved by
requiring the following conditions for a process with a non-root euid
to migrate a target process into a cgroup by writing its PID to the
"cgroup.procs" file.
-->
権限の低いユーザへの権限委譲は、非 root euid を持つプロセスが、"cgroup.procs" ファイルへ PID を書き込み、ターゲットとなるプロセスを cgroup 内に移動するために、以下の条件を要求することで実現されます。

<!--
\- The writer must have write access to the "cgroup.procs" file.
-->
- 書き込み側は "cgroup.procs" ファイルへの書き込み権を持っていなければなりません。

<!--
\- The writer must have write access to the "cgroup.procs" file of the
   common ancestor of the source and destination cgroups.
   -->
- 書きこみ側はソースとデスティネーションの cgroup の共通の祖先の "cgroup.procs" ファイルへの書き込み権を持っていなければなりません。

<!--
The above three constraints ensure that while a delegatee may migrate
processes around freely in the delegated sub-hierarchy it can't pull
in from or push out to outside the sub-hierarchy.
-->
以上の 3 つの制約は、移譲された側が移譲されたサブ階層内で自由にプロセスを移動できる一方、サブ階層の外から内へ、もしくはサブ階層から外へプロセスを移動できないことを保証します。

<!--
For an example, let's assume cgroups C0 and C1 have been delegated to
user U0 who created C00, C01 under C0 and C10 under C1 as follows and
all processes under C0 and C1 belong to U0.
-->
例えば、cgroup C0 と C1 は、U0 に権限委譲されているとします。U0 は C0 以下に C00 と C01、C1 以下に C1 を以下のように作成しています。そして、C0 と C1 以下のすべてのプロセスは U0 に属していると仮定してみましょう。

```
  ~~~~~~~~~~~~~ - C0 - C00
  ~ cgroup    ~      \ C01
  ~ hierarchy ~
  ~~~~~~~~~~~~~ - C1 - C10
```

<!--
Let's also say U0 wants to write the PID of a process which is
currently in C10 into "C00/cgroup.procs".  U0 has write access to the
file; however, the common ancestor of the source cgroup C10 and the
destination cgroup C00 is above the points of delegation and U0 would
not have write access to its "cgroup.procs" files and thus the write
will be denied with -EACCES.
-->
U0 が、現在 C10 内にいるプロセスの PID を "C00/cgroup.procs" に書きたい！と言ってみましょう。U0 はファイルへの書き込み権を持っています。しかし、ソース cgroup の C10 とデスティネーション cgroup C00 の共通の祖先は、権限移譲の点では立場が上で、U0 はその祖先の "cgroup.procs" に対する書き込み権を持っていません。ですので、この書きこみは -EACCES で拒否されます。

<!--
For delegations to namespaces, containment is achieved by requiring
that both the source and destination cgroups are reachable from the
namespace of the process which is attempting the migration.  If either
is not reachable, the migration is rejected with -ENOENT.
-->
名前空間への権限委譲では、移動元と移動先の cgroup 両方が、移動しようとするプロセスの名前空間から到達できることで封じ込めが達成されます。どちらか一方の到達できなければ、移動は `-ENOENT` で拒否されます。

<!-- 2-6. Guidelines -->
ガイドライン
---------

<!-- Organize Once and Control -->
### 一度だけ組織化してコントロールする

<!--
Migrating a process across cgroups is a relatively expensive operation
and stateful resources such as memory are not moved together with the
process.  This is an explicit design decision as there often exist
inherent trade-offs between migration and various hot paths in terms
of synchronization cost.
-->
cgroup 間でプロセスを移動することは比較的コストの高い操作であり、メモリのようなステートフルなリソースはプロセスと一緒には移動しません。同期コストの観点で、移動と色々なホットパスの間の固有のトレードオフが存在するので、これは明確なデザインの決定です。

<!--
As such, migrating processes across cgroups frequently as a means to
apply different resource restrictions is discouraged.  A workload
should be assigned to a cgroup according to the system's logical and
resource structure once on start-up.  Dynamic adjustments to resource
distribution can be made by changing controller configuration through
the interface files.
-->
このように、異なるリソース制限を適用するための手段として、頻繁にcgroup 間でプロセスを移動することは推奨されません。作業負荷は、起動時に一回だけ、システムの論理的な、そしてリソースの構造によって、cgroup に割り当てられるべきです。リソース配分の動的な調整は、インターフェースファイル経由でコントローラの設定を変えることにより行えます。


<!-- Avoid Name Collisions -->
### 名前の衝突を避ける

<!--
Interface files for a cgroup and its children cgroups occupy the same
directory and it is possible to create children cgroups which collide
with interface files.
-->
cgroup 用のインターフェースファイルと、その子 cgroup は同じディレクトリを使用します。そして、インターフェースファイルと衝突する名前の子cgroup を作ることができます。

<!--
All cgroup core interface files are prefixed with "cgroup." and each
controller's interface files are prefixed with the controller name and
a dot.  A controller's name is composed of lower case alphabets and
'_'s but never begins with an '_' so it can be used as the prefix
character for collision avoidance.  Also, interface file names won't
start or end with terms which are often used in categorizing workloads
such as job, service, slice, unit or workload.
-->
cgroup コアのインターフェースファイルはすべて "cgroup." のプレフィックスを持ち、コントローラ固有のインターフェースファイルはコントローラ名に "." を付与したプレフィックスを持ちます。コントローラ名は小文字のアルファベットと "\_" で構成されますが、"\_" で始まることはありません。ですので、衝突を避けるために "\_" で始めることは可能です。そして、インターフェースファイル名は、job、service、slice、unit、workload といった作業負荷関連で良く使われるような単語で始まったり終わったりしません。

<!--
cgroup doesn't do anything to prevent name collisions and it's the
user's responsibility to avoid them.
-->
cgroup は名前の衝突を防ぐようなことは一切しません。それを防ぐのはユーザです。

<!-- Resource Distribution Models -->
リソース分配のモデル
================

<!--
cgroup controllers implement several resource distribution schemes
depending on the resource type and expected use cases.  This section
describes major schemes in use along with their expected behaviors.
-->
cgroup コントローラは、リソースのタイプと期待するユースケースに依存する、いくつかのリソース分配スキームを実装しています。このセクションでは、期待する振る舞いとあわせて、使われている主なスキーマを説明します。


<!-- Weights -->
ウェイト
------

<!--
A parent's resource is distributed by adding up the weights of all
active children and giving each the fraction matching the ratio of its
weight against the sum.  As only children which can make use of the
resource at the moment participate in the distribution, this is
work-conserving.  Due to the dynamic nature, this model is usually
used for stateless resources.
-->
親のリソースは、アクティブな子供のすべてのウェイトを合計し、合計に対してそれぞれに与えたウェイトの比とマッチする割合で分配されます。ちょうどその瞬間にリソースを使える子供だけが分配に参加できるので、これは「仕事量保存型」です。

<!--
All weights are in the range [1, 10000] with the default at 100.  This
allows symmetric multiplicative biases in both directions at fine
enough granularity while staying in the intuitive range.
-->
すべてのウェイトは [1, 10000] の間で、デフォルト値は 100 です。直感的な範囲に収まっている間は、十分に細かい粒度で両方のディレクトリ内で対称な乗法のバイアスが可能です。

<!--
As long as the weight is in range, all configuration combinations are
valid and there is no reason to reject configuration changes or
process migrations.
-->
ウェイトが範囲内である限りは、あらゆる設定の組み合わせが有効で、設定の変更やプロセスの移動を拒否する理由はありません。

<!--
"cpu.weight" proportionally distributes CPU cycles to active children
and is an example of this type.
-->
"cpu.weight" は、アクティブな子に対して比例的に CPU を分配します。これがこのタイプの例です。


<!-- Limits -->
制限
----

<!--
A child can only consume upto the configured amount of the resource.
Limits can be over-committed - the sum of the limits of children can
exceed the amount of resource available to the parent.
-->
子は設定量までだけリソースを消費できます。制限はオーバーコミットできます。子の制限の合計は親で利用可能なリソースの量を超えられます。

<!--
Limits are in the range [0, max] and defaults to "max", which is noop.
-->
制限は [0, max] の範囲で、デフォルトは "max" です。"max" は制限なし (プロセスに対してなんの処理もしない) です。

<!--
As limits can be over-committed, all configuration combinations are
valid and there is no reason to reject configuration changes or
process migrations.
-->
制限はオーバーコミットできるので、あらゆる設定の組み合わせが有効で、設定の変更やプロセスの移動を拒否する理由はありません。

<!--
"io.max" limits the maximum BPS and/or IOPS that a cgroup can consume
on an IO device and is an example of this type.
-->
"io.max" は cgroup が IO デバイス上で消費できる BPS と IOPS のどちらかか、両方を制限します。これがこのタイプの例です。


<!-- Protections -->
保護
----

<!--
A cgroup is protected to be allocated upto the configured amount of
the resource if the usages of all its ancestors are under their
protected levels.  Protections can be hard guarantees or best effort
soft boundaries.  Protections can also be over-committed in which case
only upto the amount available to the parent is protected among
children.
-->
cgroup は、すべての祖先が保護レベル以下である場合は、設定したリソース量まで割り当てられることは保護されています。保護は強く保証することも、ベストエフォートでのソフトな制限とすることも可能です。この場合、親が利用可能な量までだけが子の間で保護されます。

<!--
Protections are in the range [0, max] and defaults to 0, which is
noop.
-->
保護は [0, max] の範囲であり、デフォルトは 0 です。これは何もしません。

<!--
As protections can be over-committed, all configuration combinations
are valid and there is no reason to reject configuration changes or
process migrations.
-->
保護はオーバーコミットできるので、あらゆる設定の組み合わせが有効で、設定の変更やプロセスの移動を拒否する理由はありません。

<!--
"memory.low" implements best-effort memory protection and is an
example of this type.
-->
"memory.low" はベストエフォートのメモリ保護を実装しています。これがこのタイプの例です。


<!-- Allocations -->
割り当て
------

<!--
A cgroup is exclusively allocated a certain amount of a finite
resource.  Allocations can't be over-committed - the sum of the
allocations of children can not exceed the amount of resource
available to the parent.
-->
cgroup には、独占的にある量の有限のリソースが割り当てられます。割り当てはオーバーコミットできません。子への割り当て量の和は、親が利用可能なリソース量を超えられません。

<!--
Allocations are in the range [0, max] and defaults to 0, which is no
resource.
-->
割り当ては [0, max] の範囲で、デフォルトは 0 です。これはリソースがない状態です。

<!--
As allocations can't be over-committed, some configuration
combinations are invalid and should be rejected.  Also, if the
resource is mandatory for execution of processes, process migrations
may be rejected.
-->
割り当てはオーバーコミットできませんので、設定の組み合わせには有効でない組み合わせがあり、そのような組み合わせは拒否しなくてはいけません。そして、リソースがプロセスの実行のために強制された場合、プロセスの移動は拒否されます。

<!--
"cpu.rt.max" hard-allocates realtime slices and is an example of this
type.
-->
"cpu.rt.max" は強制的に割り当てられるリアルタイムのスライスです。これがこのタイプの例です。


<!-- Interface Files -->
インターフェースファイル
===================

<!-- Format -->
フォーマット
---------

<!--
All interface files should be in one of the following formats whenever
possible.
-->
インターフェースファイルは可能なら、以下のうちの一つのフォーマットであるべきです。

<!--
  New-line separated values
  (when only one value can be written at once)
  -->
  New-line で区切られた値
  (一度に一つの値だけが書いても良い場合)

	VAL0\n
	VAL1\n
	...

<!--
  Space separated values
  (when read-only or multiple values can be written at once)
  -->
  スペース区切りの値
  (読み込み専用の場合、もしくは複数の値を一度に書き込める場合)

	VAL0 VAL1 ...\n

<!--
  Flat keyed
  -->
  フラットなキー

	KEY0 VAL0\n
	KEY1 VAL1\n
	...

<!--
  Nested keyed
  -->
  ネストしたキー

	KEY0 SUB_KEY0=VAL00 SUB_KEY1=VAL01...
	KEY1 SUB_KEY0=VAL10 SUB_KEY1=VAL11...
	...

<!--
For a writable file, the format for writing should generally match
reading; however, controllers may allow omitting later fields or
implement restricted shortcuts for most common use cases.
-->
書き込めるファイルの場合、書きこみフォーマットは通常は読み込み時とマッチすべきです。しかし、コントローラは、後でフィールドを省略できるようにしたり、最も一般的なユースケースのために制限されたショートカットを実装したりしても構いません。

<!--
For both flat and nested keyed files, only the values for a single key
can be written at a time.  For nested keyed files, the sub key pairs
may be specified in any order and not all pairs have to be specified.
-->
フラットとネストしたキーのあるファイルは、単一のキーの値のみを一度に書き込めます。ネストしたキーのファイルは、サブキーのペアは任意の順序で指定して構いません。そして、すべてのペアを指定する必要はありません。


<!-- Conventions -->
規約
----

<!--
- Settings for a single feature should be contained in a single file.
-->
- 単一の機能に対する設定は単一のファイルに含めなければいけません

<!--
- The root cgroup should be exempt from resource control and thus
  shouldn't have resource control interface files.  Also,
  informational files on the root cgroup which end up showing global
  information available elsewhere shouldn't exist.
  -->
- root cgroup はリソースコントロールから免除されなければいけません。それゆえ、コントロールのためのインターフェースファイルを持ってはいけません。そして、最終的に利用可能なグローバル情報を表示する root cgroup のインフォメーションファイルが他の場所にあってはなりません。

<!--
- If a controller implements weight based resource distribution, its
  interface file should be named "weight" and have the range [1,
  10000] with 100 as the default.  The values are chosen to allow
  enough and symmetric bias in both directions while keeping it
  intuitive (the default is 100%).
  -->
- コントローラがウェイトベースのリソース分配を実装する場合、インターフェースファイルは "weight" と名付け、範囲として [1,10000] をもち、デフォルトが 100 でなければなりません。値は、十分な量で、両方向に対称にでき、さらに直感的な状態であるように選択されます。(デフォルト値は 100%)

<!--
- If a controller implements an absolute resource guarantee and/or
  limit, the interface files should be named "min" and "max"
  respectively.  If a controller implements best effort resource
  guarantee and/or limit, the interface files should be named "low"
  and "high" respectively.

  In the above four control files, the special token "max" should be
  used to represent upward infinity for both reading and writing.
  -->
- コントローラが絶対値でのリソース保証と制限の両方かどちらかを実装する場合、インターフェースファイルはそれぞれ "min" と "max" と名付けなければなりません。コントローラがベストエフォートのリソース保証と制限の両方かどちらかを実装する場合、インターフェースファイルはそれぞれ "low" と "high" と名付けなければなりません。この 4 つのコントロールファイル内では、読み取りと書きこみの両方で、特別なトークン "max" を上限が無限であることを表すために使用しなければなりません。

<!--
- If a setting has a configurable default value and keyed specific
  overrides, the default entry should be keyed with "default" and
  appear as the first entry in the file.

  The default value can be updated by writing either "default $VAL" or
  "$VAL".

  When writing to update a specific override, "default" can be used as
  the value to indicate removal of the override.  Override entries
  with "default" as the value must not appear when read.

  For example, a setting which is keyed by major:minor device numbers
  with integer values may look like the following.

    # cat cgroup-example-interface-file
    default 150
    8:0 300

  The default value can be updated by

    # echo 125 > cgroup-example-interface-file

  or

    # echo "default 125" > cgroup-example-interface-file

  An override can be set by

    # echo "8:16 170" > cgroup-example-interface-file

  and cleared by

    # echo "8:0 default" > cgroup-example-interface-file
    # cat cgroup-example-interface-file
    default 125
    8:16 170
-->
- もし、設定が変更可能なデフォルト値を持っている場合で、キーを指定して値の上書きができる場合、デフォルトエントリは "default" というキーで、ファイルの最初の行に現れなければなりません。デフォルト値は、"default $VAL" もしくは "$VAL" という指定で更新できます。デフォルト値の上書きを更新する際、上書きされた値の消去 (して元のデフォルト値に戻すこと) を指示するための値として "default" が使えます。読み込み時には、値が "default" というデフォルト値を表示してはいけません。例えば、整数値の major:minor デバイス番号がキーである設定は以下のようになるでしょう。
```
    # cat cgroup-example-interface-file
    default 150
    8:0 300
```
  デフォルト値は
```
    # echo 125 > cgroup-example-interface-file
```
  もしくは
```
    # echo "default 125" > cgroup-example-interface-file
```
  のように更新できます。
  上書きは以下のように行います。
```
    # echo "8:16 170" > cgroup-example-interface-file
```
  そして上書きした値のクリアは
```
    # echo "8:0 default" > cgroup-example-interface-file
    # cat cgroup-example-interface-file
    default 125
    8:16 170
```
  のようになります。

<!--
- For events which are not very high frequency, an interface file
  "events" should be created which lists event key value pairs.
  Whenever a notifiable event happens, file modified event should be
  generated on the file.
  -->
- それほど高頻度でないイベントに対して、イベントキーと値のペアからなるインターフェースファイル "event" を作成しなければなりません。通知すべきイベントが起きたときはいつでも、ファイルが更新されたイベントがファイル上で生成されなければなりません。

<!-- Core Interface Files -->
コアインターフェースファイル
----------------------

<!--
All cgroup core files are prefixed with "cgroup."
-->
cgroup コアファイルはすべて "cgroup." というプレフィックスが付与されま
す。

### cgroup.type

        A read-write single value file which exists on non-root
        cgroups.

        When read, it indicates the current type of the cgroup, which
        can be one of the following values.

        - "domain" : A normal valid domain cgroup.

        - "domain threaded" : A threaded domain cgroup which is
          serving as the root of a threaded subtree.

        - "domain invalid" : A cgroup which is in an invalid state.
          It can't be populated or have controllers enabled.  It may
          be allowed to become a threaded cgroup.

        - "threaded" : A threaded cgroup which is a member of a
          threaded subtree.

        A cgroup can be turned into a threaded cgroup by writing
        "threaded" to this file.

###  cgroup.procs

<!--
	A read-write new-line separated values file which exists on
	all cgroups.
-->
すべての cgroup に存在する、読み書き可能な new-line で区切られた値のファイルです。

<!--
	When read, it lists the PIDs of all processes which belong to
	the cgroup one-per-line.  The PIDs are not ordered and the
	same PID may show up more than once if the process got moved
	to another cgroup and then back or the PID got recycled while
	reading.
-->
読んだ場合は、cgroup に属するプロセスがすべて、一行にひとつずつリストされます。PID は順番には並ばず、同じ PID が一回以上現れる可能性があります。これは、プロセスが別の cgroup に移動した後戻ってきたり、ファイルを読んでいる間に PID が再利用された場合などに起こります。

<!--
	A PID can be written to migrate the process associated with
	the PID to the cgroup.  The writer should match all of the
	following conditions.
-->
PID に関連したプロセスを cgroup に移動させるために PID を書き込めます。書き手は以下の条件をすべて満たす必要があります。

<!--
	- It must have write access to the "cgroup.procs" file.
-->
- "cgroup.procs" ファイルへの書き込み権を持っていなければいけません。

<!--
	- It must have write access to the "cgroup.procs" file of the
	  common ancestor of the source and destination cgroups.
-->
- 移動元と移動先に cgroup の共通の祖先の "cgroup.procs" ファイルへの書き込み権を持っていなければいけません。

<!--
	When delegating a sub-hierarchy, write access to this file
	should be granted along with the containing directory.
-->
サブ階層に権限委譲を行う場合は、このファイルに対する書き込み権は、ファイルが含まれるディレクトリと一緒に付与される必要があります。

### cgroup.threads

        A read-write new-line separated values file which exists on
        all cgroups.

        When read, it lists the TIDs of all threads which belong to
        the cgroup one-per-line.  The TIDs are not ordered and the
        same TID may show up more than once if the thread got moved to
        another cgroup and then back or the TID got recycled while
        reading.

        A TID can be written to migrate the thread associated with the
        TID to the cgroup.  The writer should match all of the
        following conditions.

        - It must have write access to the "cgroup.threads" file.

        - The cgroup that the thread is currently in must be in the
          same resource domain as the destination cgroup.

        - It must have write access to the "cgroup.procs" file of the
          common ancestor of the source and destination cgroups.

        When delegating a sub-hierarchy, write access to this file
        should be granted along with the containing directory.

###  cgroup.controllers

<!--
	A read-only space separated values file which exists on all
	cgroups.
-->
読み込み専用のスペース区切りの値のファイルです。すべての cgroup に存在します。

<!--
	It shows space separated list of all controllers available to
	the cgroup.  The controllers are not ordered.
-->
その cgroup で使えるすべてのコントローラのスペース区切りのリストを示します。コントローラの並べかえは行われません。

###  cgroup.subtree_control

<!--
	A read-write space separated values file which exists on all
	cgroups.  Starts out empty.
-->
読み書き可能なスペース区切りのファイルです。すべての cgroup に存在します。最初は空のファイルです。

<!--
	When read, it shows space separated list of the controllers
	which are enabled to control resource distribution from the
	cgroup to its children.
-->
読み込み時は、コントローラのスペース区切りのリストを示します。リストされたコントローラは、その cgroup から子供に対してリソース分配をコントロールできるコントローラです。

<!--
	Space separated list of controllers prefixed with '+' or '-'
	can be written to enable or disable controllers.  A controller
	name prefixed with '+' enables the controller and '-'
	disables.  If a controller appears more than once on the list,
	the last one is effective.  When multiple enable and disable
	operations are specified, either all succeed or all fail.
-->
'+' または '-' が頭に付いたコントローラをスペース区切りで書き込むと、コントローラを有効にしたり無効にしたりできます。'+' が頭に付いたコントローラは有効になり、'-' が頭に付いたコントローラは無効になります。リストに同じコントローラが複数回現れた場合は、最後に指定されたコントローラが有効になります。複数の有効化、無効化の操作が指定した場合、すべて成功するか、すべて失敗するかのどちらかです。

###  cgroup.events

<!--
	A read-only flat-keyed file which exists on non-root cgroups.
	The following entries are defined.  Unless specified
	otherwise, a value change in this file generates a file
	modified event.
-->
読み込み専用のフラットなキーのファイルです。root 以外の cgroup に存在します。以下のエントリが定義されています。他に特に規定がなければ、このファイルの値の変更は、ファイルが変更されたイベントを生成します。

#### populated

<!--
		1 if the cgroup or its descendants contains any live
		processes; otherwise, 0.
-->
その cgroup かその cgroup の子孫が実行中のプロセスを含む場合は 1、そうでなければ 0 となります。

###  cgroup.max.descendants
        A read-write single value files.  The default is "max".

        Maximum allowed number of descent cgroups.
        If the actual number of descendants is equal or larger,
        an attempt to create a new cgroup in the hierarchy will fail.

###  cgroup.max.depth
        A read-write single value files.  The default is "max".

        Maximum allowed descent depth below the current cgroup.
        If the actual descent depth is equal or larger,
        an attempt to create a new child cgroup will fail.

###  cgroup.stat
        A read-only flat-keyed file with the following entries:

#### nr_descendants
                Total number of visible descendant cgroups.

#### nr_dying_descendants
                Total number of dying descendant cgroups. A cgroup becomes
                dying after being deleted by a user. The cgroup will remain
                in dying state for some time undefined time (which can depend
                on system load) before being completely destroyed.

                A process can't enter a dying cgroup under any circumstances,
                a dying cgroup can't revive.

                A dying cgroup can consume system resources not exceeding
                limits, which were active at the moment of cgroup deletion.

<!-- Controllers -->
コントローラ
=========

<!-- 5-1. CPU -->
CPU
---

<!--
[NOTE: The interface for the cpu controller hasn't been merged yet]
-->
[注意: CPU コントローラ用のインターフェースはまだマージされていません]

<!--
The "cpu" controllers regulates distribution of CPU cycles.  This
controller implements weight and absolute bandwidth limit models for
normal scheduling policy and absolute bandwidth allocation model for
realtime scheduling policy.
-->
"cpu" コントローラは CPU サイクルの分配を調整します。このコントローラは、通常のスケジューリングポリシー用に weight と絶対値バンド幅制限のモデルを実装します。また、リアルタイムスケジューリングポリシー用に絶対値バンド幅制限を実装します。


<!-- 5-1-1. CPU Interface Files -->
#### 5-1-1. CPU インターフェースファイル

<!--
All time durations are in microseconds.
-->
すべて、時間の単位はマイクロ秒です。

####  cpu.stat

<!--
	A read-only flat-keyed file which exists on non-root cgroups.
-->
読み込み専用のフラットなキーのファイルです。このファイルは root 以外の cgroup に存在します。

<!--
	It reports the following six stats.
-->
以下の 6 つの統計値をレポートします。

-  usage_usec
-  user_usec
-  system_usec
-  nr_periods
-  nr_throttled
-  throttled_usec

####  cpu.weight

<!--
	A read-write single value file which exists on non-root
	cgroups.  The default is "100".
-->
読み書き可能な単一の値が書かれたファイルです。root 以外の cgroup に存在します。デフォルトは "100" です。

<!--
	The weight in the range [1, 10000].
-->
weight の範囲は [1, 10000] です。

####  cpu.max

<!--
	A read-write two value file which exists on non-root cgroups.
	The default is "max 100000".
-->
読み書き可能な 2 つの値が書かれたファイルです。root 以外の cgroup に存在します。デフォルトは "max 100000" です。

<!--
	The maximum bandwidth limit.  It's in the following format.
-->
バンド幅の最大値で、以下のようなフォーマットです。

	  $MAX $PERIOD

<!--
	which indicates that the group may consume upto $MAX in each
	$PERIOD duration.  "max" for $MAX indicates no limit.  If only
	one number is written, $MAX is updated.
-->
これは、このグループは $PERIOD の間に最大 $MAX までリソースを消費できることを示します。$MAX の値が "max" である場合は無制限であることを示します。値をひとつだけ書き込んだ場合は $MAX が更新されます。

####  cpu.rt.max

<!--
  [NOTE: The semantics of this file is still under discussion and the
   interface hasn't been merged yet]
-->
  [注意: このファイルのセマンティクスは議論中です。インターフェースはまだマージされていません]

<!--
	A read-write two value file which exists on all cgroups.
	The default is "0 100000".
-->
読み書き可能な 2 つの値が書かれたファイルです。すべての cgroup に存在します。デフォルトは "0 100000" です。

<!--
	The maximum realtime runtime allocation.  Over-committing
	configurations are disallowed and process migrations are
	rejected if not enough bandwidth is available.  It's in the
	following format.
-->
リアルタイムの実行時間の割り当ての最大値です。オーバーコミットとなる設定はできません。そして、十分な帯域が利用できない場合はプロセスの移動は拒否されます。このファイルは以下のようなフォーマットとなります。

	  $MAX $PERIOD

<!--
	which indicates that the group may consume upto $MAX in each
	$PERIOD duration.  If only one number is written, $MAX is
	updated.
-->
これは、このグループは $PERIOD の間に最大 $MAX までリソースを消費できることを示します。$MAX の値が "max" である場合は無制限であることを示します。値をひとつだけ書き込んだ場合は $MAX が更新されます。


Memory
------

<!--
The "memory" controller regulates distribution of memory.  Memory is
stateful and implements both limit and protection models.  Due to the
intertwining between memory usage and reclaim pressure and the
stateful nature of memory, the distribution model is relatively
complex.
-->
"memory" コントローラはメモリの分配を調整します。メモリはステートフルで制限と保護の両方のモデルを実装します。メモリ消費量と回収圧力とメモリのステートフルな性質を結びつけるため、分配モデルは比較的複雑になります。

<!--
While not completely water-tight, all major memory usages by a given
cgroup are tracked so that the total memory consumption can be
accounted and controlled to a reasonable extent.  Currently, the
following types of memory usages are tracked.
-->
完全ではないものの、トータルのメモリ消費がカウントされ、合理的な範囲に制御されることができるように、与えられた cgroup に対するすべての主なメモリ消費が追跡されます。

<!--
- Userland memory - page cache and anonymous memory.
-->
- ユーザランドのメモリ - ページキャッシュとアノニマスメモリ

<!--
- Kernel data structures such as dentries and inodes.
-->
- dentry や inode のようなカーネルデータ構造

<!--
- TCP socket buffers.
-->
TCP ソケットバッファ

<!--
The above list may expand in the future for better coverage.
-->
上記のリストは将来カバーする範囲の改良のために拡張されるかもしれません。

<!-- Memory Interface Files -->
### メモリインターフェースファイル

<!--
All memory amounts are in bytes.  If a value which is not aligned to
PAGE_SIZE is written, the value may be rounded up to the closest
PAGE_SIZE multiple when read back.
-->
すべてのメモリ量はバイトです。PAGE\_SIZE で割り切れない値が書かれている場合、その値は読み出しの際に最も近い PAGE\_SIZE に切り上げられるかもしれません。

####  memory.current

<!--
	A read-only single value file which exists on non-root
	cgroups.
-->
読み込み専用の単一の値のファイルです。root 以外の cgroup に存在します。

<!--
	The total amount of memory currently being used by the cgroup
	and its descendants.
-->
その cgroup と子孫が現在使っているメモリの総量です。

####  memory.low

<!--
	A read-write single value file which exists on non-root
	cgroups.  The default is "0".
-->
読み書き可能な単一の値のファイルです。root 以外の cgroup に存在し、デフォルト値は 0 です。

<!--
	Best-effort memory protection.  If the memory usages of a
	cgroup and all its ancestors are below their low boundaries,
	the cgroup's memory won't be reclaimed unless memory can be
	reclaimed from unprotected cgroups.
-->
ベストエフォートのメモリ保護です。ある cgroup とすべての祖先のメモリ使用量が low 限界より下であれば、保護されていない cgroup からの回収ができない場合をのぞいて、その cgroup のメモリが回収されることはないでしょう。

<!--
	Putting more memory than generally available under this
	protection is discouraged.
-->
この保護の下に一般的に利用可能な以上のメモリを置くことは推奨しません。

####  memory.high

<!--
	A read-write single value file which exists on non-root
	cgroups.  The default is "max".
-->
読み書き可能な単一の値のファイルです。root 以外の cgroup に存在します。デフォルト値は "max" です。

<!--
	Memory usage throttle limit.  This is the main mechanism to
	control memory usage of a cgroup.  If a cgroup's usage goes
	over the high boundary, the processes of the cgroup are
	throttled and put under heavy reclaim pressure.
-->
メモリ使用量スロットルの制限値です。これが cgroup のメモリ使用量をコントロールするためのメインのメカニズムです。cgroup のメモリ使用量が上限を超えた場合、cgroup のプロセスは調節され、厳しい回収圧力の下に置かれます。

<!--
	Going over the high limit never invokes the OOM killer and
	under extreme conditions the limit may be breached.
-->
上限の超過は決して OOM killer を呼び出すことはありません。極限の状態下では、制限値を超えるかもしれません。

####  memory.max

<!--
	A read-write single value file which exists on non-root
	cgroups.  The default is "max".
-->
読み書き可能な単一の値を含むファイルです。root 以外の cgroup に存在します。デフォルト値は "max" です。

<!--
	Memory usage hard limit.  This is the final protection
	mechanism.  If a cgroup's memory usage reaches this limit and
	can't be reduced, the OOM killer is invoked in the cgroup.
	Under certain circumstances, the usage may go over the limit
	temporarily.
-->
メモリ使用量のハードリミットで、最後の保護メカニズムです。cgroup のメモリ使用量がこの制限に達し、減らすことができない場合、OOM killer が cgroup内で呼びだされます。特定の環境下では、使用量が一時的に制限を超えるかもしれません。

<!--
	This is the ultimate protection mechanism.  As long as the
	high limit is used and monitored properly, this limit's
	utility is limited to providing the final safety net.
-->
これは最終的な保護メカニズムです。high の制限を使い、適切にモニタリングされている限り、この制限は最終的なセーフティーネットを提供という役割に限られるでしょう。

####  memory.events

<!--
	A read-only flat-keyed file which exists on non-root cgroups.
	The following entries are defined.  Unless specified
	otherwise, a value change in this file generates a file
	modified event.
-->
読み込み専用のフラットなキーのファイルです。root 以外の cgroup に存在します。以下のエントリが定義されています。特に指定しない限り、このファイルの値の変更はファイルが修正されたイベントを生成します。

##### low

<!--
		The number of times the cgroup is reclaimed due to
		high memory pressure even though its usage is under
		the low boundary.  This usually indicates that the low
		boundary is over-committed.
-->
cgroup で、使用量が low 以下であるにも関わらず、高いメモリ圧力により回収が行われた回数です。これは、通常は low の値がオーバーコミットされていることを示します。

##### high

<!--
		The number of times processes of the cgroup are
		throttled and routed to perform direct memory reclaim
		because the high memory boundary was exceeded.  For a
		cgroup whose memory usage is capped by the high limit
		rather than global memory pressure, this event's
		occurrences are expected.
-->
high の値を超過したため、cgroup のプロセス数が調節され、直接メモリ回収が実行された回数です。グローバルのメモリ圧力よりもhigh の制限でメモリ使用量が制限されている cgroup では、このイベントの発生が起こる可能性があります。

##### max

<!--
		The number of times the cgroup's memory usage was
		about to go over the max boundary.  If direct reclaim
		fails to bring it down, the cgroup goes to OOM state.
-->

cgroupのメモリ使用量が max 制限を超えようとした回数です。直接
メモリ回収がメモリ使用量の減少に失敗した場合、cgroup は OOM 状態に移行します。


##### oom

<!--
		The number of time the cgroup's memory usage was
		reached the limit and allocation was about to fail.

		Depending on context result could be invocation of OOM
		killer and retrying allocation or failing alloction.
-->
cgroup のメモリ消費が制限に達し、割り当てが失敗した回数。

コンテキストに応じて、結果は OOM killer 起動と、割り当ての再試行、また
は割り当ての失敗があります。

Failed allocation in its turn could be returned into
userspace as -ENOMEM or siletly ignored in cases like
disk readahead.  For now OOM in memory cgroup kills
tasks iff shortage has happened inside page fault.

##### oom_kill

<!--
	  oom_kill

		The number of processes belonging to this cgroup
		killed by any kind of OOM killer.
-->

この cgroup に属するプロセスが、あらゆる種類の OOM killer によって
kill された回数。

####  memory.stat

<!--
	A read-only flat-keyed file which exists on non-root cgroups.
-->
読み込み専用のフラットキーなファイルです。root 以外の cgroup に存在します。

<!--
	This breaks down the cgroup's memory footprint into different
	types of memory, type-specific details, and other information
	on the state and past events of the memory management system.
-->
このファイルは cgroup のメモリ使用を異なるタイプのメモリ、タイプごとの詳細、状態に応じた他の情報、メモリ管理システムの過去のイベントに分解します。

<!--
	All memory amounts are in bytes.
-->
メモリ量の単位はすべて byte です。

<!--
	The entries are ordered to be human readable, and new entries
	can show up in the middle. Don't rely on items remaining in a
	fixed position; use the keys to look up specific values!
-->
各エントリは人が読みやすいように並べられます。新しいエントリが途中で現れることもあります。項目が決まった位置にあることを期待しないでください。特定の値を見つけるのにはキーを使いましょう。

#####  anon

<!--
		Amount of memory used in anonymous mappings such as
		brk(), sbrk(), and mmap(MAP_ANONYMOUS)
-->
brk(), sbrk(), mmap(MAP_ANONYMOUS) のような匿名マッピングに使われているメモリ量

#####  file

<!--
		Amount of memory used to cache filesystem data,
		including tmpfs and shared memory.
-->
tmpfs や共有メモリを含む、ファイルシステムデータのキャッシュに使われているメモリ量

          kernel_stack
                Amount of memory allocated to kernel stacks.

          slab
                Amount of memory used for storing in-kernel data
                structures.

          sock
                Amount of memory used in network transmission buffers

          shmem
                Amount of cached filesystem data that is swap-backed,
                such as tmpfs, shm segments, shared anonymous mmap()s

#####  file_mapped

<!--
		Amount of cached filesystem data mapped with mmap()
-->
mmap()でマップされたファイルシステムのキャッシュデータの量

#####  file_dirty

<!--
		Amount of cached filesystem data that was modified but
		not yet written back to disk
-->
変更されたがまだディスクに書き戻されていないファイルシステムのキャッシュデータの量

#####  file_writeback

<!--
		Amount of cached filesystem data that was modified and
		is currently being written back to disk
-->
変更されて、ディスクに書き戻し中のファイルシステムのキャッシュデータの量

#####  inactive_anon, active_anon, inactive_file, active_file, unevictable

<!--
		Amount of memory, swap-backed and filesystem-backed,
		on the internal memory management lists used by the
		page reclaim algorithm
-->
ページ回収アルゴリズムが使う内部的なメモリ管理リスト上のメモリ量、Swap-backedの量、Filesystem-backed の量

          slab_reclaimable
                Part of "slab" that might be reclaimed, such as
                dentries and inodes.

          slab_unreclaimable
                Part of "slab" that cannot be reclaimed on memory
                pressure.

#####  pgfault

<!--
		Total number of page faults incurred
-->
ページフォルトの総数

#####  pgmajfault

<!--
		Number of major page faults incurred
-->
メジャーページフォルトの総数

#####  workingset_refault

<!--
		Number of refaults of previously evicted pages
-->

##### workingset_activate

<!--
                Number of refaulted pages that were immediately
                activated
-->

##### workingset_nodereclaim

                Number of times a shadow node has been reclaimed

##### pgrefill

                Amount of scanned pages (in an active LRU list)

##### pgscan

                Amount of scanned pages (in an inactive LRU list)

##### pgsteal

                Amount of reclaimed pages

##### pgactivate

                Amount of pages moved to the active LRU list

##### pgdeactivate

                Amount of pages moved to the inactive LRU lis

##### pglazyfree

                Amount of pages postponed to be freed under memory pressure

##### pglazyfreed

                Amount of reclaimed lazyfree pages

####  memory.swap.current

<!--
	A read-only single value file which exists on non-root
	cgroups.
-->
読み込み専用の単一の値を含むファイル。root 以外の cgroup に存在します。

<!--
	The total amount of swap currently being used by the cgroup
	and its descendants.
-->
cgroup と自分の子孫が現在使用中の swap の総量。

####  memory.swap.max

<!--
	A read-write single value file which exists on non-root
	cgroups.  The default is "max".
-->
読み書き可能な単一の値を含むファイル。root 以外の cgroup に存在します。デフォルト値は "max"。

<!--
	Swap usage hard limit.  If a cgroup's swap usage reaches this
	limit, anonymous meomry of the cgroup will not be swapped out.
-->
スワップ使用量のハードリミット。cgroup のスワップ使用量がこの制限に達した場合、cgroup の匿名メモリはスワップアウトしません。

<!-- Usage Guidelines -->
### 使用量のガイドライン

<!--
"memory.high" is the main mechanism to control memory usage.
Over-committing on high limit (sum of high limits > available memory)
and letting global memory pressure to distribute memory according to
usage is a viable strategy.
-->
"memory.high" は、メモリ使用量をコントロールするためのメインのメカニズムです。上限をオーバーコミット (上限の和 > 使用可能なメモリ) し、グローバルなメモリ圧力に使用量に応じてメモリを分配させることは実行可能な戦略です。

<!--
Because breach of the high limit doesn't trigger the OOM killer but
throttles the offending cgroup, a management agent has ample
opportunities to monitor and take appropriate actions such as granting
more memory or terminating the workload.
-->
上限を突き破ることは OOM killer のトリガーでなく、対象の cgroup の調整を行うことになるため、管理エージェントがモニタリングする機会をたっぷり持ち、より多くのメモリを与えると言った対応や、高負荷を終了させると言った対応のような、適切なアクションが取れます。

<!--
Determining whether a cgroup has enough memory is not trivial as
memory usage doesn't indicate whether the workload can benefit from
more memory.  For example, a workload which writes data received from
network to a file can use all available memory but can also operate as
performant with a small amount of memory.  A measure of memory
pressure - how much the workload is being impacted due to lack of
memory - is necessary to determine whether a workload needs more
memory; unfortunately, memory pressure monitoring mechanism isn't
implemented yet.
-->
作業がよりメモリを与えることで利益を得るかどうかは、メモリ使用量からはわからないように、cgroup が十分なメモリを持っているかどうかを判定するのは自明ではありません。例えば、ネットワークから受信したデータをファイルに書く作業はすべての使用可能なメモリを使う可能性がありますが、少ないメモリでの動作として操作もできます。メモリ圧力の測定、つまりメモリの不足が作業にどのくらいインパクトを与えるのかは、作業がより多くのメモリを必要としているかどうかを判定するのに必要です。不幸なことに、メモリ圧力をモニタリングする仕組みはまだ実装されていません。

<!-- Memory Ownership -->
### メモリの所有権

<!--
A memory area is charged to the cgroup which instantiated it and stays
charged to the cgroup until the area is released.  Migrating a process
to a different cgroup doesn't move the memory usages that it
instantiated while in the previous cgroup to the new cgroup.
-->
メモリ領域は、インスタンス化された cgroup にチャージされ、領域が開放されるまでその cgroup にチャージし続けられます。異なる cgroup へのプロセスのマイグレーションでは、移動前の cgroup 内でインスタンス化されているメモリ消費が新しい cgroup へ移動しません。

<!--
A memory area may be used by processes belonging to different cgroups.
To which cgroup the area will be charged is in-deterministic; however,
over time, the memory area is likely to end up in a cgroup which has
enough memory allowance to avoid high reclaim pressure.
-->
メモリ領域は、他の cgroup に属するプロセスが使う可能性があります。領域がどの cgroup にチャージされるかは常にひとつに決まります。しかし、時間の経過とともに、メモリ領域は高いメモリ回収圧力を避けることができる十分なメモリを持った cgroup 内に落ち着くでしょう。

<!--
If a cgroup sweeps a considerable amount of memory which is expected
to be accessed repeatedly by other cgroups, it may make sense to use
POSIX_FADV_DONTNEED to relinquish the ownership of memory areas
belonging to the affected files to ensure correct memory ownership.
-->
ある cgroup が、他の cgroup から定期的にアクセスされるであろう相当量のメモリを一掃した場合、正しいメモリ所有権を保証するために、影響を受けるファイルが属するメモリ領域を放棄するために POSIX_FADV_DONTNEED を使う意味があるかもしれません。

<!-- IO -->
IO
--

<!--
The "io" controller regulates the distribution of IO resources.  This
controller implements both weight based and absolute bandwidth or IOPS
limit distribution; however, weight based distribution is available
only if cfq-iosched is in use and neither scheme is available for
blk-mq devices.
-->
"io" コントローラは IO リソースの分配を調整します。このコントローラではウェイトベースと絶対値での帯域もしくは IOPS 制限での分配の両方が実装されています。しかし、ウェイトベースの分配は cfq-iosched が使用中の時のみ利用可能であり、両方の方法とも、blk-mq デバイスでは利用できません。


<!-- IO Interface Files -->
### 5-3-1. IO インターフェースファイル

####  io.stat

<!--
	A read-only nested-keyed file which exists on non-root
	cgroups.
-->
読み込み専用のネストされたキーのファイル。ルート以外の cgroup に存在します。

<!--
	Lines are keyed by $MAJ:$MIN device numbers and not ordered.
	The following nested keys are defined.
-->
行は $MAJ:$MIN というデバイス番号がキーになっており、順番には並んでいません。以下のネストしたキーが定義されています。

	  rbytes	Bytes read
	  wbytes	Bytes written
	  rios		Number of read IOs
	  wios		Number of write IOs

<!--
	An example read output follows.
-->
読み込んだ際の例は以下のようになります。

	  8:16 rbytes=1459200 wbytes=314773504 rios=192 wios=353
	  8:0 rbytes=90430464 wbytes=299008000 rios=8950 wios=1252

####  io.weight

<!--
	A read-write flat-keyed file which exists on non-root cgroups.
	The default is "default 100".
-->
読み書き可能なフラットキーなファイルです。ルート以外の cgroup に存在します。デフォルトは "default 100" です。

<!--
	The first line is the default weight applied to devices
	without specific override.  The rest are overrides keyed by
	$MAJ:$MIN device numbers and not ordered.  The weights are in
	the range [1, 10000] and specifies the relative amount IO time
	the cgroup can use in relation to its siblings.
-->
1 行目は特に指定しないデバイスに対して適用されるデフォルトのウェイトです。残りの行はデバイス番号 $MAJ:$MIN をキーに持つデバイスの値で、順番には並んでいません。ウェイトは [1, 10000] の範囲で、cgroup が他のデバイスと比較して使える IO 時間の相対的な量を定義します。

<!--
	The default weight can be updated by writing either "default
	$WEIGHT" or simply "$WEIGHT".  Overrides can be set by writing
	"$MAJ:$MIN $WEIGHT" and unset by writing "$MAJ:$MIN default".
-->
デフォルトウェイトは "default $WEIGHT" もしくは単に "$WEIGHT" を書き込んで更新できます。

<!--
	An example read output follows.
-->
ファイルを読み込んだ際の例は以下のようになります。

	  default 100
	  8:16 200
	  8:0 50

####  io.max

<!--
	A read-write nested-keyed file which exists on non-root
	cgroups.
-->
読み書き可能なネストしたキーのファイルです。ルート以外の cgroup に存在します。

<!--
	BPS and IOPS based IO limit.  Lines are keyed by $MAJ:$MIN
	device numbers and not ordered.  The following nested keys are
	defined.
-->
IO 制限をベースにした BPS と IOPS です。行はデバイス番号 $MAJ:$MIN がキーになっており、順番には並んでいません。以下のネストしたキーが定義されています。

	  rbps		Max read bytes per second
	  wbps		Max write bytes per second
	  riops		Max read IO operations per second
	  wiops		Max write IO operations per second

<!--
	When writing, any number of nested key-value pairs can be
	specified in any order.  "max" can be specified as the value
	to remove a specific limit.  If the same key is specified
	multiple times, the outcome is undefined.
-->
書き込みの際、任意の数のネストしたキー・値のペアを任意の順番で指定できます。"max" を指定した制限を削除するために指定できます。同じキーを複数回指定した場合の結果は不定です。

<!--
	BPS and IOPS are measured in each IO direction and IOs are
	delayed if limit is reached.  Temporary bursts are allowed.
-->
BPS と IOPS は IO の向きそれぞれで計測されます。制限に達した場合、IO は遅延します。一時的なバーストは許されます。

<!--
	Setting read limit at 2M BPS and write at 120 IOPS for 8:16.
-->
8:16 に対して、読み込みの制限を 2M BPS に、書き込みの制限を 120 IOPS に設定するには

	  echo "8:16 rbps=2097152 wiops=120" > io.max

<!--
	Reading returns the following.
-->
読み込むと以下のように出力されます。

	  8:16 rbps=2097152 wbps=max riops=max wiops=120

<!--
	Write IOPS limit can be removed by writing the following.
-->
書き込みの IOPS 制限を削除するには、以下のように書き込みます。

	  echo "8:16 wiops=max" > io.max

<!--
	Reading now returns the following.
-->
削除後、読み込むと以下のようになります。

	  8:16 rbps=2097152 wbps=max riops=max wiops=max


### Writeback

<!--
Page cache is dirtied through buffered writes and shared mmaps and
written asynchronously to the backing filesystem by the writeback
mechanism.  Writeback sits between the memory and IO domains and
regulates the proportion of dirty memory by balancing dirtying and
write IOs.
-->
ページキャッシュはバッファードライトとシェアード mmap を通して dirty になります。そしてライトバックのメカニズムによってファイルシステムに非同期に書き込まれます。ライトバックはメモリと IO の領域の間に存在します。そして、dirty とマークする処理と書き込み IO の間でバランスを取りながら、dirty メモリの割合を調節します。

<!--
The io controller, in conjunction with the memory controller,
implements control of page cache writeback IOs.  The memory controller
defines the memory domain that dirty memory ratio is calculated and
maintained for and the io controller defines the io domain which
writes out dirty pages for the memory domain.  Both system-wide and
per-cgroup dirty memory states are examined and the more restrictive
of the two is enforced.
-->
メモリコントローラと協力して、io コントローラはページキャッシュのライトバック IO のコントロールを実装しています。メモリコントローラは、dirty なメモリの割合を計算し、維持を行うメモリ領域を定義します。そして、io コントローラはメモリ領域から dirty なページを書き出す io 領域を定義します。

<!--
cgroup writeback requires explicit support from the underlying
filesystem.  Currently, cgroup writeback is implemented on ext2, ext4
and btrfs.  On other filesystems, all writeback IOs are attributed to
the root cgroup.
-->
cgroup のライトバックは、使用するファイルシステムのサポートが必ず必要です。現時点では、cgroup ライトバックは ext2、ext4、btrfs に実装されています。他のファイルシステムでは、すべてのライトバック IO は root cgroup に属します。

<!--
There are inherent differences in memory and writeback management
which affects how cgroup ownership is tracked.  Memory is tracked per
page while writeback per inode.  For the purpose of writeback, an
inode is assigned to a cgroup and all IO requests to write dirty pages
from the inode are attributed to that cgroup.
-->
どのように cgroup への所属がトラッキングされるかに影響する、メモリとライトバックの管理の固有の違いがあります。ライトバックは inode ごとですが、メモリはページごとにトラッキングされます。ライトバックの目的のために、inode は cgroup にアサインされます。inode から dirty ページを書き込むための IO リクエストは、その cgroup に所属します。

<!--
As cgroup ownership for memory is tracked per page, there can be pages
which are associated with different cgroups than the one the inode is
associated with.  These are called foreign pages.  The writeback
constantly keeps track of foreign pages and, if a particular foreign
cgroup becomes the majority over a certain period of time, switches
the ownership of the inode to that cgroup.
-->
メモリに対する cgroup の所有権はページごとにトラッキングされるので、inode が関連付けられている cgroup とは違う cgroup に関連付けられているページもありえます。これらは foreign ページと呼ばれます。ライトバックは常に foreign ページをトラックし続けます。そして、もし特定の外部 cgroup がある一定期間多数派になった場合、その cgroup に対する inode の所有権が変わるでしょう。

<!--
While this model is enough for most use cases where a given inode is
mostly dirtied by a single cgroup even when the main writing cgroup
changes over time, use cases where multiple cgroups write to a single
inode simultaneously are not supported well.  In such circumstances, a
significant portion of IOs are likely to be attributed incorrectly.
As memory controller assigns page ownership on the first use and
doesn't update it until the page is released, even if writeback
strictly follows page ownership, multiple cgroups dirtying overlapping
areas wouldn't work as expected.  It's recommended to avoid such usage
patterns.
-->
このモデルは与えられた inode がほとんど単一の cgroup によって dirty となるようなほとんどのユースケースだけでなく、メインの書き込み cgroup が時間とともに変化していく場合でも十分な一方、複数の cgroup が単一のinode にいっせいに書き込みにいくようなユースケースは十分にサポートできません。このような環境下では、IO の重要な部分が正しく所属できなさそうです。メモリコントローラは最初に使ったものにページの所有権を割り当て、ページが解放されるまで所有権を更新しないので、複数の cgroup の dirty がオーバラップするエリアでは、期待通り動作しないでしょう。このような使用パターンは避けることを推奨します。

<!--
The sysctl knobs which affect writeback behavior are applied to cgroup
writeback as follows.
-->
writeback に影響を与える sysctl 設定は cgroup の writeback に以下のように適用されます。

####  vm.dirty_background_ratio, vm.dirty_ratio

<!--
	These ratios apply the same to cgroup writeback with the
	amount of available memory capped by limits imposed by the
	memory controller and system-wide clean memory.
-->
これらの ratio は、メモリコントローラとシステムワイドのクリーンなメモリによって課せられる制限によって制限される利用可能なメモリ量の cgroup writeback に対して同じものが適用されます。

####  vm.dirty_background_bytes, vm.dirty_bytes

<!--
	For cgroup writeback, this is calculated into ratio against
	total available memory and applied the same way as
	vm.dirty[_background]_ratio.
-->
cgroup writeback の場合、これは利用可能なメモリの総量に対する割合に計算されます。そして、vm.dirty[_background]_ratio と同様の方法で適用されます。

PID
---

The process number controller is used to allow a cgroup to stop any
new tasks from being fork()'d or clone()'d after a specified limit is
reached.

The number of tasks in a cgroup can be exhausted in ways which other
controllers cannot prevent, thus warranting its own controller.  For
example, a fork bomb is likely to exhaust the number of tasks before
hitting memory restrictions.

Note that PIDs used in this controller refer to TIDs, process IDs as
used by the kernel.

PID Interface Files
~~~~~~~~~~~~~~~~~~~

  pids.max
        A read-write single value file which exists on non-root
        cgroups.  The default is "max".

        Hard limit of number of processes.

  pids.current
        A read-only single value file which exists on all cgroups.

        The number of processes currently in the cgroup and its
        descendants.

Organisational operations are not blocked by cgroup policies, so it is
possible to have pids.current > pids.max.  This can be done by either
setting the limit to be smaller than pids.current, or attaching enough
processes to the cgroup such that pids.current is larger than
pids.max.  However, it is not possible to violate a cgroup PID policy
through fork() or clone(). These will return -EAGAIN if the creation
of a new process would cause a cgroup policy to be violated.

RDMA
----

The "rdma" controller regulates the distribution and accounting of
of RDMA resources.

RDMA Interface Files
~~~~~~~~~~~~~~~~~~~~

  rdma.max
        A readwrite nested-keyed file that exists for all the cgroups
        except root that describes current configured resource limit
        for a RDMA/IB device.

        Lines are keyed by device name and are not ordered.
        Each line contains space separated resource name and its configured
        limit that can be distributed.

        The following nested keys are defined.

          ==========    =============================
          hca_handle    Maximum number of HCA Handles
          hca_object    Maximum number of HCA Objects
          ==========    =============================

        An example for mlx4 and ocrdma device follows::

          mlx4_0 hca_handle=2 hca_object=2000
          ocrdma1 hca_handle=3 hca_object=max

  rdma.current
        A read-only file that describes current resource usage.
        It exists for all the cgroup except root.

        An example for mlx4 and ocrdma device follows::

          mlx4_0 hca_handle=1 hca_object=20
          ocrdma1 hca_handle=1 hca_object=23

Misc
----

perf_event
~~~~~~~~~~

perf_event controller, if not mounted on a legacy hierarchy, is
automatically enabled on the v2 hierarchy so that perf events can
always be filtered by cgroup v2 path.  The controller can still be
moved to a legacy hierarchy after v2 hierarchy is populated.

Namespace
=========

Basics
------

cgroup namespace provides a mechanism to virtualize the view of the
"/proc/$PID/cgroup" file and cgroup mounts.  The CLONE_NEWCGROUP clone
flag can be used with clone(2) and unshare(2) to create a new cgroup
namespace.  The process running inside the cgroup namespace will have
its "/proc/$PID/cgroup" output restricted to cgroupns root.  The
cgroupns root is the cgroup of the process at the time of creation of
the cgroup namespace.

Without cgroup namespace, the "/proc/$PID/cgroup" file shows the
complete path of the cgroup of a process.  In a container setup where
a set of cgroups and namespaces are intended to isolate processes the
"/proc/$PID/cgroup" file may leak potential system level information
to the isolated processes.  For Example::

  # cat /proc/self/cgroup
  0::/batchjobs/container_id1

The path '/batchjobs/container_id1' can be considered as system-data
and undesirable to expose to the isolated processes.  cgroup namespace
can be used to restrict visibility of this path.  For example, before
creating a cgroup namespace, one would see::

  # ls -l /proc/self/ns/cgroup
  lrwxrwxrwx 1 root root 0 2014-07-15 10:37 /proc/self/ns/cgroup -> cgroup:[4026531835]
  # cat /proc/self/cgroup
  0::/batchjobs/container_id1

After unsharing a new namespace, the view changes::

  # ls -l /proc/self/ns/cgroup
  lrwxrwxrwx 1 root root 0 2014-07-15 10:35 /proc/self/ns/cgroup -> cgroup:[4026532183]
  # cat /proc/self/cgroup
  0::/

When some thread from a multi-threaded process unshares its cgroup
namespace, the new cgroupns gets applied to the entire process (all
the threads).  This is natural for the v2 hierarchy; however, for the
legacy hierarchies, this may be unexpected.

A cgroup namespace is alive as long as there are processes inside or
mounts pinning it.  When the last usage goes away, the cgroup
namespace is destroyed.  The cgroupns root and the actual cgroups
remain.

The Root and Views
------------------

The 'cgroupns root' for a cgroup namespace is the cgroup in which the
process calling unshare(2) is running.  For example, if a process in
/batchjobs/container_id1 cgroup calls unshare, cgroup
/batchjobs/container_id1 becomes the cgroupns root.  For the
init_cgroup_ns, this is the real root ('/') cgroup.

The cgroupns root cgroup does not change even if the namespace creator
process later moves to a different cgroup::

  # ~/unshare -c # unshare cgroupns in some cgroup
  # cat /proc/self/cgroup
  0::/
  # mkdir sub_cgrp_1
  # echo 0 > sub_cgrp_1/cgroup.procs
  # cat /proc/self/cgroup
  0::/sub_cgrp_1

Each process gets its namespace-specific view of "/proc/$PID/cgroup"

Processes running inside the cgroup namespace will be able to see
cgroup paths (in /proc/self/cgroup) only inside their root cgroup.
From within an unshared cgroupns::

  # sleep 100000 &
  [1] 7353
  # echo 7353 > sub_cgrp_1/cgroup.procs
  # cat /proc/7353/cgroup
  0::/sub_cgrp_1

From the initial cgroup namespace, the real cgroup path will be
visible::

  $ cat /proc/7353/cgroup
  0::/batchjobs/container_id1/sub_cgrp_1

From a sibling cgroup namespace (that is, a namespace rooted at a
different cgroup), the cgroup path relative to its own cgroup
namespace root will be shown.  For instance, if PID 7353's cgroup
namespace root is at '/batchjobs/container_id2', then it will see::

  # cat /proc/7353/cgroup
  0::/../container_id2/sub_cgrp_1

Note that the relative path always starts with '/' to indicate that
its relative to the cgroup namespace root of the caller.

Migration and setns(2)
----------------------

Processes inside a cgroup namespace can move into and out of the
namespace root if they have proper access to external cgroups.  For
example, from inside a namespace with cgroupns root at
/batchjobs/container_id1, and assuming that the global hierarchy is
still accessible inside cgroupns::

  # cat /proc/7353/cgroup
  0::/sub_cgrp_1
  # echo 7353 > batchjobs/container_id2/cgroup.procs
  # cat /proc/7353/cgroup
  0::/../container_id2

Note that this kind of setup is not encouraged.  A task inside cgroup
namespace should only be exposed to its own cgroupns hierarchy.

setns(2) to another cgroup namespace is allowed when:

(a) the process has CAP_SYS_ADMIN against its current user namespace
(b) the process has CAP_SYS_ADMIN against the target cgroup
    namespace's userns

No implicit cgroup changes happen with attaching to another cgroup
namespace.  It is expected that the someone moves the attaching
process under the target cgroup namespace root.

Interaction with Other Namespaces
---------------------------------

Namespace specific cgroup hierarchy can be mounted by a process
running inside a non-init cgroup namespace::

  # mount -t cgroup2 none $MOUNT_POINT

This will mount the unified cgroup hierarchy with cgroupns root as the
filesystem root.  The process needs CAP_SYS_ADMIN against its user and
mount namespaces.

The virtualization of /proc/self/cgroup file combined with restricting
the view of cgroup hierarchy by namespace-private cgroupfs mount
provides a properly isolated cgroup view inside the container.


Information on Kernel Programming

<!--
This section contains kernel programming information in the areas
where interacting with cgroup is necessary.  cgroup core and
controllers are not covered.
-->
このセクションは cgroup と情報をやりとりする必要のある分野のカーネルのプログラミング情報について記載します。cgroup core とコントローラについては含みません。

Information on Kernel Programming
=================================

This section contains kernel programming information in the areas
where interacting with cgroup is necessary.  cgroup core and
controllers are not covered.


Filesystem Support for Writeback
--------------------------------
<!--
A filesystem can support cgroup writeback by updating
address_space_operations->writepage[s]() to annotate bio's using the
following two functions.
-->
ファイルシステムは、以下の 2 つの関数を使って bio をアノテートするために `address_space_operations->writepage[s]()` を更新することで cgroup の writeback をサポートできます。

```
  wbc_init_bio(@wbc, @bio)
```

<!--
	Should be called for each bio carrying writeback data and
	associates the bio with the inode's owner cgroup.  Can be
	called anytime between bio allocation and submission.
-->
bio それぞれが writeback データを運ぶために呼び出されます。そして、inode のオーナーである cgroup と bio を結びつけます。bio のアロケーションとサブミッション (提出) の間いつでも呼び出せます。

```
  wbc_account_io(@wbc, @page, @bytes)
```

<!--
	Should be called for each data segment being written out.
	While this function doesn't care exactly when it's called
	during the writeback session, it's the easiest and most
	natural to call it as data segments are added to a bio.
-->
各データセグメントが書き出されるたびに呼び出します。この関数は writeback セッションのいつ呼び出されるか正確には気にしませんが、データセグメントが bio に追加されるときに呼び出すのがもっとも簡単で自然です。

<!--
With writeback bio's annotated, cgroup support can be enabled per
super_block by setting SB_I_CGROUPWB in ->s_iflags.  This allows for
selective disabling of cgroup writeback support which is helpful when
certain filesystem features, e.g. journaled data mode, are
incompatible.
-->
writeback bio がアノテートされると、`SB_I_CGROUPWB` が `-> s_iflags` に設定されることでスーパーブロックごとにcgroup サポートが有効になります。このことにより、cgroup writeback サポートの選択的に無効化できます。これは、ファイルシステムの特定の機能 (e.g. journaled data mode) に互換性がない場合に役立ちます。

<!--
wbc_init_bio() binds the specified bio to its cgroup.  Depending on
the configuration, the bio may be executed at a lower priority and if
the writeback session is holding shared resources, e.g. a journal
entry, may lead to priority inversion.  There is no one easy solution
for the problem.  Filesystems can try to work around specific problem
cases by skipping wbc_init_bio() or using bio_associate_blkcg()
directly.
-->
`wbc_init_bio()` は指定した bio をその cgroup に結びつけます。設定に依存して、bio は低い優先度で実行されるかもしれません。そして writeback セッションが共有リソースに縛り付けられた場合 (e.g. a journal entry)、優先度が逆転するかもしれません。この問題を解決する簡単な方法はありません。ファイルシステムは `wbc_init_bio()` をスキップするか、`bio_associate_blkcg()` を直接使用して、特有の問題のケースを回避しようとすることができます。

Deprecated v1 Core Features
===========================

<!--
- Multiple hierarchies including named ones are not supported.

- All mount options and remounting are not supported.

- The "tasks" file is removed and "cgroup.procs" is not sorted.

- "cgroup.clone_children" is removed.

- /proc/cgroups is meaningless for v2.  Use "cgroup.controllers" file at the root instead.
-->

- 名前付きの階層を含む複数階層構造はサポートされません 
- 一切のマウントオプションと再マウントはサポートされません
- "tasks" ファイルは削除され、"cgroup.procs" ファイルはソートされません
- "cgroup.clone_children" は削除されました
- /proc/cgroup は v2 では無意味です。root にある "cgroup.controllers" ファイルを代わりに使います

<!--
R. Issues with v1 and Rationales for v2
-->
v1 の問題と v2 の原理
==================

<!--
Multiple Hierarchies
-->
複数階層構造
----------

<!--
cgroup v1 allowed an arbitrary number of hierarchies and each
hierarchy could host any number of controllers.  While this seemed to
provide a high level of flexibility, it wasn't useful in practice.
-->
cgroup v1 では任意の数の階層構造が許されており、それぞれの階層構造は任意の数のコントローラを含められました。これは高いレベルでの柔軟性を提供しているように見えた一方で、実際には役に立ちませんでした。

<!--
For example, as there is only one instance of each controller, utility
type controllers such as freezer which can be useful in all
hierarchies could only be used in one.  The issue is exacerbated by
the fact that controllers couldn't be moved to another hierarchy once
hierarchies were populated.  Another issue was that all controllers
bound to a hierarchy were forced to have exactly the same view of the
hierarchy.  It wasn't possible to vary the granularity depending on
the specific controller.
-->
例えば、それぞれのコントローラはインスタンスがひとつだけですので、freezer のようなすべての階層で使えると便利なユーティリティタイプのコントローラがひとつでしか使えませんでした。この問題は、コントローラは一度有効化されると、他の階層に移動することができないので深刻です。他の問題は、ある階層にバインドされたコントローラはすべて同じ階層構造のビューを強制されるということでした。特定のコントローラに応じて粒度を変化させることができませんでした。

<!--
In practice, these issues heavily limited which controllers could be
put on the same hierarchy and most configurations resorted to putting
each controller on its own hierarchy.  Only closely related ones, such
as the cpu and cpuacct controllers, made sense to be put on the same
hierarchy.  This often meant that userland ended up managing multiple
similar hierarchies repeating the same steps on each hierarchy
whenever a hierarchy management operation was necessary.
-->
実際、これらの問題はどのコントローラを同じ階層に置くかを強く制限しましたし、結局はほとんどの場合、コントローラそれぞれを自身だけが属する階層に置くことになりました。cpu と cpuacct コントローラのように、密接に関係するコントローラだけが同じ階層に置く意味がありました。このことは、結局は階層を管理する操作が必要なときは、常にユーザランドは同じような階層を階層ごとに繰り返し同じような操作を行うことになることを意味しました。

<!--
Furthermore, support for multiple hierarchies came at a steep cost.
It greatly complicated cgroup core implementation but more importantly
the support for multiple hierarchies restricted how cgroup could be
used in general and what controllers was able to do.
-->
その上、複数階層構造のサポートは法外なコストにつながりました。cgroup コアの実装は複雑になりましたが、更に重要なのは、複数階層構造のサポートが一般的な cgroup の使われ方の可能性を制限し、コントローラができることを制限しました。

<!--
There was no limit on how many hierarchies there might be, which meant
that a thread's cgroup membership couldn't be described in finite
length.  The key might contain any number of entries and was unlimited
in length, which made it highly awkward to manipulate and led to
addition of controllers which existed only to identify membership,
which in turn exacerbated the original problem of proliferating number
of hierarchies.
-->
何階層存在する可能性があるのかの制限はありませんでした。これはスレッドの cgroup のメンバーシップが有限の長さで記述できないことを意味しました。ここでのキーは、任意の数のエントリが含まれ、長さに制限がないことで、これは操作をひどく扱いにくくし、メンバーシップを識別するためだけに存在するコントローラの追加につながり、巡り巡って急増する階層数という元の問題をより悪化させるものでした。

<!--
Also, as a controller couldn't have any expectation regarding the
topologies of hierarchies other controllers might be on, each
controller had to assume that all other controllers were attached to
completely orthogonal hierarchies.  This made it impossible, or at
least very cumbersome, for controllers to cooperate with each other.
-->
加えて、あるコントローラは他のコントローラがどのようなトポロジになるのかを全く予想できないので、他のコントローラがそれぞれ完全に直交する階層の上で操作されることを仮定しなければなりませんでした。コントローラがお互いに協調して動作するには、これは不可能であるか、少なくとも非常に負荷が大きくります。

<!--
In most use cases, putting controllers on hierarchies which are
completely orthogonal to each other isn't necessary.  What usually is
called for is the ability to have differing levels of granularity
depending on the specific controller.  In other words, hierarchy may
be collapsed from leaf towards root when viewed from specific
controllers.  For example, a given configuration might not care about
how memory is distributed beyond a certain level while still wanting
to control how CPU cycles are distributed.
-->
ユースケースのほとんどでは、お互い完全に直交している階層構造にコントローラを置く必要はありません。通常求められているものは、特定のコントローラに依存する異なる粒度のレベルを持つ能力です。言い換えると、階層は特定のコントローラの視点から見た時、末端からルートに向かって崩壊していくかもしれないということです。例えば、与えられた設定では、あるレベルを超えてメモリを配分するかについては設定されないかもしれないが、CPU サイクルの分配はコントロールしたいかもしれないというような事です。

<!--
R-2. Thread Granularity
-->
スレッドの粒度
-----------

<!--
cgroup v1 allowed threads of a process to belong to different cgroups.
This didn't make sense for some controllers and those controllers
ended up implementing different ways to ignore such situations but
much more importantly it blurred the line between API exposed to
individual applications and system management interface.
-->
cgroup v1 はプロセスのスレッドが異なる cgroup に属せました。コントローラによってはこれは意味がありませんでした。これらのコントローラでは色々な方法でこの状況を無視する方法を実装しましたが、さらに重要なことには、これはそれぞれのアプリケーションに提供される API とシステム管理インターフェースの間の境界を曖昧にしました。

<!--
Generally, in-process knowledge is available only to the process
itself; thus, unlike service-level organization of processes,
categorizing threads of a process requires active participation from
the application which owns the target process.
-->
一般的に、プロセス内の情報はプロセス自身のみが利用可能です。それゆえ、プロセスのサービスレベルの集合と違って、プロセスのスレッドをカテゴライズするには、ターゲットとなるプロセスを所有するアプリケーションからの積極的な関与が必要です。

<!--
cgroup v1 had an ambiguously defined delegation model which got abused
in combination with thread granularity.  cgroups were delegated to
individual applications so that they can create and manage their own
sub-hierarchies and control resource distributions along them.  This
effectively raised cgroup to the status of a syscall-like API exposed
to lay programs.
-->
cgroup v1 は、スレッドの粒度との組み合わせで濫用があった、曖昧に定義された委任モデルを持っていました。cgroup は、個別のアプリケーションが自身のサブ階層を作り管理し、その階層間でのリソース分配をコントロールできるように、個々のアプリケーションに権限委譲されていました。このことが、プログラムに対して提供されるシステムコール風の API のステータスへと cgroupを効果的に高めました。

<!--
First of all, cgroup has a fundamentally inadequate interface to be
exposed this way.  For a process to access its own knobs, it has to
extract the path on the target hierarchy from /proc/self/cgroup,
construct the path by appending the name of the knob to the path, open
and then read and/or write to it.  This is not only extremely clunky
and unusual but also inherently racy.  There is no conventional way to
define transaction across the required steps and nothing can guarantee
that the process would actually be operating on its own sub-hierarchy.
-->
まず第一に、基本的には cgroup はこのように提供される不適切なインターフェースを持っています。プロセスが自身の調整ノブへアクセスするために、/proc/self/cgroup から得られるターゲットのパスを展開し、調整ノブの名前をパスに追加してパス名を構築し、オープンし、それからそれに対して読み書きをする必要があります。これはとても格好悪く、一般的ではないだけでなく、本質的に下品です。必要とするステップにわたってトランザクションを定義するための従来のような方法がなく、プロセスが実際に自身のサブ階層上で操作されることを保証するものは何もありません。

<!--
cgroup controllers implemented a number of knobs which would never be
accepted as public APIs because they were just adding control knobs to
system-management pseudo filesystem.  cgroup ended up with interface
knobs which were not properly abstracted or refined and directly
revealed kernel internal details.  These knobs got exposed to
individual applications through the ill-defined delegation mechanism
effectively abusing cgroup as a shortcut to implementing public APIs
without going through the required scrutiny.
-->
cgroup コントローラは公開 API としては決して受け入れられない多数の調整ノブを実装しました。なぜなら、それらは単にシステム管理の擬似ファイルシステムに対する調整ノブを追加するだけだったからです。cgroup は、適切に抽象化されたり改良されない、カーネルの内部の詳細を直接露呈するものになってしまいました。これらの調整ノブは、必要な精査を受けることなく、cgroup を公開API を実装するショートカットとして乱用する、不適切に定義された委任メカニズムを通して個々のアプリケーションに晒されます。

<!--
This was painful for both userland and kernel.  Userland ended up with
misbehaving and poorly abstracted interfaces and kernel exposing and
locked into constructs inadvertently.
-->
これはユーザランドとカーネル両方にとってつらいものでした。ユーザランドは誤った振る舞いをするようになり、貧弱に抽象化されたインターフェースとカーネルは不注意に構築されたものにロックされ、さらされています。


<!--
R-3. Competition Between Inner Nodes and Threads
-->
Competition Between Inner Nodes and Threads
-------------------------------------------

<!--
cgroup v1 allowed threads to be in any cgroups which created an
interesting problem where threads belonging to a parent cgroup and its
children cgroups competed for resources.  This was nasty as two
different types of entities competed and there was no obvious way to
settle it.  Different controllers did different things.
-->
cgroup v1 は任意の cgroup にスレッドが所属できました。これは親の cgroup とその子の cgroup に属するスレッドがリソースを争うところで興味深い問題を作り出しました。ふたつの異なるタイプのエンティティが競合し、それを解決するための明確な方法がなかったので、これは厄介な問題でした。コントローラが異なると違うことをやっていました。

<!--
The cpu controller considered threads and cgroups as equivalents and
mapped nice levels to cgroup weights.  This worked for some cases but
fell flat when children wanted to be allocated specific ratios of CPU
cycles and the number of internal threads fluctuated - the ratios
constantly changed as the number of competing entities fluctuated.
There also were other issues.  The mapping from nice level to weight
wasn't obvious or universal, and there were various other knobs which
simply weren't available for threads.
-->
CPU コントローラはスレッドと cgroup を等価に扱い、nice レベルを cgroup ウェイトにマッピングしました。これで働くケースもありましたが、子どもが CPU サイクルの特定の割合の割り当てを要求し、内部スレッドの数が上下するとき、つまり競合するエンティティの数が上下するので割合が常に変化する場合にはうまくいきませんでした。

<!--
The io controller implicitly created a hidden leaf node for each
cgroup to host the threads.  The hidden leaf had its own copies of all
the knobs with "leaf_" prefixed.  While this allowed equivalent
control over internal threads, it was with serious drawbacks.  It
always added an extra layer of nesting which wouldn't be necessary
otherwise, made the interface messy and significantly complicated the
implementation.
-->
IO コントローラは、スレッドを扱うために cgroup それぞれに隠れたリーフノードを暗黙のうちに作成します。この隠れたリーフは自身のコピーとして、頭に "leaf_" と付いた全てのノブを持ちます。これは同等のコントロールが内部タスクにも可能になりますが、重大な欠点も持ちます。そうでなければ常に必要ではないかもしれないネストした余分なレイヤーを追加し、インターフェースを乱雑にして、実装をかなり複雑にします。。

<!--
The memory controller didn't have a way to control what happened
between internal tasks and child cgroups and the behavior was not
clearly defined.  There were attempts to add ad-hoc behaviors and
knobs to tailor the behavior to specific workloads which would have
led to problems extremely difficult to resolve in the long term.
-->
メモリコントローラは内部タスクと子cgroup間で起こっていることをコントロールする方法がありませんでした。そして、振る舞いが明確には定義されていませんでした。。この方向性を続けることは長期間に渡って解決が非常に難しい問題を引き起こすであろう、特定の作業に振る舞いを合わせるためのアドホックな振る舞いとノブが追加される試みがありました。

<!--
Multiple controllers struggled with internal tasks and came up with
different ways to deal with it; unfortunately, all the approaches were
severely flawed and, furthermore, the widely different behaviors
made cgroup as a whole highly inconsistent.
-->
複数のコントローラが内部タスクと奮闘していました。そして、それを解決する色々な方法を考えだしてきました。不幸にも、全てのアプローチは重大な欠点がありました。その上、非常に異なった振る舞いが cgroup 全体を完全に一貫性のないものにしている。

<!--
This clearly is a problem which needs to be addressed from cgroup core
in a uniform way.
-->
これはあきらかに統一された方法で cgroup コアから対処する必要がある問題です。

<!--
R-4. Other Interface Issues
-->
Ohter Interface Issues
----------------------

<!--
cgroup v1 grew without oversight and developed a large number of
idiosyncrasies and inconsistencies.  One issue on the cgroup core side
was how an empty cgroup was notified - a userland helper binary was
forked and executed for each event.  The event delivery wasn't
recursive or delegatable.  The limitations of the mechanism also led
to in-kernel event delivery filtering mechanism further complicating
the interface.
-->
cgroup v1 は管理がないまま進化し、多数の独自性と矛盾が明らかになりました。cgroup コア側にある問題のひとつは、空の cgroup を通知する方法でした。それは、ユーザーランドのヘルパーバイナリが起動され、イベントごとに実行されました。このイベントは再帰的でも委任可能でもありませんでした。メカニズムの制限は、カーネル内のイベント配送のフィルタリングメカニズムをより複雑なインターフェースにした。

<!--
Controller interfaces were problematic too.  An extreme example is
controllers completely ignoring hierarchical organization and treating
all cgroups as if they were all located directly under the root
cgroup.  Some controllers exposed a large amount of inconsistent
implementation details to userland.
-->
コントローラインターフェースにも問題がありました。極端な例は、階層構造を完全に無視し、すべての cgroup が root cgroup 直下にあるように扱うコントローラです。大量の一貫性のない実装を大量にユーザランドにさらしているコントローラもありました。

<!--
There also was no consistency across controllers.  When a new cgroup
was created, some controllers defaulted to not imposing extra
restrictions while others disallowed any resource usage until
explicitly configured.  Configuration knobs for the same type of
control used widely differing naming schemes and formats.  Statistics
and information knobs were named arbitrarily and used different
formats and units even in the same controller.
-->
コントローラ間の一貫性のなさもありました。新しい cgroup が作られたとき、特に制限がかからないコントローラもあれば、明示的に設定されるまではリソースが使えないコントローラもありました。同じタイプのコントロールをするための設定ノブは、異なるネーミングスキームとフォーマットで広く使われています。統計や情報を表示するノブは任意の名前が付けられ、同じコントローラ内でさえ異なるフォーマットと単位が用いられたりしました。

<!--
cgroup v2 establishes common conventions where appropriate and updates
controllers so that they expose minimal and consistent interfaces.
-->
cgroup v2 では必要に応じて共通の規約が確立され、最小限の一貫性あるインターフェースが提供されるようにコントローラが更新されます。

<!-- R-5. Controller Issues and Remedies -->
コントローラの問題と改善
-------------------

<!-- R-5-1. Memory -->
### R-5-1. メモリ

<!--
The original lower boundary, the soft limit, is defined as a limit
that is per default unset.  As a result, the set of cgroups that
global reclaim prefers is opt-in, rather than opt-out.  The costs for
optimizing these mostly negative lookups are so high that the
implementation, despite its enormous size, does not even provide the
basic desirable behavior.  First off, the soft limit has no
hierarchical meaning.  All configured groups are organized in a global
rbtree and treated like equal peers, regardless where they are located
in the hierarchy.  This makes subtree delegation impossible.  Second,
the soft limit reclaim pass is so aggressive that it not just
introduces high allocation latencies into the system, but also impacts
system performance due to overreclaim, to the point where the feature
becomes self-defeating.
-->
以前の機能での下限である、ソフトリミットは、デフォルトでは設定されていない制限として定義されます。その結果、グローバルに回収される cgroup のセットは、オプトアウトでなく、オプトインになります。これらのほとんどがネガティブになるルックアップの最適化のコストはとても高いので、その実装は、巨大なサイズにも関わらず、基本的な望ましい動作を提供するものではありません。まず、ソフトリミットには階層的な意味はありません。すべての設定されたグループがグローバルな rbtree に構造化され、階層に位置するにも関わらず、等価なピアのように扱われます。このため、サブツリーの委任ができません。つぎに、ソフトリミットの回収パスはとても変化が激しいので、高い割り当てレイテンシをシステムにはもたらすだけでなく、過度な回収によるシステムパフォーマンスへの影響も引き起こし、将来的には自滅するに至るでしょう。

<!--
The memory.low boundary on the other hand is a top-down allocated
reserve.  A cgroup enjoys reclaim protection when it and all its
ancestors are below their low boundaries, which makes delegation of
subtrees possible.  Secondly, new cgroups have no reserve per default
and in the common case most cgroups are eligible for the preferred
reclaim pass.  This allows the new low boundary to be efficiently
implemented with just a minor addition to the generic reclaim code,
without the need for out-of-band data structures and reclaim passes.
Because the generic reclaim code considers all cgroups except for the
ones running low in the preferred first reclaim pass, overreclaim of
individual groups is eliminated as well, resulting in much better
overall workload performance.
-->
一方、制限値 memory.low はトップダウンで割り当てられる予約です。cgroup は、サブツリーへの委譲が可能な、自身と祖先が low 限界値を下回っている場合は、回収の保護を楽しめます。第 2 に、新しい cgroup はデフォルトでの予約はなく、一般的なケースでは、ほとんどの cgroup が優先回収パスに対する資格があります。これにより、帯域外のデータ構造と回収パスを必要とせずに、ジェネリックは回収コードへ少し追加を行うだけで、新しい low 制限値が効率的に実装できます。ジェネリックな回収コードは、優先する最初の回収パス内の low で実行しているものを除いて、すべての cgroup を考慮するので、個々のグループの過度な回収をなくすことができ、さらに全体のワークロードパフォーマンスを向上させます。

<!--
The original high boundary, the hard limit, is defined as a strict
limit that can not budge, even if the OOM killer has to be called.
But this generally goes against the goal of making the most out of the
available memory.  The memory consumption of workloads varies during
runtime, and that requires users to overcommit.  But doing that with a
strict upper limit requires either a fairly accurate prediction of the
working set size or adding slack to the limit.  Since working set size
estimation is hard and error prone, and getting it wrong results in
OOM kills, most users tend to err on the side of a looser limit and
end up wasting precious resources.
-->
元の上限であるハードリミットは、OOM Killer を呼ぶ必要がある場合ですら、変更できない厳密な制限として定義されています。しかし、これは一般的には利用可能なメモリを最大限利用するというゴールに反します。負荷のメモリ消費は実行中に変化し、ユーザはオーバーコミットする必要があります。しかし、厳格な制限値でこれを行うには、ワーキングセットのサイズをかなり正確に予測するか、制限に余裕を加える必要があります。ワーキングセットの見積もりは難しく、エラーが発生しやすく、OOK Killer で間違った結果になるため、ほとんどのユーザは制限を緩めてエラーををお越し、貴重なリソースを無駄にする傾向があります。

<!--
The memory.high boundary on the other hand can be set much more
conservatively.  When hit, it throttles allocations by forcing them
into direct reclaim to work off the excess, but it never invokes the
OOM killer.  As a result, a high boundary that is chosen too
aggressively will not terminate the processes, but instead it will
lead to gradual performance degradation.  The user can monitor this
and make corrections until the minimal memory footprint that still
gives acceptable performance is found.
-->
一方、制限値 memory.high は、より控えめに設定できます。この値にヒットしたときには、超過分を減らすために強制的に direct reclaim (メモリ要求時に同期してページ回収を実行する) を呼び出すことにより、割り当てを絞ります。しかし、OOM Killer が呼び出されることはありません。結果として、頻繁にヒットする制限値はプロセスを終了させず、代わりに徐々にパフォーマンスの劣化を引き起こします。ユーザはこれをモニタリングし、受け入れできるパフォーマンスが得られる最小のメモリフットプリントが見つかるまで調整が行えます。

In extreme cases, with many concurrent allocations and a complete
breakdown of reclaim progress within the group, the high boundary can
be exceeded.  But even then it's mostly better to satisfy the
allocation from the slack available in other groups or the rest of the
system than killing the group.  Otherwise, memory.max is there to
limit this type of spillover and ultimately contain buggy or even
malicious applications.

The combined memory+swap accounting and limiting is replaced by real
control over swap space.

The main argument for a combined memory+swap facility in the original
cgroup design was that global or parental pressure would always be
able to swap all anonymous memory of a child group, regardless of the
child's own (possibly untrusted) configuration.  However, untrusted
groups can sabotage swapping by other means - such as referencing its
anonymous memory in a tight loop - and an admin can not assume full
swappability when overcommitting untrusted jobs.

For trusted jobs, on the other hand, a combined counter is not an
intuitive userspace interface, and it flies in the face of the idea
that cgroup controllers should account and limit specific physical
resources.  Swap space is a resource like all others in the system,
and that's why unified hierarchy allows distributing it separately.
