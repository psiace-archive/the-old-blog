+++
title = "在 Fedora 上为 Btrfs 新建 32GiB Swapfile"
slug = "btrfs-32gib-swapfile-on-fedora"
date = 2022-01-23
drafts = false
[taxonomies]
tags = ["2022", "Btrfs", "Swapfile", "Fedora", "Linux"]
+++

受到 [Databend - set swap to 10G](https://github.com/datafuselabs/databend/pull/3946) 的感召，检查了一下自己本子的 Swap ，只有大概 8G 的 zram 。

```shell
[psiace@fedora ~]$ swapon -s
Filename				Type		  Size		  Used	Priority
/dev/zram0      partition	8388604		0	    100
```

完蛋，不能顺利跑完 grcov 限定版 unit-test 的原因大概就在这里了（之前有讨论过是 OOM）。本着「大就是好，多就是美」的原则，决定给它来个超级加倍，再塞个 32 GiB 的 Swapfile 上去。

## Btrfs 限定之初始化 Swapfile

自 5.0 内核之后，Btrfs 才支持创建 Swapfile ，而且有一些特别的要求：

- Swapfile 不能放在 snapshotted subvolume （快照子卷）上。
- 不支持跨多设备文件系统上的 Swapfile 。

所以正确的做法是：新建一个 non-snapshotted subvolume ，然后在该子卷之下创建禁用压缩的 Swapfile 。

```shell
# 创建 non-snapshotted subvolume 。
[psiace@fedora /]$ sudo btrfs subvolume create swap
Create subvolume './swap'
# 进入子卷。
[psiace@fedora /]$ cd swap
# 新建长度为 0 的 Swapfile 。
[psiace@fedora swap]$ sudo truncate -s 0 ./swapfile
# 设置交换文件的属性，使其免于 copy-on-write 。
[psiace@fedora swap]$ sudo chattr +C ./swapfile
# 禁用压缩。
[psiace@fedora swap]$ sudo btrfs property set ./swapfile compression none
```

注意，这些需要在系统根目录下完成，以避免权限问题和设置问题。

## 设定 Swapfile 作为 Swap 成分之一

Swapfile 是创建特定交换分区的一种替代方案，好处是方便创建和删除、也便于动态变更大小。

这种方案比较适合 SSD 空间充裕的情况。刚好可以组成一个 memory -> zram -> swapfile 的多级交换。

```shell
# 将 Swapfile 填充至合适的大小，一般选择内存空间的一半或者与内存空间相当。
# 这里仅仅是为了好玩，选择了巨量的 32GiB ，有浪费之嫌。
[psiace@fedora swap]$ sudo dd if=/dev/zero of=./swapfile bs=1M count=32768 status=progress
33980153856字节（34 GB，32 GiB）已复制，20 s，1.7 GB/s
记录了32768+0 的读入
记录了32768+0 的写出
34359738368字节（34 GB，32 GiB）已复制，21.2624 s，1.6 GB/s
# 设置正确的权限。
[psiace@fedora swap]$ sudo chmod 600 ./swapfile
# 格式化 Swapfile 作为交换类型。
[psiace@fedora swap]$ sudo mkswap ./swapfile
正在设置交换空间版本 1，大小 = 32 GiB (34359734272  个字节)
无标签，UUID=2e48f371-62a9-487a-9613-382b386b2836
# 激活交换文件，并设定优先级。
# 由于 zram 的优先级是 100 ，所以这里设定成 50 。毕竟 zram 的性能比 swapfile 要强不少。
[psiace@fedora swap]$ sudo swapon --priority 50 ./swapfile
```

## 检查 Swap 空间并设置自动挂载

那么，经过之前两步，已经得到了接近 40GiB 的 Swap 空间，接下来就是检查一下，并设置挂载。

```
# 使用 free 查看概览。
[psiace@fedora ~]$ free -m
               total        used        free      shared  buff/cache   available
Mem:           15453        5206         290         109        9956        9808
Swap:          40959           2       40957
# 使用 swapon 查看详情。
[psiace@fedora ~]$ swapon -s
Filename				Type		  Size		  Used		Priority
/dev/zram0      partition	8388604		2560		100
/swap/swapfile  file		  33554428	0		    50
# 编辑 fstab ，添加指定条目以完成挂载。
# 这里必须带上子卷名，UUID 可以不写。
[psiace@fedora ~]$ sudo nano /etc/fstab
/swap/swapfile    none    swap    defaults    0    0
```

## 参考资料

笑死，一个多年 Fedora 用户看的文档大多都来自 Arch Wiki 。

- https://wiki.archlinux.org/title/Improving_performance#zram_or_zswap
- https://wiki.archlinux.org/title/Btrfs#Swap_file
- https://wiki.archlinux.org/title/Swap#Swap_file_creation
- https://wiki.archlinux.org/title/Fstab

---

克制 Rust 编译大型项目时 OOM 还有一些小技巧，也许下次可以水一点内容。

我没有摸鱼！（手动狗头）

