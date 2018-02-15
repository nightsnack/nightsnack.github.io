##         二叉查找树的删除操作

对于二叉查找树的删除操作（这里根据值删除，而非结点）分三种情况：
不过在此之前，我们应该确保根据给定的值找到了要删除的结点，如若没找到该结点
不会执行删除操作！
下面三种情况假设已经找到了要删除的结点。
1. 如果结点为叶子结点（没有左、右子树），此时删除该结点不会玻化树的结构
   直接删除即可，并修改其父结点指向它的引用为null.如下图:
    ![img](http://img.blog.csdn.net/20130506115644277)
2. 如果其结点只包含左子树，或者右子树的话，此时直接删除该结点，并将其左子树
   或者右子树设置为其父结点的左子树或者右子树即可，此操作不会破坏树结构。
     ![img](http://img.blog.csdn.net/20130506115712159)
3. 当结点的左右子树都不空的时候，一般的删除策略是用其右子树的最小数据

（容易找到）代替要删除的结点数据并递归删除该结点（此时为null），因为

  右子树的最小结点不可能有左孩子，所以第二次删除较为容易。

  z的左子树和右子树均不空。找到z的后继y，因为y一定没有左子树，所以可以删除y，
  并让y的父亲节点成为y的右子树的父亲节点，并用y的值代替z的值.如图：

 	![img](http://img.blog.csdn.net/20130506120027521)




```java
 /**
     * 二叉查找树的删除
     */
    public BinaryNode<T> remove(BinaryNode<T> node, T data) {
        if (node == null) {
            return node;
        }
        int res = data.compareTo(node.data);
        if (res == 0) {
            if (node.leftNode != null && node.rightNode != null) {
                node.data = findMin(node.rightNode);
                node.rightNode = remove(node.rightNode, node.data);
            }
            // 如果左子树不为空，则返回左子树；否则返回右子树，右子树为 null，也没问题
            node = node.leftNode != null ? node.leftNode : node.rightNode;
        }
        if (res > 0) {
            node.rightNode = remove(node.rightNode, data);
        } else {
            node.leftNode = remove(node.leftNode, data);
        }
        return node;
    }
```

