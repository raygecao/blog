---
weight: 20
title: "Golang应用中的一些IO优化"
subtitle: ""
date: 2022-04-20T13:02:27+08:00
lastmod: 2022-04-20T13:02:27+08:00
draft: false
author: "raygecao"
authorLink: ""
description: ""
resources:
- name: "featured-image"
  src: "featured-image.png"

tags: ["go"]
categories: ["探索与实战"]


toc:
  auto: false
lightgallery: true
license: ""
---

Go Writer 和 Reader接口的设计遵循了Unix的输入和输出，一个程序的输出可以是另外一个程序的输入，与之类比，Reader将流中的数据写进缓冲中，是一种数据的流入；Writer将缓冲的数据写至流，是一种数据的流出。
<!--more-->

## 问题背景

ToB的业务场景中的一种常见模式是私有云部署。对于一些保密性质的机关单位，应用只能在私有网络下部署，因此将数据包、镜像等部署所需的**artifact**克隆到用户现场便成为私有化交付中重要的一环。

完整流程大致如下：

- 解析对应的应用编排包及依赖，并下载所需的镜像及数据包（**artifact**）
- 将artifacts（可能需要压缩）、编排包pack成tar包
- 将tar包通过离线存储设备带到现场
- 在现场网络环境下将tar包load到私有化集群中

项目中采用virtual FS来实现封包/解包函数，借助两个helper函数：

```go
// ArchiveFS 将fs封包成tar并写到buf中
func ArchiveFS(buf io.Writer, files fs.FS) error {
	tw := tar.NewWriter(buf)
	err := fs.WalkDir(files, "", func(path string, entry fs.DirEntry, err error) error {
		if err != nil {
			return err
		}
		if path == ".git" || path == ".idea" {
			return filepath.SkipDir
		}
		// don't put "" into tar, since it can't be parsed
		if path == "" {
			return nil
		}
		file, err := files.Open(path)
		if err != nil {
			return err
		}
		defer file.Close()
		info, err := entry.Info()
		if err != nil {
			return err
		}
		hdr, err := tar.FileInfoHeader(info, "")
		if err != nil {
			return err
		}
		hdr.Name = normalizeFilepathSeparator(path)
		if info.IsDir() {
			hdr.Mode = 0o755
		} else {
			hdr.Mode = 0o644
		}
		if err := tw.WriteHeader(hdr); err != nil {
			return err
		}
		// dir must be writen to tar header for index files within it
		if info.IsDir() {
			return nil
		}
		if _, err := io.Copy(tw, file); err != nil {
			return err
		}
		return nil
	})
	tw.Close()
	if err != nil {
		return err
	}
	return nil
}

// LoadArchive 将tar解包到memory fs中
func LoadArchive(r io.Reader) (afero.Fs, error) {
	t := tar.NewReader(r)
	fs := afero.NewMemMapFs()

	for {
		hdr, err := t.Next()
		if err == io.EOF {
			break
		}
		if err != nil {
			return nil, err
		}
		if hdr.FileInfo().IsDir() {
			path := hdr.Name
			fs.MkdirAll(path, 0o755)
			fs.Chtimes(path, time.Now(), time.Now())
			continue
		}

		var buf bytes.Buffer
		size, err := buf.ReadFrom(t)
		if err != nil {
			panic("tarfs: reading from tar:" + err.Error())
		}
		if size != hdr.Size {
			panic("tarfs: size mismatch")
		}
		afero.WriteFile(fs, hdr.Name, buf.Bytes(), 0o644)
	}
	return fs, nil
}
```

起初，这两个函数只是作为编排包（几KB）的封包/解包helper。后来引入了带artifact的全量打包，为了方便就复用了这两个helper函数。最终在测试时发现在模拟现场加载tar包时出现OOM的问题。

## 问题分析

上述两个helper的Reader/Writer的入参是`bytes.Buffer`，virtual FS都是memory FS。这样就导致每调用一次helper函数时，都会在内存中新增一份全量包的拷贝。对于应用编排包而言，其大小不过几KB，完全可以忽略掉内存的变化。而当引入了镜像及数据包这类size比较大的artifact时，pack出的全量包一度达到20+GB。即使在内存中完整存在一份全量包，都可能引入OOM的风险。因此我们考虑通过对IO进行优化，避免将全量包加载到内存。

