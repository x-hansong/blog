title: Hiho 1289 403 Forbidden（微软编程题）
date: 2016-04-09 16:10:40
tags: [字典树, Trie]
categories: 解题报告
---

## 题意

    allow 1.2.3.4/30
    deny 1.1.1.1
    allow 127.0.0.1
    allow 123.234.12.23/3
    deny 0.0.0.0/0

输入 ip 按顺序匹配规则，优先匹配前面的规则，如果没有规则可以匹配则视为合法。注意：掩码为 0 的时候表示匹配所有 ip。

## 思路
一开始做的时候用遍历匹配的方法，直接超时了。后来才想到用字典树的方法来做，这道题本质上是一道字典树变形。

### 数据结构

    class Node {
            byte flag;//0代表普通节点，1代表允许规则终点，2代表禁止规则终点
            int seq;//规则顺序
            Node[] next = new Node[2];

            Node(byte flag){
                this.flag = flag;
        }
    }

### 建树
1. seq 自增，记录规则的顺序
2. 解析掩码：如果输入的 ip 没有 mask，则默认 `mask = 32`
3. 把ip地址转化为二进制形式
4. 插入节点：如果该节点不存在，则新建；否则沿着节点往下走
5. 设置节点的 flag 和 seq：后面的规则不能覆盖前面的规则，所以要查看当前节点是否为某规则的终点。

代码如下：

    public void insert(String ip, byte flag) {
        seq++;
        int mask = 32;
        int index = ip.indexOf('/');
        //检测是否有掩码
        if (index != -1) {
            mask = Integer.parseInt(ip.substring(index + 1));
        } else {
            index = ip.length();
        }
         //把ip地址转化为二进制形式
        String binary = toBinary(ip.substring(0, index));

        char[] binarys = binary.toCharArray();

        Node node = root;

        for (int i = 0; i < mask; i++) {
            int pos = binarys[i] - '0';
            if (node.next[pos] == null){
                node.next[pos] = new Node((byte) 0);
            }
            node = node.next[pos];
        }
        //后面的规则不能覆盖前面的
        if (node.flag == 0){
            node.flag = flag;
        }
        if (node.seq == 0) {
            node.seq = seq;
        }
    }

### 匹配
1. 把ip地址转化为二进制形式
2. 遍历字典树
3. 如果当前节点为规则的终点，则需要比较该规则的顺序，seq 小的优先匹配
4. 遍历完字典树之后，isAllow 的值就是该 ip 最先匹配到的规则所规定的权限

代码如下：

    public boolean isAllow(String ip){
        String binary = toBinary(ip);
        char[] binarys = binary.toCharArray();

        Node node = root;
        int seq = Integer.MAX_VALUE;
        boolean isAllow = true;
        int i = 0;
        int pos = 0;
        while (node != null){
            //字典树最多会有33个节点，而ip的二进制最多只有32位
            //另一种避免判断的方法是在ip的二进制后面补一个0
            if (i < binarys.length){
                pos = binarys[i] - '0';
            }
            if (node.flag == 1){
                if (node.seq < seq){
                    isAllow = true;
                    seq = node.seq;
                }
            } else if (node.flag == 2){
                if (node.seq < seq){
                    isAllow = false;
                    seq = node.seq;
                }
            } else {
                //flag=0说明是普通节点，直接跳过即可。
            }
            node = node.next[pos];
            i++;
        }
        return isAllow;
    }


_[完整代码][1]_


  [1]: https://github.com/x-hansong/JavaCodes/blob/master/src/main/java/com/contest/Forbidden.java