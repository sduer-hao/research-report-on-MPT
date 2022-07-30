# research-report-on-MPT

MPT树定义
一种经过改良的、融合了默克尔树和前缀树两种树结构优点的数据结构，以太坊中，MPT是一个非常重要的数据结构，在以太坊中，帐户的交易信息、状态以及相应的状态变更，还有相关的交易信息等都使用MPT来进行管理，其是整个数据存储的重要一环。交易树，收据树，状态树都是采用的MPT结构。

MPT树的作用是什么？
1.存储任意长度的key-value键值对数据；
2.提供了一种快速计算所维护数据集哈希标识的机制；
3.提供了快速状态回滚的机制；
4.提供了一种称为默克尔证明的证明方法，进行轻节点的扩展，实现简单支付验证；

MPT树中的节点包括空节点、叶子节点、扩展节点和分支节点:

空节点，简单的表示空，在代码中是一个空串。

叶子节点（leaf），表示为[key,value]的一个键值对，其中key是key的一种特殊十六进制编码，value是value的RLP编码。

扩展节点（extension），也是[key，value]的一个键值对，但是这里的value是其他节点的hash值，这个hash可以被用来查询数据库中的节点。也就是说通过hash链接到其他节点。

分支节点（branch），因为MPT树中的key被编码成一种特殊的16进制的表示，再加上最后的value，所以分支节点是一个长度为17的list，前16个元素对应着key中的16个可能的十六进制字符，如果有一个[key,value]对在这个分支节点终止，最后一个元素代表一个值，即分支节点既可以搜索路径的终止也可以是路径的中间节点。


MPT树中另外一个重要的概念是一个特殊的十六进制前缀(hex-prefix, HP)编码，用来对key进行编码。因为字母表是16进制的，所以每个节点可能有16个孩子。因为有两种[key,value]节点(叶节点和扩展节点)，引进一种特殊的终止符标识来标识key所对应的是值是真实的值，还是其他节点的hash。如果终止符标记被打开，那么key对应的是叶节点，对应的值是真实的value。如果终止符标记被关闭，那么值就是用于在数据块中查询对应的节点的hash。无论key奇数长度还是偶数长度，HP都可以对其进行编码。最后我们注意到一个单独的hex字符或者4bit二进制数字，即一个nibble。

HP编码很简单。一个nibble被加到key前（下图中的prefix），对终止符的状态和奇偶性进行编码。最低位表示奇偶性，第二低位编码终止符状态。如果key是偶数长度，那么加上另外一个nibble，值为0来保持整体的偶特性。

HP编码
HP-编码：特殊的十六进制前缀编码

引入：对nibble和节点奇偶性进行编码

Nibble是什么？
引进一种特殊的终止符标识来标识key所对应的是值是真实的值，还是其他节点的hash。如果终止符标记被打开，那么key对应的是叶节点，对应的值是真实的value。如果终止符标记被关闭，那么值就是用于在数据块中查询对应的节点的hash。
为什么需要对节点奇偶性进行编码？
数据最小的表示单位为一位16进制，如1、a等，但在编程实现中，数据的最小表示单位往往是byte（8bit，2位16进制数），这样在用byte来表示一串奇数长度的16进制串时会出现问题。
例如"5b3"和"5b30"，直接转成byte都是5b30。

举例:
对"0x5b3ed"编码（奇数位）
"0x5b3ed" = "0005 1011 0003 1110 1101"t=0 时, "0001"+"0005 1011 0003 1110 1101"->"00010005 10110003 11101101"->"0x15b3ed"
t !=0时 "0011"+"0005 1011 0003 1110 1101"->"00110005 10110003 11101101"->"0x35b3ed“

对"0x5b3e"编码（偶数位）
"0x5b3e" = "0005 1011 0003 1110"t=0 时, "0000"+"0005 1011 0003 1110 1101"->"00000005 10110003 11101101"->"0x005b3e"
t !=0时 "0010"+"0005 1011 0003 1110 1101"->"00100005 10110003 11101101"->"0x205b3e"

这里的t就是是否结束的标志位
最低位表示奇偶性，第二低位编码终止符状态。
最低位为0的时候表示偶数位置，反之奇数。
第二低位为1的时候表示结束，反之不结束。