经过分析，我们的处理流程中有以下部分可以优化：

- Artifact的下载都是通过http call，通过`ioutil.ReadAll(resp.Body)`copy到`bytes.Buffer`中。可以考虑将artifact下载到`os.File`中，借助磁盘节省内存。
- 内存里维护memory FS存放各类artifacts，可以考虑通过物理文件系统来维护相应的结构。
- FS的打包是通过上述`ArchiveFS` helper将FS以tar的形式写到bytes.Buffer中，可以考虑直接写到`os.File`中。
- Artifact的上传是通过http.Post形式发送大文件，可以改造成流式传输。

FS的优化比较简单，可以直接通过tempDir来构造物理文件系统替换掉memory FS，下面着重讨论相应的IO优化。

## IO优化

选用`bytes.Buffer`的好处是其实现了`io.Reader`与`io.Writer`方法，可以灵活地读写。此外，其可以很方便地获取到完整的字节流，便于额外构造一个新的reader，这对于我们既要将reader写入文件，也要通过reader算md5的场景提供了便利。

### 使用`os.File`来替代`bytes.Buffer`

```go
file, err := os.Create(tempPath)
defer file.Close()
resp, err := http.Get(url)
// ignoring handle err
defer resp.Body.Close()
// file作为writer，放置下载后的artifact
io.Copy(file, resp.Body)

// file作为reader，计算MD5
hash := md5.New()
io.Copy(hash, file)
// ... other logic
```

上述会将artifact下载到`tempPath`下，但是**计算出的md5值总为空内容的md5**。

究其原因，`os.File`写完后不能直接读，其在write后会将offset置为写入末尾处，**需要手动将offset seek到文件开头才能读取完整的文件**。举个例子：

```go
a := "text.txt"
file, _ := os.Create(a)
file.WriteString("test")

// file.Seek(0, io.SeekStart)
buf := new(bytes.Buffer)
buf.ReadFrom(file)
fmt.Println(buf.String())
```

输出结果为`""`，去掉第5行注释的话，输出的结果便是`test`。

### 使用`io.TeeReader`实现双写

我们的案例中存在两种双写的场景：

- 将下载的内容写入到文件，同时计算md5
- 由于某种artifact存储方式为content-addressable，下载到文件的同时需要写入本地文件系统某个缓存目录（类比docker）

使用`bytes.Buffer`可以方便地获取到内容，从而构建另一个reader，完成独立的两次写入。除此之外，`io` package为我们提供了专门的双写方法：`io.TeeReader`，其实现十分简单：

```go
func TeeReader(r Reader, w Writer) Reader {
	return &teeReader{r, w}
}

func (t *teeReader) Read(p []byte) (n int, err error) {
	n, err = t.r.Read(p)
	if n > 0 {
		if n, err := t.w.Write(p[:n]); err != nil {
			return n, err
		}
	}
	return
}
```

`teeReader`内部wrap了一个writer，在进行read时，同时会将读出来的内容write到内部的writer中。在我们的场景下，以计算md5并写入文件为例：

```go
// reader comes from http.Body
hash := md5.New()
reader := io.TeeReader(reader, hash)
if _, err := io.Copy(writer, reader); err != nil {
   return err
}
hashsum := fmt.Sprintf("%x", hash.Sum(nil))
// handle hashsum
```

### 使用io.Pipe实现读写管道

`io.Pipe`是native go的一种实现，**类似于生产者消费者模式**，主要靠channel实现。在使用中，读写需要是异步的，否则会出现死锁的情况。在我们的案例中，加载全量包时会以http post的形式发送大文件。通常我们会在内存中构建body：

```go
// bytes.Buffer构造post body
buf := new(bytes.Buffer)
writer := multipart.NewWriter(buf)
defer writer.Close()
part, err := writer.CreateFormFile("upload", fn)
if err != nil {
    return err
}
if _, err = io.Copy(part, file); err != nil {
    return err
}
http.Post(url, writer.FormDataContentType(), buf)
```

