# LeetCode周打卡

## 1 两数之和

题目: 给定一个整数数组 nums 和一个目标值 target，请你在该数组中找出和为目标值的那 两个 整数，并返回他们的数组下标。

你可以假设每种输入只会对应一个答案。但是，你不能重复利用这个数组中同样的元素。

>  示例
>
> 给定 nums = [2, 7, 11, 15], target = 9
>
> 因为 nums[0] + nums[1] = 2 + 7 = 9
>
> 所以返回 [0, 1]

思考：这个问题是leetcode第一道题，看起来好像不难，但是也不能掉以轻心。一眼看去，两重遍历就可以解决，但是这种方式时间复杂度是$O(n^2)$ ，一定不是最好的。再想想其他的，用空间换时间？将时间降下来，这里可以采用HashMap，这样将空间复杂度提升到$T(n)$，时间复杂度也降到$O(n)$了。

**题解一**

```java
private static int[] twoSum(int[] nums, int target) {
        int[] result = new int[2];
        for (int i = 0; i < nums.length; i++) {
            for (int j = i+1; j < nums.length; j++) {
                if(nums[i]+nums[j] == target) {
                    result[0] = i;
                    result[1] = j;
                    return result;
                }
            }
        }
        return result;
    }
```

两次for循环时间复杂度$O(n^2)$，空间复杂度$T(1)$。

**题解二**

```
public int[] twoSum(int[] nums, int target) {
        int[] result = new int[2];
        HashMap<Integer, Integer> map = new HashMap();      //空间复杂度T(n)
        for (int i = 0; i < nums.length; i++) {             
            if(map.get(nums[i]) == null) {
                map.put(nums[i], i);
            }
            if(map.get(target - nums[i])!=null && map.get(target - nums[i])!=i) {
                result[0] = i;
                result[1] = map.get(target - nums[i]);
                return result;
            }
        }
        return result;
    }
```

可见，只进行了一个遍历for循环，所以空间复杂度提升到$T(n)$，时间复杂度也降到$O(n)$。

## 2 两数相加

给出两个 非空 的链表用来表示两个非负的整数。其中，它们各自的位数是按照 逆序 的方式存储的，并且它们的每个节点只能存储 一位 数字。

如果，我们将这两个数相加起来，则会返回一个新的链表来表示它们的和。

您可以假设除了数字 0 之外，这两个数都不会以 0 开头。

> **示例：**
>
> ```java
> 输入：(2 -> 4 -> 3) + (5 -> 6 -> 4)
> 输出：7 -> 0 -> 8
> 原因：342 + 465 = 807
> ```

考点：如何操作链表、代码健壮性。 两个数相加有几种边界条件，链表长度不一致，两个数相加大于9进位的情况。

先创建一个result链表，然后用head作为指针，往后移动。

```java
private static ListNode findTwoSum(ListNode l1, ListNode l2) {
        ListNode result = new ListNode(-1);
        ListNode head = result;

        int add = 0;                                //进位的数，等于1的时候表示进位
        while (l1!=null || l2!=null || add>0) { //add>0  是防止，两个链表都遍历完了，还要进位的情况
            int sum = 0;
            if(l1!=null) {
                sum = sum + l1.val;
                l1 = l1.next;
            }
            if(l2!=null) {
                sum = sum + l2.val;
                l2 = l2.next;
            }
            if(add>0) {
                sum = sum + (add--);
            }
            if(sum>9) {
                add++;
                sum = sum%10;                  //取模操作，留下当前为的数，add+1
            }
            head.next = new ListNode(sum);
            head = head.next;
        }
        return result.next;
    }
```