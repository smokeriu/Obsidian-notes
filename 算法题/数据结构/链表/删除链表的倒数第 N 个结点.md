
# 思路

主要有三种思路：
## 获取长度

直接遍历一遍链表，获取其长度，得到实际要移除的节点是第几个，再实施移除。

```java
class Solution {
    public ListNode removeNthFromEnd(ListNode head, int n) {
        ListNode dh = new ListNode(-1, head);

        int len = 0;
        ListNode cur = head;
        while (cur != null) {
          len++;
          cur = cur.next;
        }

        int move = len - n;

        ListNode pre = dh;
        cur = head;
        for (int i = 0; i < move; i++) {
          pre = cur;
          cur = cur.next;
        }

        pre.next = cur.next;

        return dh.next;
  }
}
```
##  利用栈

栈具有先进后出的逻辑，我们可以将链表中的元素全部入栈，遍历完后，在依次弹出N个节点即可。



# 题目

- [19. 删除链表的倒数第 N 个结点](https://leetcode.cn/problems/remove-nth-node-from-end-of-list/)