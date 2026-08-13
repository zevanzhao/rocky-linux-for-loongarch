# 介绍

Rocky Linux作为Cent OS操作系统的后续，是一个很受欢迎的、非滚动更新的发行版。遗憾的是，这个发行版官方尚未支持龙架构。同时，目前龙架构下可用的操作系统虽然很多，如Gentoo, Debian, ArchLinux等，但仍然缺少非滚动更新、稳定的操作系统发行版。

为了弥补这个不足，同时也是为了学习孙海勇老师的《用“芯”探索：教你构建龙芯平台的Linux操作系统》一书，我决定进行Rocky Linux 10的龙架构移植尝试。

本仓库记录了参照孙海勇老师的教程，在龙架构(Loongarch64)上，移植Rocky Linux 10 的整个流程，包括构建中遇到的问题，以及解决问题的方案。有一些问题在编译中未能解决，也一并记录下来。希望此笔记能够对其他人有用。阅读此笔记的时候，建议参考孙海勇老师的《用“芯”探索》一书。

由于这是个人笔记，记录仓促，存在很多的笔误; Markdown格式的笔记是由docx文档转换而来，格式正在逐步修改，还请读者酌情使用。

# 笔记内容

* [龙架构上Rocky Linux 10 操作系统移植笔记](Rocky_linux_10_for_loongarch64_porting_notes.md)
* [脚本](scripts/)
* [源码](sources)

# 致谢

在操作系统的移植过程中，我遇到了很多问题，得到了[孙海勇](https://github.com/sunhaiyong1978/CLFS-for-LoongArch/)老师的很多指导，也参考了孙海勇老师的[龙架构下CLFS系统构建](https://github.com/sunhaiyong1978/CLFS-for-LoongArch/) 教程。

弟子在此向孙老师表示感谢！