# pgpageshell


  一款用于在页面级别检查 PostgreSQL 数据文件的桌面应用程序。

  **[在线试用](https://jespino.github.io/pgpageshell/)** — 无需安装，预加载了 [Pagila](https://github.com/devrimgunduz/pagila) 示例数据库。

PostgreSQL 将所有表和索引数据存储在磁盘上的 8KB 页面中。这些页面具有定义明确的二进制格式——包括头部、行指针、元组数据、MVCC
元数据、索引特定结构——但 PostgreSQL 本身没有提供直接查看这些内容的内置方式。`pgpageshell` 允许你打开任意 PostgreSQL 数据文件，
并在其页面间导航，查看支撑每个查询的原始数据结构。

## 安装

### Debian / Ubuntu

从 [最新版本](https://github.com/jespino/pgpageshell/releases/latest) 下载 `.deb` 包并安装：

```bash
sudo dpkg -i pgpageshell_*.deb
```

### RPM（Fedora / RHEL）

```bash
sudo rpm -i pgpageshell_*.rpm
```

### 二进制文件下载

Linux（amd64 和 arm64 架构）和 Windows（amd64 架构）的预编译二进制文件可在
[发布页面](https://github.com/jespino/pgpageshell/releases/latest) 获取。
下载对应平台的压缩包，解压后将二进制文件放入你的 `PATH` 环境变量指向的目录。
对于 macOS 系统，请从源码构建（见下文）。

### 从源码构建

构建需要 Go 1.22+、Node.js 20+ 和 pnpm。桌面版本构建还需安装 [Wails v2](https://wails.io/)。

```bash
# 安装 Wails 命令行工具（如未安装）
go install github.com/wailsapp/wails/v2/cmd/wails@latest

# 构建桌面应用程序
make
```

## 使用方法

### 桌面应用程序（默认）

```bash
# 无文件启动——打开欢迎界面，显示“打开文件”按钮
./pgpageshell

# 预加载一个或多个文件启动
./pgpageshell /var/lib/postgresql/data/base/16384/17543
./pgpageshell file1 file2 file3
```

图形界面（GUI）展示一个页面网格（32×64 单元格，每个单元格 4 字节），并对不同区域进行颜色编码：
- 头部（蓝色）
- 行指针（绿色）
- 元组（紫色）
- 空闲空间（灰色）
- 特殊区域（橙色）

索引元页面、哈希位图页面和 BRIN 反向映射页面使用专用颜色（金色、青色、紫色），并在鼠标悬停时显示字段级详情——每个单元格会展示结构体字段名和解码后的值。

悬停在单元格上时，会高亮相关元素（例如，行指针及其对应的元组）。
侧边栏列出所有页面，并显示所选元素的详细信息。
图例会自适应显示当前页面中存在的区域类型。

你可以随时通过工具栏打开其他文件或关闭已打开的文件。

<div align="center">
  <img src="screenshot.webp" alt="pgpageshell 截图" width="800">
</div>

你可以通过以下 SQL 语句获取表或索引的文件路径：

```sql
SELECT pg_relation_filepath('my_table');
-- 返回示例：base/16384/17543


```

查看数据目录路径：

```sql
show data_directory;
       data_directory        
-----------------------------
 /var/lib/postgresql/16/main
```

### 导出为 JSON

将页面数据导出为 JSON 格式，以便在静态网页版或外部工具中使用：

```bash
./pgpageshell --export-json file1 file2 > data.json

# 使用自定义显示名称
./pgpageshell --export-json "actor=/path/to/16423" "actor_pkey (btree)=/path/to/16606" > data.json
```

### 交互式 Shell

```bash
./pgpageshell --shell <postgres-data-file>
```

该 Shell 提供基于文本的页面内部结构检查功能：

| 命令 | 描述 |
|---------|-------------|
| `page <n>` | 按编号选择页面（从 0 开始） |
| `cat` | 整个 8192 字节页面的十六进制转储 |
| `format` | 页面区域的 ASCII 可视化展示 |
| `info` | 解码后的页面头部和特殊区域数据 |
| `data` | 行指针表和解码后的元组数据 |
| `pages` | 文件中所有页面的摘要信息 |
| `help` | 显示命令列表 |
| `quit` | 退出 |

## 支持的页面类型

`pgpageshell` 会从特殊区域自动检测页面类型，并进行相应解码：

| 类型 | 检测方式 | 特殊区域内容 |
|------|-----------|------------------------|
| **堆（Heap）** | 无特殊空间 | — |
| **B 树（B-tree）** | 16 字节特殊区域，包含有效的 btpo_flags | 前/后兄弟节点、层级、标志位。元页面显示字段级详情（魔数、根节点、层级、快速根节点）。内部页面显示子块指针。 |
| **哈希（Hash）** | 16 字节特殊区域，page_id = `0xFF80` | 前/后块、桶编号、页面类型。元页面显示字段级详情（魔数、元组数量、填充因子、掩码、备用值、映射）。位图页面显示每个字的比特计数。 |
| **GiST** | 16 字节特殊区域，page_id = `0xFF81` | NSN、右链接、标志位（叶子/已删除/跟随右链接）。 |
| **GIN** | 8 字节特殊区域，包含有效标志位 | 右链接、最大偏移量、标志位（数据/叶子/元/列表/压缩）。元页面显示字段级详情（待处理列表、条目/数据页面计数、条目数量）。 |
| **SP-GiST** | 8 字节特殊区域，page_id = `0xFF82` | 标志位（元/已删除/叶子/空值）、重定向和占位符计数。元页面显示字段级详情（魔数、最近使用页面缓存）。 |
| **BRIN** | 8 字节特殊区域，type = `0xF091`–`0xF093` | 标志位、页面类型（元/反向映射/常规）。元页面显示字段级详情（魔数、版本、每个范围的页面数）。反向映射页面显示每个条目的（块、偏移量）目标。 |

## 详细教程

请参阅 [**pgpageshell 详细教程.md**](pgpageshell 详细教程.md) 获取实操指南，该指南使用Pagila 示例数据库从原始页面数据的角度探索堆页面、B 树索引、哈希索引、GiST、GIN 和 BRIN 索引。

## pgpageshell 原理

基本原理就是基于Postgresql 堆文件，索引文件的格式约定，逐页的读取Page。读取Page 之后定位 元组，读取元组的头部信息。然后展示出来。

读Page:

```go
func ReadPage(filename string, pageNum int) (*Page, error) {
	f, err := os.Open(filename)
	if err != nil {
		return nil, fmt.Errorf("open: %w", err)
	}
	defer f.Close()

	offset := int64(pageNum) * PageSize
	if _, err = f.Seek(offset, io.SeekStart); err != nil {
		return nil, fmt.Errorf("seek to page %d: %w", pageNum, err)
	}

	var data [PageSize]byte
	n, err := io.ReadFull(f, data[:])
	if err != nil {
		return nil, fmt.Errorf("read page %d (got %d bytes): %w", pageNum, n, err)
	}

	p := ParsePage(data)
	p.PageNum = pageNum
	return p, nil
}
```

分段解码Page 内容：

```go
func ParsePage(data [PageSize]byte) *Page {
	p := &Page{Data: data}
	le := binary.LittleEndian

	xlogid := le.Uint32(data[0:4])
	xrecoff := le.Uint32(data[4:8])
	p.Header.LSN = uint64(xlogid)<<32 | uint64(xrecoff)
	p.Header.Checksum = le.Uint16(data[8:10])
	p.Header.Flags = le.Uint16(data[10:12])
	p.Header.Lower = le.Uint16(data[12:14])
	p.Header.Upper = le.Uint16(data[14:16])
	p.Header.Special = le.Uint16(data[16:18])
	p.Header.PageSizeVer = le.Uint16(data[18:20])
	p.Header.PruneXID = le.Uint32(data[20:24])

	numItems := 0
	if p.Header.Lower > PageHeaderSize {
		numItems = int(p.Header.Lower-PageHeaderSize) / ItemIdSize
	}
	p.Items = make([]ItemId, numItems)
	for i := 0; i < numItems; i++ {
		off := PageHeaderSize + i*ItemIdSize
		p.Items[i] = ItemId{Raw: le.Uint32(data[off : off+4])}
	}

	p.Detected = p.detectPageType()
	return p
}
```

解码 tuple 头部信息

```go
func (p *Page) ParseHeapTupleHeader(offset uint16) HeapTupleHeader {
	// d 为 tuple 起始位置
	d := p.Data[offset:]
	le := binary.LittleEndian
	var t HeapTupleHeader
	t.Xmin = le.Uint32(d[0:4])
	t.Xmax = le.Uint32(d[4:8])
	t.Field3 = le.Uint32(d[8:12])
	biHi := le.Uint16(d[12:14])
	biLo := le.Uint16(d[14:16])
	t.CtidBlock = uint32(biHi)<<16 | uint32(biLo)
	t.CtidOffset = le.Uint16(d[16:18])
	t.Infomask2 = le.Uint16(d[18:20])
	t.Infomask = le.Uint16(d[20:22])
	t.Hoff = d[22]
	return t
}

func (p *Page) ParseIndexTupleHeader(offset uint16) IndexTupleHeader {
	d := p.Data[offset:]
	le := binary.LittleEndian
	var it IndexTupleHeader
	biHi := le.Uint16(d[0:2])
	biLo := le.Uint16(d[2:4])
	it.TidBlock = uint32(biHi)<<16 | uint32(biLo)
	it.TidOffset = le.Uint16(d[4:6])
	it.Info = le.Uint16(d[6:8])
	return it
}
```

## 参考链接

- Postgresql Page 布局 ：[PG Page Layout](https://www.postgresql.org/docs/current/storage-page-layout.html)