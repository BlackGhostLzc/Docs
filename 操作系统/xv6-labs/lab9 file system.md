## **Large Files**

我们知道，在我们现在的xv6文件系统设计中，每一个文件最大只能有268*BSIZE(12 + 256)，一个文件(inode)有12个direct block，还有1个singlely-indirect block。

```c
uint addrs[NDIRECT+1];    // struct inode
```

这里，我们将为文件进行扩容，添加一个doubly-indirect的block，现在`addrs` 数组仍旧有NDIRECT+1(13个)，我们现在9个direct block,1个singlely-indirect block,1个doubly-indirect block，所以现在文件的最大大小为：(11+256+256*256 ) * BSIZE。



下面我们看代码实现：

1. 更改`inode` 和`dinode` 结构体。
2. 修改`bmap(struct inode* ip, uint bn)` 函数 (这个函数就是获取 inode 中第 bn 个块的块号，如果没有就获取1个Block并返回块号) 和`itrunc(struct inode* ip)` 函数(这个函数释放inode的所有的数据块)

先来看`bmap` ，首先前面的逻辑并不需要更改，这里就是判断`bn` 是否落在direct block，还是落在singly-indirect block。从singly-indirect block开始，就要把indirect block读取进内存，然后再以`uint` 数组形式进行访问，具体细节见代码：

```c
static uint
bmap(struct inode *ip, uint bn)
{
  uint addr, *a;
  struct buf *bp;

  if(bn < NDIRECT){
    if((addr = ip->addrs[bn]) == 0)
      ip->addrs[bn] = addr = balloc(ip->dev);
    return addr;
  }
  bn -= NDIRECT;

  if(bn < NINDIRECT){ // singly-indirect
    // Load indirect block, allocating if necessary.
    if((addr = ip->addrs[NDIRECT]) == 0)
      ip->addrs[NDIRECT] = addr = balloc(ip->dev);
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if((addr = a[bn]) == 0){
      a[bn] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);
    return addr;
  }
  bn -= NINDIRECT；
```

接下来就是我们需要实现的部分了，其实也比较简单：

1. 如果doubly-indirect block不存在，就先分配一个block并读取这个block。
2. 我们要判断`bn` 在doubly-indirect block的位置，我们知道，在doubly-indirect block中每一项都是一个singly-indirect block的位置，也就是我们需要先判断`a[bn/NINDIRECT]` 是否存在，如果不存在，我们先要分配，并打开这个下一层级的block。
3. 然后`bn %= NINDIRECT` 就是`bn` 在`singly-indirect block` 的位置了。如果不存在就分配，并返回这个block number。

```c
if(bn < NINDIRECT * NINDIRECT) { // doubly-indirect
    // Load indirect block, allocating if necessary.
    if((addr = ip->addrs[NDIRECT+1]) == 0)
      ip->addrs[NDIRECT+1] = addr = balloc(ip->dev);
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if((addr = a[bn/NINDIRECT]) == 0){
      a[bn/NINDIRECT] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);
    bn %= NINDIRECT;
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if((addr = a[bn]) == 0){
      a[bn] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);
    return addr;
  }
```



`itrunc` 函数也就比较简单了，就是不断调用`bfree(int dev, uint b)` 函数是否每一个文件占用的块。这里就不展开讲了。

