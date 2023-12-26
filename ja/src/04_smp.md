# SMP and CPU specific configuration

このセクションではVMMのSMP（Symmetric MultiProcessing）対応について主題として議論する
副題としてIntel/AMD特有の設定についても触れることとする

これまで仮想マシンに割り当てることのできるvCPUの数は1つだった。
SMPの対応を行うことで、仮想マシンに複数のCPUを割り当てて、複数プロセッサでの並行処理に対応させたい。
この詳細については [04-1. SMP - Symmetric MultuProcessing](./04-1_smp_symmetric_multiprocessing.md)に詳細を記載することとする。

また、SMPの実装に合わせてコードを変更するついでに、ベンダーの異なる各CPU（Intel・AMD）に対して独自の設定ができるような実装にリファインする。
共通の設定については [04-2. Common CPU configuration](./04-2_common_CPU_configuration.md)、Intel CPUに関する設定については [04-3. Intel CPU specific configuration](./04-3_Intel_CPU_specific_configuration.md)、AMD CPUに関する設定については [04-4. AMD CPU specific configuration](./04-4_AMD_CPU_specific_configuration.md)で議論することとするが、これらのCPU設定については必要に応じて設定をする類のものが多いこともあり、ベストエフォートで記載を進めることさせてもらう。
そのため、およびについては不定期に更新が入るものと読者には想定してもらいたい。

改めて、本セクションの各トピックは次のようになっている

* [04-1. SMP - Symmetric MultuProcessing](./04-1_smp_symmetric_multiprocessing.md)
* [04-2. Common CPU configuration](./04-2_common_CPU_configuration.md)
* [04-3. Intel CPU specific configuration](./04-3_Intel_CPU_specific_configuration.md)
* [04-4. AMD CPU specific configuration](./04-4_AMD_CPU_specific_configuration.md)

本資料では、上述したとおり不定期に更新が入る想定にしているため、ベースとしているコミットナンバーについては各トピックの話題毎にそれぞれ記載することとする。
