# WinDbg的使用

### 实验过程

- 点击 `File - Attach to a Process(F6)` 附加到notepad进程，列出已分配的堆，查找字符串 `ABCDABCD`

  ![1](img/1.png)

- 修改`ezu 0000026f f608d600 "AAAAAAAA"`

  ![2](img/2.png)