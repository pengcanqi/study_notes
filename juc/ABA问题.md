# ABA问题

## 什么是ABA问题

```
ABA问题发生在多线程环境中，当某线程连续读取同一块内存地址两次，两次得到的值一样，它简单地认为“此内存地址的值并没有被修改过”，然而，同时可能存在另一个线程在这两次读取之间把这个内存地址的值从A修改成了B又修改回了A，这时还简单地认为“没有修改过”显然是错误的。
```

## ABA问题带来的危害

### 举例1：

```
就是首先小明来取钱，需要把账户从100变成50，但是线程的cpu执行权被抢走了，先执行的是朋友的汇款，把账户从100变成150，然后骗子开始盗刷，从150变成了100元，这个时间取款的线程抢到了cpu执行权，成功的把账户从100变成了50，对于小明来说，因为不知道取钱过程发生的事情，会认为是正常现象，毕竟原来我的账户就是100元，取款成功变成50，很正常，但是实际是损失了50元，小明并不知道。所以ABA问题可能带来的危害是因为不知道值改变的过程带来的，所以我觉得如果你需要关注每一个值的来历，那么cas慎用，如果不关注过程，就关注结果，那么aba问题不会给你带来困扰。
```

### 举例2：

```
首先说结论，值的问题造不成什么不良影响，其他答案举的好多例子都翻车了。我举一个真正能造成ABA的问题。就是期望那里是一个有指针的节点时，没有版本号的cas会造成ABA后果很严重的问题，完全错乱。你可能会问了，谁设计业务时候放指针的？我咋没碰见过这么设计的？确实业务里一般不这么设计～但是JDK源码里juc那个包里面 锁lock相关的，阻塞队列实现的底层，同步器那里，就是用原子类CAS操作指针，当然是代版本号的CAS～是的，真他妈的去操作指针了～所以我们就有版本号的CAS了～
```

```java
public class ABATest {

    static class Stack {
        // 将top放在原子类中
        private AtomicReference<Node> top = new AtomicReference<>();
        // 栈中节点信息
        static class Node {
            int value;
            Node next;

            public Node(int value) {
                this.value = value;
            }
        }
        // 出栈操作
        public Node pop() {
            for (;;) {
                // 获取栈顶节点
                Node t = top.get();
                if (t == null) {
                    return null;
                }
                // 栈顶下一个节点
                Node next = t.next;
                // CAS更新top指向其next节点
                if (top.compareAndSet(t, next)) {
                    // 把栈顶元素弹出，应该把next清空防止外面直接操作栈
                    t.next = null;
                    return t;
                }
            }
        }
        // 入栈操作
        public void push(Node node) {
            for (;;) {
                // 获取栈顶节点
                Node next = top.get();
                // 设置栈顶节点为新节点的next节点
                node.next = next;
                // CAS更新top指向新节点
                if (top.compareAndSet(next, node)) {
                    return;
                }
            }
        }
    }
}
```

测试代码如下：

```java
private static void testStack() {
    // 初始化栈为 top->1->2->3
    Stack stack = new Stack();
    stack.push(new Stack.Node(3));
    stack.push(new Stack.Node(2));
    stack.push(new Stack.Node(1));

    new Thread(()->{
        // 线程1出栈一个元素
        stack.pop();
    }).start();

    new Thread(()->{
        // 线程2出栈两个元素
        Stack.Node A = stack.pop();
        Stack.Node B = stack.pop();
        // 线程2又把A入栈了
        stack.push(A);
    }).start();
}

public static void main(String[] args) {
    testStack();
}
```

假如，我们初始化栈结构为 top->1->2->3，然后有两个线程分别做如下操作：

（1）线程1执行pop()出栈操作，但是执行到`if (top.compareAndSet(t, next)) {`这行之前暂停了，所以此时节点1并未出栈；

（2）线程2执行pop()出栈操作弹出节点1，此时栈变为 top->2->3；

（3）线程2执行pop()出栈操作弹出节点2，此时栈变为 top->3；

（4）线程2执行push()入栈操作添加节点1，此时栈变为 top->1->3；

（5）线程1恢复执行，比较节点1的引用并没有改变，执行CAS成功，此时栈变为 top->2；

What？点解变成 top->2 了？不是应该变成 top->3 吗？

那是因为线程1在第一步保存的next是节点2，所以它执行CAS成功后top节点就指向了节点2了。

## 如何解决ABA问题

AtomicStampedReference

AtomicMarkableReference