当`file`的size很大时（我们场景会是GB量级），body占用的内存会增大，增加了OOM的风险，因此我们采用io.Pipe进行内存优化：

```go
r, w := io.Pipe()
m := multipart.NewWriter(w)
// 异步写入，否则会出现死锁
go func() {
    defer w.Close()
    defer m.Close()
    part, err := m.CreateFormFile("upload", fn)
    if err != nil {
        return
    }
    if _, err = io.Copy(part, file); err != nil {
        return
    }
}()
http.Post(url, m.FormDataContentType(), r)
```

观察`io.Pipe`的核心实现，主要依赖于一个数据channel流式地进行数据传递：

```go
func Pipe() (*PipeReader, *PipeWriter) {
	p := &pipe{
		wrCh: make(chan []byte),
		rdCh: make(chan int),
		done: make(chan struct{}),
	}
	return &PipeReader{p}, &PipeWriter{p}
}


func (p *pipe) write(b []byte) (n int, err error) {
	// ignore some control logic
	for once := true; once || len(b) > 0; once = false {
		select {
    // wrCh是数据channel
		case p.wrCh <- b:
      // rdCh是读写计数channel
			nw := <-p.rdCh
			b = b[nw:]
			n += nw
		case <-p.done:
			return n, p.writeCloseError()
		}
	}
	return n, nil
}

func (p *pipe) read(b []byte) (n int, err error) {
	// ignore some control logic
	select {
	case bw := <-p.wrCh:
		nr := copy(b, bw)
		p.rdCh <- nr
		return nr, nil
	case <-p.done:
		return 0, p.readCloseError()
	}
}
```

## 一坑多踩

除了OOM的问题，在开发过程中经常出现**unexpected  EOF**的报错。此类问题排查的切入点比较模糊，经过深入地单步调试才得以定位：项目中使用了`tar.Writer`进行封包，在使用`tar.Reader`解包时报EOF错误。这意味着这个tar包不是一个合法的tar包，使用tar命令去操作此tar包也会有类似的报错。

究其原因，tar会在封包结束后，通过`Close()`方法**在包尾部加入一层padding**：

```go
// Flush is called by tw.Close()
func (tw *Writer) Flush() error {
	if tw.err != nil {
		return tw.err
	}
	if nb := tw.curr.logicalRemaining(); nb > 0 {
		return fmt.Errorf("archive/tar: missed writing %d bytes", nb)
	}
	if _, tw.err = tw.w.Write(zeroBlock[:tw.pad]); tw.err != nil {
		return tw.err
	}
	tw.pad = 0
	return nil
}
```

我们在封包时，没有调用`Close()`，也就不会再尾部添加padding。在解包时，如果没有在尾部发现这个padding，就无法定位此tar包结束的位置，因此会认为此tar包是invalid，因此抛出`unexpected EOF`

那么，为什么会多此踩到这个坑呢？

除了使用了tar进行封包，我们还是用了`zstd.Writer`对artifact进行了压缩，同样忘记了调用`Close()`方法。在此总结出一条经验教训：对于`ReadCloser`以及`WriteCloser`的实现，在读/写完一定要记得调用`Close()`方法，否则对于前者读而言，存在内存泄露的风险；对于后者写而言，存在因缺失尾部标识而导致包格式错误。

## 经验总结

本文描述了与**io相关的OOM问题及unexpected EOF问题**，总结了如下经验：

- 当待处理的内容足够大时，避免使用`bytes.Buffer`加载到内存中进行处理，**借助磁盘或管道减少内存占用**
- `os.File`写入后直接读，需要将offset重置到起始处
- 使用http.Post传大文件时，可以使用`io.Pipe`构建body，减少内存占用
- 对于所有的`WriteCloser`的实现，写完记得调用`Close()`以防写出的格式不完整

---
配图引自 https://medium.com/learning-the-go-programming-language/streaming-io-in-go-d93507931185
