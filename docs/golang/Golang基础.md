视频：[最新Go语言急速入门视频教程（七米出品）](https://www.bilibili.com/video/BV1ZJ411W7jG)

思维导图：https://www.processon.com/view/link/5ecb3925f346fb6907154970#map



依旧是Hello,World!

新建main.go

```go
package main

import "fmt"

func main() {
    // 单行注释，和Java一样
	fmt.Println("Hello,World!")	
}
```

生成.exe

```
go build main.go
```



### 常量和变量

> var
>
> const





















```go
package main

import (
	"fmt"
	"os"
)

func main() {
	var s, sep string
	//os.Args

	for i := 1; i < len(os.Args); i++ {
		s += sep + os.Args[i]
		sep = " "
	}
	fmt.Println(s)
}
```