```c
void itrunc(struct inode *ip)
{
  int i, j;
  struct buf *bp;
  uint *a;

  for(i = 0; i < NDIRECT; i++){
    if(ip->addrs[i]){
      bfree(ip->dev, ip->addrs[i]);
      ip->addrs[i] = 0;
    }
  }

  if(ip->addrs[NDIRECT]){
    bp = bread(ip->dev, ip->addrs[NDIRECT]);
    a = (uint*)bp->data;
    for(j = 0; j < NINDIRECT; j++){
      if(a[j])
        bfree(ip->dev, a[j]);
    }
    brelse(bp);
    bfree(ip->dev, ip->addrs[NDIRECT]);
    ip->addrs[NDIRECT] = 0;
  }

  if(ip->addrs[NDIRECT+1]){
    bp = bread(ip->dev, ip->addrs[NDIRECT+1]);
    a = (uint*)bp->data;
    for(j = 0; j < NINDIRECT; j++){
      if(a[j]) {
        struct buf *bp2 = bread(ip->dev, a[j]);
        uint *a2 = (uint*)bp2->data;
        for(int k = 0; k < NINDIRECT; k++){
          if(a2[k])
            bfree(ip->dev, a2[k]);
        }
        brelse(bp2);
        bfree(ip->dev, a[j]);
      }
    }
    brelse(bp);
    bfree(ip->dev, ip->addrs[NDIRECT+1]);
    ip->addrs[NDIRECT + 1] = 0;
  }

  ip->size = 0;
  iupdate(ip);
}
```





## **Symbolic links**

首先区分一下什么是硬链接和软链接。
- 硬链接：原文件和链接文件共用一个 `inode`号，指向同一块磁盘区域。
- 软链接：原文件和链接文件拥有不同的 `inode`号，这是两个不同的文件。它包含了指向另一个文件的路径。软链接相当于一个指针，指向另一个文件或目录。

那么在这个实验中，实现软链接应该新建一个`inode`，然后再这个`inode`中填写原文件的地址，如果找出的内容仍然是一个软链接，则继续递归查找。这里要修改`sys_open` 系统调用。

```c
// creates a new symbolic link at path that refers to file named by target
symlink(char *target, char *path)
```



我们先来看`sys_symlink` 系统调用的实现，这个实现比较简单，就是新创建一个文件`ip = create(path, T_SYMLINK, 0, 0);` 文件类型为软链接，接下来再用`writei` 函数把`target` 文件的路径写入inode文件中。

> `int writei(struct inode ip, int user_src, uint64 src, uint off, uint n)`  这里`user_src` 表示是从用户空间地址中写还是从内核空间中写入。

```c
uint64 sys_symlink(void){
  struct inode *ip;
  char target[MAXPATH], path[MAXPATH];
  if(argstr(0, target, MAXPATH) < 0 || argstr(1, path, MAXPATH) < 0)
    return -1;

  begin_op();

  ip = create(path, T_SYMLINK, 0, 0);
  if(ip == 0){
    end_op();
    return -1;
  }

  // use the first data block to store target path.
  if(writei(ip, 0, (uint64)target, 0, strlen(target)) < 0) {
    end_op();
    return -1;
  }

  iunlockput(ip);

  end_op();
  return 0;
}
```



再完成`sys_symlink` 系统调用后，再来对`sys_open` 函数进行修改。

根据实验的提示，我们需要一个`O_NOFOLLOW` 的标志，在打开软链接文件的时候，是想得到这个软链接文件的真实内容，还是这个软链接文件所指向的文件。

具体代码如下：

```c
if(omode & O_CREATE){
    ip = create(path, T_FILE, 0, 0);
    if(ip == 0){
      end_op();
      return -1;
    }
  } else {
    int symlink_depth = 0;
    while(1) { // recursively follow symlinks
      if((ip = namei(path)) == 0){
        end_op();
        return -1;
      }
      ilock(ip);
      if(ip->type == T_SYMLINK && (omode & O_NOFOLLOW) == 0) {
        if(++symlink_depth > 10) {
          // too many layer of symlinks, might be a loop
          iunlockput(ip);
          end_op();
          return -1;
        }
        if(readi(ip, 0, (uint64)path, 0, MAXPATH) < 0) {
          iunlockput(ip);
          end_op();
          return -1;
        }
        iunlockput(ip);
      } else {
        break;
      }
    }
    if(ip->type == T_DIR && omode != O_RDONLY){
      iunlockput(ip);
      end_op();
      return -1;
    }
  }
```





