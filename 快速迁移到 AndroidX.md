---
title: 迁移到AndroidX
date: 2019-10-27 17:12:03
tags: 
- Android
categories:
- Android
---

Android Studio 提供的 Migrate to AndroidX 功能不是特别完善，很多文件的包名替换不完整。可以通过自定义脚本来处理这个流程。

<!--more-->

包名映射文件下载地址： `https://developer.android.com/topic/libraries/support-library/downloads/androidx-class-mapping.csv`

脚本文件：

```shell
#!/usr/bin/env bash

MAPPING_FILE=csv文件路径
PROJECT_DIR=项目目录

replace=""
while IFS=, read -r from to
do
	replace+="; s/$from/$to/g"
done <<< "$(cat $MAPPING_FILE)"
fd . $PROJECT_DIR -e kt -e java -e xml --print0 | xargs -0 gsed -i "$replace"

```



注：mac 需要安装 fd 和 gnu-sed

```
brew install fd # fd 是 find 命令的替代品
brew install gnu-sed 
```



##  参考资料与学习资源推荐

- https://gist.github.com/dlew/5db1b780896bbc6f542e7c00a11db6a0
- https://gist.github.com/dudeinthemirror/cb4942e0ee5c3df0fcb678d1798e1d4d
- [迁移到 AndroidX](https://developer.android.com/jetpack/androidx/migrate)

由于本人水平有限，可能出于误解或者笔误难免出错，如果发现有问题或者对文中内容存在疑问请在下面评论区告诉我，谢谢！