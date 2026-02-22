+++
date = '2025-11-04T12:49:14+08:00'
draft = false
title = '[xv6 學習紀錄 09-2] Lec14 File Systems'
series = ["xv6 學習紀錄"]
weight = 92
+++

課程連結：[6.S081 Fall 2020 Lecture 14: File Systems](https://www.youtube.com/watch?v=ADzLv1nRtR8)
{{< youtube ADzLv1nRtR8 >}}

# 實際操作
在影片中 50:00 左右的地方開始有實際的操作

```sh
make clean
make
```

* 先修改 `kernel/bio.c: bwrite()`

```c
// Write b's contents to disk.  Must be locked.
void
bwrite(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("bwrite");
  if (b->blockno >= 32) 
    printf("write: %d\n", b->blockno);
  virtio_disk_rw(b, 1);
}
```

