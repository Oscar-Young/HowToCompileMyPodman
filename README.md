# Podman Source Code 解析 

## 章節
- [前言](#前言)
- [學習 Rootless Container 一定要分析 Source Code ?](#學習-rootless-container-一定要分析-source-code-)
- [初次編譯 Podman](#%E5%88%9D%E6%AC%A1%E7%B7%A8%E8%AD%AF-podman)
- [在 Podman 每次執行時 HelloWorld](#%E5%9C%A8-podman-%E6%AF%8F%E6%AC%A1%E5%9F%B7%E8%A1%8C%E6%99%82-helloworld)
- [客製化 podman info 資訊](#%E5%AE%A2%E8%A3%BD%E5%8C%96-podman-info-%E8%B3%87%E8%A8%8A)

## 前言

Podman 官方網站對其本身的定義為 `Podman is a tool for managing OCI containers and pods`。

Podman 本質上就是一個 Command line 工具, 我們透過使用 `podman` 的命令，
能夠管理基於 `OCI (Open Container Interface)` 的標準所建立的 Container。

而 Podman 的是使用 Go 語言編譯而成，其主要核心為 `libpod`，`libpod` 是 Go 語言編寫的 Library，提供了管理 Container、Pod、Container Image、Volumes的 API。

所以要了解 Podman 的 Source code 就要去研究 libpod 的 Source Code。

本文件將引導你從頭開始研究 Podman 的 source code。

## 學習 Rootless Container 一定要分析 Source Code ?

能夠管理 OCI Container 的工具除了 Podman 之外，最廣為人知的絕對是 `Docker`，Docker 雖然因為其設計架構、安全性等問題遭到詬病，但其操作文件與使用教學相較於 Podman 完整許多，而 Podman 的主要貢獻者 `Red Hat` 目前仍然致力於推廣並完善 Podman 的相關文件，其在 Rootful Container 的操作與 Docker 的操作方式差不多，
甚至可以說幾乎相同，在一些操作文件中，甚至出現以下命令讓 docker 使用者能夠無縫接軌 podman。

```bash
alias docker=podman
```

但在 Rootless Container，也就是 Podman 強調在安全性遠勝於 docker 的部分，其操作文件極度欠缺，欠缺到[官方文件](https://github.com/containers/podman/blob/master/rootless.md)中都寫說這是Rootless Container 的缺點。

而在文件不足的情況下，想要了解 Rootless Container 的運作原理1不妨可以試試直接透過分析其原始碼。

## 初次編譯 Podman

本次操作皆在 Ubuntu 20.04 系統中進行。

此處參考 [Podman 官網安裝教學](https://podman.io/getting-started/installation)


### 安裝 podman

使用 apt 命令透過安裝 podman，可以在安裝過程中自動安裝編譯 podman 所需的套件。

```bash
$ . /etc/os-release
$ echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_${VERSION_ID}/ /" | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
$ curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_${VERSION_ID}/Release.key | sudo apt-key add -
$ sudo apt-get update
$ sudo apt-get -y upgrade
$ sudo apt-get -y install podman
```

使用 `podman info` 驗證安裝是否成功。

```
Version:      3.1.0-dev
API Version:  3.0.0
Go Version:   go1.13.8
Git Commit:   9ec8106841c55bc085012727748e2d73826be97d-dirty
Built:        Fri Feb 26 09:27:49 2021
OS/Arch:      linux/amd64
```

請留意此處的 `Built`，後續編譯完成後可進行比較。

### 安裝 Golang

```bash
$ sudo apt install golang
```

### 安裝 conmon

```
$ git clone https://github.com/containers/conmon
$ cd conmon
$ export GOCACHE="$(mktemp -d)"
$ make
$ sudo make podman
```

### 編譯 Podman source code

```bash
$ git clone https://github.com/containers/podman/
$ cd podman
$ make BUILDTAGS="selinux seccomp"
$ sudo make install PREFIX=/usr
```

再次使用 `podman info` 觀察編譯是否成功

```
Version:      3.1.0-dev
API Version:  3.0.0
Go Version:   go1.13.8
Git Commit:   9ec8106841c55bc085012727748e2d73826be97d-dirty
Built:        Fri Feb 26 09:47:37 2021
OS/Arch:      linux/amd64
```

可以觀察到 `Built` 的時間變更為剛才編譯結束後的時間了，至此已經學會如何自行編譯 Podman了。

## 在 Podman 每次執行時 HelloWorld

前言提過 podman 本質為 Command line 工具，其 Command Line 的判斷邏輯主要放在 `cmd/podman` 之中，而程式的起點在 `cmd/podman/main.go` 裡面，

透過觀察 `cmd/podman/main.go`中的 `func main()` 可以發現其主程式才區區幾行，再仔細看發現主程式中調用了 `func parseCommands()`，觀察後發現 Podman 是依賴 `cobra` 套件所產生的 Command line 工具，但 `cobra`套件並非本次主要研究的套件，所以如果需要修改其命令的用法可以去研究 `cobra` 套件是如何使用。
``` go
import (
...
	"github.com/sirupsen/logrus"
	"github.com/spf13/cobra"
)
func main() {
	if reexec.Init() {
		// We were invoked with a different argv[0] indicating that we
		// had a specific job to do as a subprocess, and it's done.
		return
	}

	rootCmd = parseCommands()

	Execute()
	os.Exit(0)
}
func parseCommands() *cobra.Command {
...
}

```

首先我們先嘗試使用 Go語言原生套件 `fmt` 在每次程式執行時都能輸出一段文字，請先 import `fmt`套件，並且在 `func main()`中加入 `println("Hello World")`

結果將會如下 :

``` go

import (
...
	"fmt"
...
)
func main() {
	println("Hello World")
	if reexec.Init() {
		// We were invoked with a different argv[0] indicating that we
		// had a specific job to do as a subprocess, and it's done.
		return
	}

	rootCmd = parseCommands()

	Execute()
	os.Exit(0)
}
```

接著我們需要再次將 Source code 進行編譯，請注意編譯時候的目錄一定要在 podman 專案的根目錄中，或是你可以嘗試使用 `cd ~/podman` 回到專案根目錄。

``` bash
$ make BUILDTAGS="selinux seccomp"
$ sudo make install PREFIX=/usr
```

編譯完成後可以嘗試使用 podman 命令驗證

`podman info` :

```
Hello WorldHello World
Version:      3.1.0-dev
API Version:  3.0.0
Go Version:   go1.13.8
Git Commit:   9ec8106841c55bc085012727748e2d73826be97d-dirty
Built:        Tue Mar  2 17:05:36 2021
OS/Arch:      linux/amd64
```

每次使用 podman 命令的時候都將會出現 Hello World 了 !

## 客製化 `podman info` 資訊

在上一章節，讓 podman 每次執行的時候都會輸出一段文字，接著要嘗試修改 `podman info` 顯示的資訊

打開 `cmd/podman/system/version.go` 並找到 `func formatVersion()`

```go

func formatVersion(w io.Writer, version *define.Version) {
	fmt.Fprintf(w, "Version:\t%s\n", version.Version)
	fmt.Fprintf(w, "API Version:\t%s\n", version.APIVersion)
	fmt.Fprintf(w, "Go Version:\t%s\n", version.GoVersion)
	if version.GitCommit != "" {
		fmt.Fprintf(w, "Git Commit:\t%s\n", version.GitCommit)
	}
	fmt.Fprintf(w, "Built:\t%s\n", version.BuiltTime)
	fmt.Fprintf(w, "OS/Arch:\t%s\n", version.OsArch)
}

```

前言提到 podman 在操作 Container、Pod、Container Image、Volume 時，依賴的是`libpod` 套件，所以顯然像是 `podman info` 這種顯示 Podman 本身的資訊的命令，並不歸屬於 `OCI` 標準的管轄範圍，所以再整個專案搜尋 `podman info` 所顯示的字串後，像是`Version:`、`Build:`、`API Version:` 等字串後，可以直接在 `cmd/podman/system` 的目錄中找到 `version.go` 中的 `func formatVersion()`，與上一章節輸出Hello World時相似的輸出方法，很明顯就是輸出 `podman info` 所顯示的文字。


依照程式碼的輸出語法來嘗試修改一些輸出內容。
以下範例修改 `Version` 為 `This is my podman Version`

```go

func formatVersion(w io.Writer, version *define.Version) {
    
	fmt.Fprintf(w, "This is my podman Version:\t%s\n", version.Version)
	fmt.Fprintf(w, "API Version:\t%s\n", version.APIVersion)
	fmt.Fprintf(w, "Go Version:\t%s\n", version.GoVersion)
	if version.GitCommit != "" {
		fmt.Fprintf(w, "Git Commit:\t%s\n", version.GitCommit)
	}
	fmt.Fprintf(w, "Built:\t%s\n", version.BuiltTime)
	fmt.Fprintf(w, "OS/Arch:\t%s\n", version.OsArch)
}

```

修改完成後，一樣回到專案根目錄中進行編義

``` bash
$ make BUILDTAGS="selinux seccomp"
$ sudo make install PREFIX=/usr
```

編譯完成後再次使用 podman 命令驗證

`podman info` :

```
Hello WorldHello World
This is my podman Version:      3.1.0-dev
API Version:  3.0.0
Go Version:   go1.13.8
Git Commit:   9ec8106841c55bc085012727748e2d73826be97d-dirty
Built:        Tue Mar  2 17:05:36 2021
OS/Arch:      linux/amd64
```

可以看到除了原先寫在 `cmd/podman/main.go` 的 Hello World 之外，`podman info` 的資訊也成功被修改了。
