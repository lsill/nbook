---
title: "go work"
date: 2022-04-24T16:44:48+08:00
draft: true
---

### go work 记录

本文档基于go1.18

#### 1.go work作用

​	官方说法：go 命令现在支持“工作区”模式。如果在工作目录或父目录中找到一个 go.work 文件，或者使用 GOWORK 环境变量指定了一个文件，它会将 go 命令置于工作空间模式。在工作区模式下，go.work 文件将用于确定用作模块解析根的主模块集，而不是使用通常找到的 go.mod 文件来指定单个主模块。

​	自己理解：本地引用的某个包修改，这个时候要么我提交到远端更新才能用，要么自己修改go mod文件，才能正确引用这个包，比较麻烦，但go work 出现，通过创建一个本地工作空间的模式，取代上述的操作能直接使用修改包。



#### 2. go work 命令

通过go help work

我们能得到以下命令

```shell
The commands are:
	edit        edit go.work from tools or scripts	// 通过tools或者scripts编辑go.work
	init        initialize workspace file	// 创建一个workspace
	sync        sync workspace build list to modules	// 将workspqce 构建列表同步到modules
	use         add modules to workspace file	// workspace新增go module
```

- go work init





大版本合并

![image-20220507173838724](/Users/lsill/Library/Application Support/typora-user-images/image-20220507173838724.png)

tag分支标签





删除照片墙的结构是这个

```
type GmCommonAccountModifyReqData_DelUserContent struct {
	Type  int32  `json:"Type"`  // 1 狐狸照片墙 2 聊天分享 3 美甲照片墙 5 联盟剧场图片  6 联盟剧场宣言 7 重置剧本 8 删除剧目
	SubID int32  `json:"SubID"` // 子ID  聊天分享表示类型  美甲中表示赛季id
	Param string `json:"Param"` // 狐狸中表示文件名 美甲 聊天分享中表示唯一ID
}
```
