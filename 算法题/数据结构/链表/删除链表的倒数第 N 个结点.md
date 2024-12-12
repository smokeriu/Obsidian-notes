
# 思路

主要有三种思路， 建议考虑双指针：
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
## 利用栈

栈具有先进后出的逻辑，我们可以将链表中的元素全部入栈，遍历完后，在依次弹出N个节点即可。

## 双指针

这里利用数组的[[双指针]]思路，设计快慢指针，快指针先向后移动n位。此时快慢指针相隔距离**正好是n**，随后两个指针一起移动，当快指针抵达尾部时，慢指针正好指向待删除的元素。

需要注意的时，第二次一起移动的结束需要依赖于慢指针的启示位置，一般将慢指针初始化到虚拟头节点，这样当快指针停止移动时，慢指针正好指向待删除元素的前一个元素，还能够避免快指针第二次移动开始时为空的情况。

```java
class Solution {
    public ListNode removeNthFromEnd(ListNode head, int n) {
        ListNode dh = new ListNode(-1, head);
    
        ListNode r = head;
        ListNode l = dh;
    
        for (int i = 0; i < n; i++) {
          r = r.next;
        }
    
        while (r != null) {
          l = l.next;
          r = r.next;
        }
    
        l.next = l.next.next;
    
        return dh.next;
    }
}
```



# 题目

- [19. 删除链表的倒数第 N 个结点](https://leetcode.cn/problems/remove-nth-node-from-end-of-list/)