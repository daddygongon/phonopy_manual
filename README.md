# Phonopy_manual
私の環境はPython 2.7.12，Phonopy 1.11.2です．

## Phonopyのインストール
Pythonのパッケージ管理ツールpip(rubyでいうところのgem)で簡単にinstallできる．
最近のPythonはpipをデフォルトで備えているみたいです．
pipが入ってない場合はeasy_install(Pythonが入っていれば使えるはず）を使ってpipを入れる．
```
sudo easy_install pip
```
pipが入れば
```
sudo pip install phonopy
```
でインストールできます．numpy等のライブラリが入っておらずエラーが出た場合は，
```
sudo pip install numpy
```
と入っていないライブラリをインストールする必要があるかもしれません．
その場合もエラーメッセージに足りていないライブラリが書いてあるはずです．
```
/Users/sakaki% phonopy
        _
  _ __ | |__   ___  _ __   ___   _ __  _   _
 | '_ \| '_ \ / _ \| '_ \ / _ \ | '_ \| | | |
 | |_) | | | | (_) | | | | (_) || |_) | |_| |
 | .__/|_| |_|\___/|_| |_|\___(_) .__/ \__, |
 |_|                            |_|    |___/
                                      1.11.2


Supercell matrix (DIM or --dim) is not found.
                 _
   ___ _ __   __| |
  / _ \ '_ \ / _` |
 |  __/ | | | (_| |
  \___|_| |_|\__,_|

```
となればインストール完了．

##　使用法
基本的な使用法はphonopyのHPの[example](https://atztogo.github.io/phonopy/examples.html)で確認できます．
ページの指示に従いphonopyのソースファイルを取得し，その中のexapleディレクトリ内にそれぞれの計算例に必要なデータが入っています．

###グラフが表示されない場合
プロットの際に次のようなエラーが表示される場合
```
RuntimeError: Python is not installed as a framework. The Mac OS X backend will not be able to function correctly if Python is not installed as a framework
```
画像表示のinterfaceが指定されていないから起こるエラーらしい．．
~/.matplotlib/matplotlibrc
というファイルを作成し
```
backend : TkAgg
```
と記述すればグラフが表示されるようになる．

## 計算の全体の流れ
### Phonon計算
* PhonopyにPOSCARを読み込ませ，原子を微小移動し拡張させたモデルPOSCAR-XXXを生成．
* POSCAR-XXXをVASPで計算.
* VASPの計算結果であるvasprun.xmlから力の定数FORCE_SETSを生成．
* FORCE_SETSからPhononや熱的性質が計算できる．

### 熱膨張
* 格子の長さを変化させ上記のPhonon計算を行い，-tオプションによりthermal_properties.yamlを生成．
* それぞれの格子の長さのthermal_properties.yamlと，体積と基底状態のエネルギーのデータから，熱膨張を計算できる．

## 実際の計算
###拡張したモデルの生成
POSCARのあるディレクトリで
```
phonopy -d --dim="2 2 2"
```
と入力．この場合は2-2-2に拡張される．以下のように4つファイルが生成される
```
/Users/sakaki/Research/phonopy_manual/test% tree
.
├── POSCAR
├── POSCAR-001          POSCARを拡張し，微小移動されたモデル．これをVASPに投げる．
├── SPOSCAR             SUPERCELL POSCAR? POSCARを拡張しただけのモデル
├── disp.yaml           生成したモデルの情報
└── phonopy_disp.yaml   生成したモデルの情報-Phonopyのversion等も書いている．
```
今回はPOSCARに以下のようにfcc構造のCuを使ったため拡張したモデルが１つだけ生成されている．
hcp構造などの等方的でない構造だと拡張したモデルが増える．
```
Cu                                      
   1.00000000000000     
     3.6364158117330518    0.0000000000000000    0.0000000000000000
     0.0000000000000000    3.6364158117330518    0.0000000000000000
     0.0000000000000000    0.0000000000000000    3.6364158117330518
   4
Direct
  0.0000000000000000  0.0000000000000000  0.0000000000000000
  0.5000000000000000  0.5000000000000000  0.0000000000000000
  0.5000000000000000  0.0000000000000000  0.5000000000000000
  0.0000000000000000  0.5000000000000000  0.5000000000000000
```
###VASPでの計算
先ほど生成したPOSCAR-001をPOSCARとしてVASPで計算を行う．
修士論文の中で計算に使用したINCARとKPOINTSは次のようになる．
INCAR
```
PREC = Accurate
IBRION = -1
ENCUT = 750
EDIFF = 1.0e-08
ISMEAR = 0; SIGMA = 0.01
IALGO = 38
LREAL = .FALSE.
LWAVE = .FALSE.
LCHARG = .FALSE.
ADDGRID = .TRUE.
```
KPOINTS
```
Automatic mesh
0
Gamma
12 12 12
0. 0. 0.
```

###FORCE_SETSの生成．
VASPの計算結果からvasprun.xmlを持ってくる．
```
/Users/sakaki/Research/phonopy_manual/test% ls
POSCAR            POSCAR-001        SPOSCAR           disp.yaml         phonopy_disp.yaml vasprun.xml
```
ここで
```
phonopy -f vasprun.xml
```
によりFORCE_SETSが生成される．

### Phonon計算
Phonon計算の条件を記すファイルmesh.confを用意する．修論では次のものを使用した．
mesh.conf
```
ATOM_NAME = Cu
DIM = 2 2 2
MP = 48 48 48
TMAX = 2000
```
ATOM_NAMEは元素名，DIMはPOSCARを拡張した倍率，MPはPhonon計算に使用するmesh points，TMAXは計算する温度の最大値(デフォルトは1000)．

#### Phonon-DOS
```
phonopy -p mesh.conf
```
．
#### 自由エネルギー
-t(thermal)オプションで，
```
phonopy -t mesh.conf
```
-p(plot)オプションを追加することでグラフを生成できる．
```
phonopy -t -p mesh.conf
```
また-tオプションを実行すると各温度の自由エネルギー等の情報の入ったthermal_properties.yamlが生成される．
このthermal_properties.yamlは熱膨張の計算の際に使用する．

###熱膨張
それぞれの格子の長さでthermal_properties.yamlを生成し数字をつけて区別できるようにする．
また，別途にVASPでそれぞれの体積でエネルギー計算を行い次のようなe-vファイルを用意しておく．**注意，phonopyの計算はkJ/molだがe-vファイルのエネルギーの単位はeVを用いる．**
```
41.24181308799384 -14.302246
42.55794342524488 -14.522973
43.901780756913375 -14.684095
45.27361369752454 -14.792187
46.67373086160355 -14.853231
48.102420863675576 -14.872477
49.55997231826581 -14.85458
51.04667383989944 -14.803835
52.56281404310163 -14.724137
54.10868154239757 -14.61897
55.684564952312456 -14.491449
```

####熱膨張の計算
次のコマンドで熱膨張を計算することができる．
```
phonopy-qha -p e-v.dat thermal_properties-{95..105}.yaml
```
このコマンドは
```
phonopy-qha -p e-v.dat thermal_properties-95.yaml thermal_properties-96.yaml thermal_properties-97.yaml thermal_properties-98.yaml thermal_properties-99.yaml thermal_properties-100.yaml thermal_properties-101.yaml thermal_properties-102.yaml thermal_properties-103.yaml thermal_properties-104.yaml thermal_properties-105.yaml
```
と同義でありthermal_properties-95.yamlは格子の長さを0.95倍したモデルである．
このようにe-v.datとそれぞれのthermal_properties.yamlを読み込ますことで熱膨張を算出することができる．
読み込ませたe-v.datと格子の長さによる自由エネルギーを足し合わせ，各温度でフィッティングを行い最安定の点をその温度での体積としている．

####出力されるデータについて
熱膨張の計算を行うと次のようなデータファイルが生成される．
今回の論文に使用したものに関してコメントをつけておきます．
```
├── Cp-temperature.dat
├── Cp-temperature_polyfit.dat
├── Cv-volume.dat
├── bulk_modulus-temperature.dat
├── dsdv-temperature.dat
├── entropy-volume.dat
├── gibbs-temperature.dat フィッティングによって得られた各温度の体積．修論の内部エネルギーを考慮した自由エネルギーはこのデータを使用している．
├── gruneisen-temperature.dat
├── helmholtz-volume.dat フィッティングに使用した各温度のエネルギーの値が入ってる．
├── thermal_expansion.dat 体積膨張係数，修論では3で割り線膨張係数に変換して利用している．
├── volume-temperature.dat
└── volume_expansion.dat
```
詳しくはphonopyのHPの[Quasi harmonic approximation](https://atztogo.github.io/phonopy/qha.html)に書いてあるので参考にしてください．
## 修論に使用したphonopyの計算データ
calcディレクトリ内にそれぞれの元素の計算データが入っています．
基本の構造は次のようになります．
```
├── Cu
│   ├── 100
│   ├── 101
│   ├── 102
│   ├── 103
│   ├── 104
│   ├── 105   それぞれ，0.95-1.05の格子の長さでPhonopyによりモデル生成，計算．
│   ├── 95
│   ├── 96
│   ├── 97
│   ├── 98
│   ├── 99
│   ├── TE　　熱膨張の計算．
│   └── vasp　vaspの計算結果．元素によって名前が変わってるかもしれません，，，AgだとAgという名前になっています．
```
Alは修論にある通り計算精度の比較をおこなったためディレクトリが複数ありますが，
Al_ENCUTディレクトリが修論に使用したデータとなっています．