MPT树的操作
下面从MPT树的更新，删除和查找过程来说明MPT树的操作。

1 更新

函数_update_and_delete_storage(self, node, key, value)

i. 如果node是空节点，直接返回[pack_nibbles(with_terminator(key)), value]，即对key加上终止符，然后进行HP编码。



ii. 如果node是分支节点，如果key为空，则说明更新的是分支节点的value，直接将node[-1]设置成value就行了。如果key不为空，则递归更新以key[0]位置为根的子树，即沿着key往下找，即调用_update_and_delete_storage(self._decode_to_node(node[key[0]]),key[1:], value)。




iii. 如果node是kv节点（叶子节点或者扩展节点），调用_update_kv_node(self, node, key, value)，见步骤iv

iv. curr_key是node的key，找到curr_key和key的最长公共前缀，长度为prefix_length。Key剩余的部分为remain_key，curr_key剩余的部分为remain_curr_key。

a)       如果remain_key==[]== remain_curr_key，即key和curr_key相等，那么如果node是叶子节点，直接返回[node[0], value]。如果node是扩展节点，那么递归更新node所链接的子节点，即调用_update_and_delete_storage(self._decode_to_node(node[1]),remain_key, value)



b)      如果remain_curr_key == []，即curr_key是key的一部分。如果node是扩展节点，递归更新node所链接的子节点，即调用_update_and_delete_storage(self._decode_to_node(node[1]),remain_key, value)；如果node是叶子节点，那么创建一个分支节点，分支节点的value是当前node的value，分支节点的remain_key[0]位置指向一个叶子节点，这个叶子节点是[pack_nibbles(with_terminator(remain_key[1:])),value]



c)       否则，创建一个分支节点。如果curr_key只剩下了一个字符，并且node是扩展节点，那么这个分支节点的remain_curr_key[0]的分支是node[1]，即存储node的value。否则，这个分支节点的remain_curr_key[0]的分支指向一个新的节点，这个新的节点的key是remain_curr_key[1:]的HP编码，value是node[1]。如果remain_key为空，那么新的分支节点的value是要参数中的value，否则，新的分支节点的remain_key[0]的分支指向一个新的节点，这个新的节点是[pack_nibbles(with_terminator(remain_key[1:])),value]

d)      如果key和curr_key有公共部分，为公共部分创建一个扩展节点，此扩展节点的value链接到上面步骤创建的新节点，返回这个扩展节点；否则直接返回上面步骤创建的新节点


v. 删除老的node，返回新的node

 

l  删除

删除的过程和更新的过程类似，而且很简单，函数名：_delete_and_delete_storage(self, key)

i. 如果node为空节点，直接返回空节点

ii. 如果node为分支节点。如果key为空，表示删除分支节点的值，直接另node[-1]=‘’, 返回node的正规化的结果。如果key不为空，递归查找node的子节点，然后删除对应的value，即调用self._delete_and_delete_storage(self._decode_to_node(node[key[0]]),key[1:])。返回新节点

iii. 如果node为kv节点，curr_key是当前node的key。

a) 如果key不是以curr_key开头，说明key不在node为根的子树内，直接返回node。

b) 否则，如果node是叶节点，返回BLANK_NODE if key == curr_key else node。

c)如果node是扩展节点，递归删除node的子节点，即调用_delete_and_delete_storage(self._decode_to_node(node[1]),key[len(curr_key):])。如果新的子节点和node[-1]相等直接返回node。否则，如果新的子节点是kv节点，将curr_key与新子节点的可以串联当做key，新子节点的value当做vlaue，返回。如果新子节点是branch节点，node的value指向这个新子节点，返回。

l  查找

查找操作更简单，是一个递归查找的过程函数名为：_get(self, node, key)

i. 如果node是空节点，返回空节点

ii. 如果node是分支节点，如果key为空，返回分支节点的value；否则递归查找node的子节点，即调用_get(self._decode_to_node(node[key[0]]), key[1:])

iii. 如果node是叶子节点，返回node[1] if key == curr_key else ‘’

iv. 如果node是扩展节点，如果key以curr_key开头，递归查找node的子节点，即调用_get(self._decode_to_node(node[1]),key[len(curr_key):])；否则，说明key不在以node为根的子树里，返回空

