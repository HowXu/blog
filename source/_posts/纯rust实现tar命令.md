---
title: 纯rust实现tar命令
date: 2026-5-29 10:00
tags: 
    - rust
index_img: 
---

# 正文

刷 B 站刷到了 Tsoding Daily 的视频（Emacs 神的编辑器，最古法编程之人）。刷到的那一期视频是[手写 C 语言实现 tar 打包与解析](https://www.bilibili.com/video/BV1PjR3BCEUb)，心血来潮就想自己动手实现一个。

看到一半发现教学思路比较多，构建和很多函数都用的是 nob.h 这个项目，当然 nob.h 肯定是不能在 Windows 上正常使用了。正好最近在学 Rust，所以我决定用 Rust 复现它。

本文由 [OpenCode](https://opencode.ai) TUI + [MiniMax](https://www.minimaxi.com/) M2.7 辅助写作

[仓库地址](https://github.com/HowXu/rust_lib.git)

# 关于 Tar

tar archive 文件由一系列 512 字节块组成，每个文件对应一个 header 块加若干数据块，末尾以两个全零块作为结束标记。

## Header 结构 ([POSIX](https://en.wikipedia.org/wiki/POSIX) 标准)

| 字段 | 大小 | 说明 |
|------|------|------|
| name | 100 | 文件名 |
| mode | 8 | 权限（八进制） |
| uid | 8 | 用户 ID |
| gid | 8 | 组 ID |
| size | 12 | 文件大小（八进制） |
| mtime | 12 | 修改时间（八进制时间戳） |
| chksum | 8 | 头部校验和 |
| typeflag | 1 | 文件类型标识 |
| linkname | 100 | 链接目标 |
| magic | 6 | ustar 魔数 |
| version | 2 | 版本号「00」 |
| uname | 32 | 用户名 |
| gname | 32 | 组名 |
| devmajor | 8 | 设备主号 |
| devminor | 8 | 设备次号 |
| prefix | 155 | 文件名前缀 |
| **total** | **500 字节** |

## typeflag 类型

- 「0」/「\0」— 普通文件
- 「1」— [硬链接](https://en.wikipedia.org/wiki/Hard_link)
- 「2」— [符号链接](https://en.wikipedia.org/wiki/Symbolic_link)
- 「3」— [字符设备](https://en.wikipedia.org/wiki/Device_file#Character_devices)
- 「4」— [块设备](https://en.wikipedia.org/wiki/Device_file#Block_devices)
- 「5」— 目录
- 「6」— [FIFO](https://en.wikipedia.org/wiki/Named_pipe)
- 「7」— 连续文件
- 「K」— 长链接名（下一个块是链接名）
- 「L」— 长文件名（下一个块是文件名）
- 「D」— dumpdir

根据我的实战经验，这个 Tar 格式充斥着各种来自 Tape 时代设计的对现在来说完全是无效设计的东西。比如 mtime，它现在完全不需要是八进制；version 被设计成 0x20 0x00，而 magic 又被设计成末尾为 [75, 73, 74, 61, 72, 20]。

我觉得 magic 就也应该是末尾 0x00 😡

不考虑 Link 这个情况和由于文件名过长产生的 [LongLink](https://www.gnu.org/software/tar/manual/html_node/Extensions.html)，再加上 12 字节的 padding，刚好凑到 512 了。

# 实战

## 一、类型定义与常量

我们先在 `def.rs` 中定义一些基础类型和常量，这些是整个项目的基础。

```rust
// def.rs

use std::fs::Metadata;
use std::mem;
use std::time::UNIX_EPOCH; // [Unix 时间](https://en.wikipedia.org/wiki/Unix_time) 起点 (1970-01-01 00:00:00 UTC)
use std::path::PathBuf;

// 类型别名，让代码更简洁
pub(super) type Byte = u8;
pub(super) type IOError = std::io::Error;
pub(super) type FMTError = std::fmt::Error;
```

`Byte`、`IOError`、`FMTError` 都是为了省打字存在的别名。

---

## 二、魔数常量

tar 格式要求特定的头部标识，我们定义两个常量：

```rust
// def.rs 续

// "ustar " -> [0x75, 0x73, 0x74, 0x61, 0x72, 0x20]
// 注意末尾是空格而不是 \0，这是 [POSIX](https://en.wikipedia.org/wiki/POSIX) 标准规定的
pub(super) const MAGIC_NUMBER: [Byte; 6] = [0x75, 0x73, 0x74, 0x61, 0x72, 0x20];

// " \0" -> [0x20, 0x00]
// 版本号固定为空格和 null
pub(super) const MAGIC_VERSION: [Byte; 2] = [0x20, 0x00];
```

读取 tar 文件时，靠这两个常量来验证合法性。

---

## 三、Header 结构体定义

这是最核心的结构 — 对应 tar 格式的 512 字节头部。

```rust
// def.rs 续

// 使用 C 的对齐方式（紧凑布局），保证结构体大小正好是 512 字节
#[repr(C)]
#[derive(Clone)]
pub(super) struct CommonHeader {
    pub(super) path: [Byte; 100],             // 文件名
    pub(super) mode: [Byte; 8],               // 权限（八进制）
    pub(super) owner_user: [Byte; 8],         // UID（八进制）
    pub(super) group_user: [Byte; 8],         // GID（八进制）
    pub(super) file_size: [Byte; 12],         // 文件大小（八进制）
    pub(super) modification_time: [Byte; 12], // 修改时间（八进制时间戳）
    pub(super) checksum: [Byte; 8],           // 头部校验和
    pub(super) type_flag: Byte,               // 文件类型标识
    pub(super) type_info: [Byte; 100],        // 链接名等辅助信息
    pub(super) ustar_indicator: [Byte; 6],    // 魔数「ustar 」
    pub(super) ustar_version: [Byte; 2],      // 版本「 \0」
    pub(super) owner_user_name: [Byte; 32],  // 用户名
    pub(super) owner_group_name: [Byte; 32], // 组名
    pub(super) device_major: [Byte; 8],      // 设备主号
    pub(super) device_minor: [Byte; 8],      // 设备次号
    pub(super) filename_prefix: [Byte; 155],  // 文件名前缀（长路径用）
    pub(super) paddings: [Byte; 12],          // 填充对齐到 512 字节
}
```

`#[repr(C)]` 确保内存布局和 C 语言一致，结构体大小正好是 512 字节。[`mem::transmute`](https://doc.rust-lang.org/core/mem/fn.transmute.html) 可以直接把结构体转成字节数组，供文件写入使用。

我们写个测试来验证：

```rust
#[test]
fn test_header_size() {
    assert_eq!(size_of::<CommonHeader>(), 512); // a header must be 512 size or a chunk
}
```

---

## 四、辅助结构体

除了 512 字节的 tar header，我们还需要一些运行时用的辅助结构。

```rust
// def.rs 续

// 解包时使用的临时结构，存储解析出的文件信息
pub(super) struct UntarInstance {
    pub(super) path: Box<PathBuf>,      // 文件绝对路径
    pub(super) is_folder: bool,         // 是否为目录
    pub(super) file_size: u64,          // 文件大小
    pub(super) seek_from: u64,          // 数据起始位置（以 512 字节块为单位）
}

// 打包时的中间结构，先收集文件信息，最后再转换成 CommonHeader
pub(super) struct EntarWrapper {
    pub(super) is_folder: bool,
    pub(super) instance: EntarInstance,
    pub(super) file_size: u64,
    pub(super) file_pointer: Box<PathBuf>, // 指向源文件的指针
}

struct EntarInstance {
    pub(super) path: [Byte; 100],             // 文件名（100 字节限制）
    pub(super) file_size: [Byte; 12],         // 文件大小（八进制）
    pub(super) modification_time: [Byte; 12], // 修改时间（八进制）
    pub(super) mode: [Byte; 8],               // 权限（八进制）
    pub(super) checksum: [Byte; 8],           // 校验和
}
```

`EntarWrapper` 是我们自己的封装层，先把文件信息存在这里，最后统一转成标准 tar header。

因为用的是 [Rust](https://www.rust-lang.org/)，所以在很多地方需要用到 [Box 指针](https://doc.rust-lang.org/book/ch15-01-box.html) 来减少心智负担。

---

## 五、CommonHeader 的默认构造函数

```rust
// def.rs 续

impl CommonHeader {
    pub(super) fn new() -> Self {
        Self {
            path: [0u8; 100],
            mode: [0u8; 8],
            owner_user: [0u8; 8],
            group_user: [0u8; 8],
            file_size: [0u8; 12],
            modification_time: [0u8; 12],
            checksum: [0u8; 8],
            type_flag: 0,
            type_info: [0u8; 100],
            ustar_indicator: [0u8; 6],
            ustar_version: [0u8; 2],
            owner_user_name: [0u8; 32],
            owner_group_name: [0u8; 32],
            device_major: [0u8; 8],
            device_minor: [0u8; 8],
            filename_prefix: [0u8; 155],
            paddings: [0u8; 12],
        }
    }
}
```

初始化所有字段为 0。

---

## 六、UntarInstance 的构造函数

```rust
// def.rs 续

impl UntarInstance {
    pub(super) fn new(path: Box<PathBuf>, is_folder: bool, file_size: u64, seek_from: u64) -> Self {
        Self {
            path,
            is_folder,
            file_size,
            seek_from,
        }
    }
}
```

简单构造，没什么花头。

---

## 七、EntarWrapper 核心逻辑

这里是打包的核心 —— 把文件元数据转换成 tar 格式。

```rust
// def.rs 续

impl EntarWrapper {
    // it: 源文件的绝对路径
    // rel: 相对于待打包目录的相对路径，用来生成 tar 中的文件名
    pub(super) fn new(it: PathBuf, rel: PathBuf, is_folder: bool, meta: Metadata) -> Self {
        // 1. 处理文件名：Windows 和 Linux 路径分隔符不同，统一转成 /
        let file_size = meta.len();
        let mut p: [Byte; 100] = [0u8; 100];
        if let Some(s) = rel.to_str() {
            let s_rp = s.replace('\\', "/"); // 统一斜杠方向
            let src = s_rp.as_bytes();
            let len = s.len().min(100); // 不能超过 100 字节
            p[..len].copy_from_slice(&src[..len]);
        } else {
            panic!("failed to parse PathBuf to_str");
        }

        // 2. 文件大小转八进制字符串，填充到 12 字节字段
        // 格式：前面补 0，末尾为 null
        // e.g. 文件大小 1024 -> "2000\0"（八进制）
        let file_size_to_bytes = format!("{:o}", file_size);
        let mut sz: [Byte; 12] = [0x30u8; 12]; // 默认填「0」
        let src = file_size_to_bytes.as_bytes();
        let len = file_size_to_bytes.len().min(11);
        sz[(11 - len)..11].copy_from_slice(&src[..len]); // 右对齐
        sz[11] = 0x00;                                   // 末尾 null

        // 3. 修改时间转八进制时间戳
        let mut mdf_time: [Byte; 12] = [0u8; 12];
        if let Ok(mdf) = meta.modified() {
            if let Ok(unix_timestamp) = mdf.duration_since(UNIX_EPOCH) {
                let bytes = format!("{:o}", unix_timestamp.as_secs());
                let bits = bytes.as_bytes();
                mdf_time[..11].copy_from_slice(&bits);
            } else {
                panic!("failed to get metadata unix timestamp");
            }
        } else {
            panic!("failed to get metadata modified time");
        }

        // 4. 权限处理：只读文件设 444，其他统一设 777
        let mode: [Byte; 8];
        if meta.permissions().readonly() {
            mode = [0x30, 0x30, 0x30, 0x30, 0x34, 0x34, 0x34, 0x00]; // 0444
        } else {
            mode = [0x30, 0x30, 0x30, 0x30, 0x37, 0x37, 0x37, 0x00]; // 0777
        }

        // 5. 组装 EntarInstance（先不填 checksum，那是由 as_header 算的）
        Self {
            is_folder,
            instance: EntarInstance {
                path: p,
                file_size: sz,
                modification_time: mdf_time,
                mode,
                checksum: [0u8; 8],
            },
            file_size,
            file_pointer: Box::new(it),
        }
    }
}
```

**重点：**
- Windows 路径 `\` 转 `/`，保证跨平台兼容（多数 tar 解析器是以 Unix 为标准的）
- 所有数字字段都是**八进制**字符串，这是 [POSIX](https://en.wikipedia.org/wiki/POSIX) 标准规定的
- 文件大小和修改时间都转成八进制格式

---

## 八、生成最终 Header（含校验和计算）

`EntarWrapper` 只是中间结构，最终要转成 512 字节的 `CommonHeader`，其中最关键的一步是计算校验和。

```rust
// def.rs 续

    pub(super) fn as_header(self: Self) -> CommonHeader {
        let mut header = CommonHeader::new();

        // 1. 写入魔数和版本号
        header.ustar_version = MAGIC_VERSION;
        header.ustar_indicator = MAGIC_NUMBER;

        // 2. 复制各字段
        header.file_size = self.instance.file_size;
        header.mode = self.instance.mode;
        header.modification_time = self.instance.modification_time;
        header.path = self.instance.path;

        // 3. 校验和字段先填空格（ASCII 0x20），因为计算校验和时要用空格替代
        header.checksum = [0x20, 0x20, 0x20, 0x20, 0x20, 0x20, 0x20, 0x20];

        // 4. UID/GID 填 0
        header.owner_user = [0x30; 8];
        header.owner_user[7] = 0x00;
        header.group_user = [0x30; 8];
        header.group_user[7] = 0x00;

        // 5. 文件类型：目录是「5」（0x35），普通文件是「0」（0x30）
        if self.is_folder {
            header.type_flag = 0x35;
        } else {
            header.type_flag = 0x30;
        }

        // 6. 【关键】计算校验和：
        //    把整个 header 当作字节数组求和，checksum 字段位置填空格
        let buffer: [Byte; 512] = unsafe { mem::transmute(header.clone()) };
        let mut chechsums = 0u64;
        for byte in buffer {
            chechsums += byte as u64;
        }

        // 7. 校验和转八进制字符串，写入 checksum 字段
        let checksum_s = format!("{:o}", chechsums);
        let checksum = checksum_s.as_bytes();
        let mut cs: [Byte; 8] = [0x30u8; 8];
        let len = checksum.len().min(6);
        cs[(6 - len)..6].copy_from_slice(&checksum); // 右对齐
        cs[7] = 0x20;                                 // 倒数第二位填空格
        cs[6] = 0x00;                                 // 末位 null
        header.checksum = cs;

        return header;
    }
}
```

**校验和计算规则（[POSIX](https://en.wikipedia.org/wiki/POSIX) 标准）：**
- 把 header 的 512 字节全部相加
- 计算时，`checksum` 字段位置填 8 个空格（0x20）
- 结果转八进制，前 6 位写入 checksum，最后一位填空格（0x20），末位填 null（0x00）

---

## 九、打包主逻辑：file_entar

有了上面的结构，打包就清晰了：遍历文件 → 生成 header → 写入数据。

```rust
// tar.rs

use crate::def::*;
use std::{
    collections::{LinkedList, VecDeque},
    fs::{self, File, OpenOptions},
    io::{ErrorKind, Read, Seek, SeekFrom, Write},
    mem,
    path::{Path, PathBuf},
};

// 打包函数：input_path 是待打包的文件夹，output_file 是输出的 tar 文件
pub fn file_entar<'f>(input_path: &'f Path, output_file: &'f Path) -> Result<(), IOError> {
    let mut headers: Vec<EntarWrapper> = vec![];

    // 1. 算出待打包目录的父目录，用来生成相对路径
    let father: &Path;
    if let Some(_father) = input_path.parent() {
        father = _father;
    } else {
        return Err(IOError::other("It's a top dir maybe / or other errors"));
    }

    // 2. 先把根目录本身加进去（作为目录）
    if let Ok(rel) = input_path.strip_prefix(father) {
        if let Ok(meta) = input_path.metadata() {
            headers.push(EntarWrapper::new(
                input_path.to_path_buf(),
                rel.to_path_buf(),
                true, // is_folder
                meta,
            ));
        }
    }

    // 3. 递归遍历所有文件，收集信息
    if let Err(e) = process_files_recursively(father, input_path, &mut headers) {
        return Err(e);
    }

    // 4. 打开（创建）输出文件，开始写入
    let _tar_file = OpenOptions::new()
        .write(true)
        .create(true)
        .truncate(true)
        .open(output_file);

    if let Ok(mut tar_file) = _tar_file {
        for header in headers {
            let is_folder = header.is_folder;
            let size = header.file_size;
            let pointer = header.file_pointer.clone();

            // 写入 512 字节 header
            let head_buffer: [Byte; 512] = unsafe { mem::transmute(header.as_header()) };
            if let Err(_) = tar_file.write(&head_buffer) {
                return Err(IOError::other("write header to tar file failed!"));
            }

            // 如果不是目录，写入文件内容（以 512 字节块为单位）
            if !is_folder {
                if let Ok(mut f) = File::open(pointer.as_path()) {
                    let bf_size = 512 as u64;
                    let mut writed = 0u64;

                    let mut content_buffer = vec![0u8; bf_size as usize];
                    while writed < size {
                        if let Ok(_) = f.read(&mut content_buffer) {
                            if let Ok(wd) = tar_file.write(&content_buffer) {
                                writed += wd as u64;
                                content_buffer = vec![0u8; bf_size as usize];
                            } else {
                                return Err(IOError::other("Error write tar file"));
                            }
                        } else {
                            return Err(IOError::other("Error read source file"));
                        }
                    }
                }
            }
        }

        // 5. 末尾写入两个全零块（tar 标准结束标记）
        let empty_chunk: [Byte; 512] = [0x00u8; 512];
        for _ in 0..2 {
            if let Err(_) = tar_file.write(&empty_chunk) {
                return Err(IOError::other("Error write two empty chunks"));
            }
        }

        if let Err(_) = tar_file.flush() {
            return Err(IOError::other("Error flush tar file"));
        }
    }

    Ok(())
}
```

---

## 十、递归遍历文件

```rust
// tar.rs 续

// 广度优先遍历：先处理当前目录的所有文件和子目录，再递归处理子目录
pub fn process_files_recursively<'f>(
    father: &'f Path,
    input_path: &'f Path,
    vec: &mut Vec<EntarWrapper>,
) -> Result<(), IOError> {
    let mut dirs: VecDeque<Box<PathBuf>> = VecDeque::new(); // 待处理的子目录队列

    for entry in fs::read_dir(input_path)? {
        let e = entry?;
        let p = e.path();

        if let Ok(rel) = p.strip_prefix(father) {
            if p.is_file() {
                // 普通文件：收集信息
                if let Ok(f) = File::open(p.as_path()) {
                    if let Ok(meta) = f.metadata() {
                        vec.push(EntarWrapper::new(
                            p.to_path_buf(),
                            rel.to_path_buf(),
                            false, // not folder
                            meta,
                        ));
                    }
                }
            } else if p.is_dir() {
                // 目录：先加进去，再加入待处理队列
                if let Ok(meta) = e.metadata() {
                    vec.push(EntarWrapper::new(
                        p.to_path_buf(),
                        rel.to_path_buf(),
                        true, // is folder
                        meta,
                    ));
                    dirs.push_back(Box::new(p.as_path().to_path_buf()));
                }
            }
        }
    }

    // 递归处理子目录
    for dir in dirs {
        if let Err(_e) = process_files_recursively(father, dir.as_path(), vec) {
            panic!("process_files_recursively failed !");
        }
    }

    Ok(())
}
```

注意这里是**广度优先**遍历，先处理完当前层再进子目录。

---

## 十一、解包主逻辑：file_untar

解包是打包的逆过程：读取 header → 解析文件信息 → 读取数据 → 写文件。

```rust
// tar.rs 续

// 解包函数：input_file 是 tar 文件，output_path 是输出目录
pub fn file_untar<'f>(input_file: &'f Path, output_path: &'f Path) -> Result<(), IOError> {
    let tar_file = File::open(input_file)?;
    let metadata = tar_file.metadata()?;
    let sz = metadata.len();

    // tar 文件以 512 字节块为单位，末尾有两个全零块作为结束标记
    let stop_index: u64 = (sz / 512) as u64 - 2;

    let mut chunk: [Byte; 512] = [0u8; 512];
    let mut chunk_index: u64 = 0;

    // 存储解析出的所有文件信息
    let infos = &mut LinkedList::new();

    // 1. 循环读取每个 512 字节块
    loop {
        match (&tar_file).read_exact(&mut chunk) {
            Ok(_) => {
                // 把字节数组直接解释成 CommonHeader（内存布局完全兼容）
                let header: CommonHeader = unsafe { mem::transmute(chunk) };

                // 2. 验证魔数
                if !(header.ustar_indicator == MAGIC_NUMBER) {
                    return Err(IOError::other(
                        "maybe a longlink tar file or ustar indicator failed",
                    ));
                }
                if !(header.ustar_version == MAGIC_VERSION) {
                    return Err(IOError::other(
                        "maybe a longlink tar file or ustar version failed",
                    ));
                }

                // 3. 解析 header 中的文件名
                let path = String::from_utf8_lossy(&header.path)
                    .into_owned()
                    .trim_end_matches('\0')
                    .to_string();

                // 4. 解析文件大小（八进制转 u64）
                let file_size: u64;
                if let Ok(fsz) = u64::from_str_radix(
                    String::from_utf8_lossy(&header.file_size)
                        .into_owned()
                        .trim_end_matches('\0'),
                    8,
                ) {
                    file_size = fsz;
                } else {
                    return Err(IOError::other("Parse file size failed"));
                }

                // 5. 计算数据块的偏移量（向上取整到 512 字节边界）
                let maybe_a_dir = output_path.join(&path);
                chunk_index += 1;

                if header.type_flag == 0x35 {
                    // 目录：直接记录，文件大小为 0
                    infos.push_back(UntarInstance::new(Box::new(maybe_a_dir), true, 0, 0));
                } else {
                    // 普通文件：记录信息，并跳过对应的数据块
                    let offset = (file_size + 511) / 512; // 向上取整
                    infos.push_back(UntarInstance::new(
                        Box::new(maybe_a_dir),
                        false,
                        file_size,
                        chunk_index,
                    ));
                    chunk_index += offset;
                    (&tar_file).seek(std::io::SeekFrom::Start(chunk_index * 512))?;
                }

                if chunk_index == stop_index {
                    break;
                }
            }
            Err(e) if e.kind() == ErrorKind::UnexpectedEof => break, // 正常结束
            Err(e) => return Err(e),
        }
    }

    // 6. 根据收集到的文件信息，执行实际的创建/写入操作
    infos.iter().for_each(|info| {
        if info.is_folder {
            // 创建目录
            if let Err(e) = fs::create_dir_all((&info).path.as_path()) {
                println!("failed to create folder: {}", e);
            }
        } else {
            // 写入文件内容
            let path = (&info).path.as_path();
            if let Some(path_str) = path.to_str() {
                if let Ok(mut tmp) = OpenOptions::new()
                    .write(true)
                    .create(true)
                    .truncate(true)
                    .open(path)
                {
                    let mut writed: u64 = 0;

                    // 根据文件大小选择合适的缓冲区大小
                    let sutaible_size: usize = if info.file_size > 4096 {
                        4096
                    } else if info.file_size > 512 {
                        512
                    } else {
                        info.file_size as usize
                    };

                    while writed < info.file_size {
                        // seek 到正确位置读取数据
                        if let Ok(_) = (&tar_file).seek(SeekFrom::Start(info.seek_from * 512 + writed))
                        {
                            let malloc_size = if info.file_size - writed > sutaible_size as u64 {
                                sutaible_size
                            } else {
                                (info.file_size - writed) as usize
                            };
                            let mut big_buffer = vec![0u8; malloc_size];
                            if let Ok(bytes_count) = (&tar_file).read(&mut big_buffer) {
                                let _ = tmp.seek(SeekFrom::End(0));
                                let _ = tmp.write(&big_buffer);
                                writed += bytes_count as u64;
                                let _ = tmp.flush();
                            }
                        }
                    }
                }
            }
        }
    });

    Ok(())
}
```

**解包流程总结：**
1. 读取 512 字节 header，验证魔数和版本号
2. 解析文件名（去 null）和文件大小（八进制转 u64）
3. 如果是目录 → 记录下来；如果不是 → 计算需要跳过的数据块数量
4. 全部 header 解析完后，再统一执行创建目录/写入文件的操作

---

## 文末

这个实现精简了很多 historical baggage，不支持 [LongLink](https://www.gnu.org/software/tar/manual/html_node/Extensions.html)、不支持[硬链接](https://en.wikipedia.org/wiki/Hard_link)/[软链接](https://en.wikipedia.org/wiki/Symbolic_link)、也不支持[稀疏文件](https://en.wikipedia.org/wiki/Sparse_file)。

但是我说实话，我写的比 Emacs 之神写的好了 🥰🥰🥰。Emacs 之神也只是简单实现了 untar 这个操作，entar 甚至没有实现对目录的 tar 操作。

除此之外，Rust 的这个错误处理机制设计得简直是一坨啊，用 `?` 需要新建很多变量，`unwrap` 又完全是赌徒策略。除此之外，写 tar 这种需要在底层和 byte 交互的东西确实是 pure C 会好很多，Rust 很多地方设计得并不像一个底层语言，像是 C++。虽然很多 `to_bytes` 函数和 `mem::transmute` 确实也够了，但是如果真得像 pure C 那样进内核我估计到处都是 unsafe 了。

不过 Rust 确实解决了我不想解决的 nob.h 问题和项目配置问题，因为写单元测试真的很简单了。总之各有各的好吧，下期可能出 C 和 Rust 交叉编译的内容。