---
title: 二叉搜索树Java实现
date: 2018-10-23 20:16:13
categories: 数据结构和算法
tags: 算法
---

二叉搜索树特性：
- 具有二叉树的所有特性。
- 左节点的值永远小于根结点。
- 有节点的值永远大于根结点。
- 具有较高的查找效率。
- 插入效率也高，比链表要高。

---
<!--more-->
二叉搜索树的Java代码实现：
- insert方法：插入一个节点
- getMin方法：获取最小节点
- getMax方法：获取最大节点
- search方法：查找一个节点
- delete方法：删除一个节点
- deleteMax方法：删除最小节点

```java
public class BinarySearchTree<E extends Comparable<E>> {
    private Node<E> root;

    static class Node<E> {
        E item;
        Node<E> leftChild;
        Node<E> rightChild;

        public Node(E item) {
            this.item = item;
        }
    }

    /**
     * 插入
     *
     * @param e
     */
    public void insert(E e) {
        root = insertNode(root, e);
    }

    /**
     * 插入节点
     *
     * @param node
     * @param e
     * @return
     */
    private Node<E> insertNode(Node<E> node, E e) {
        if (node == null) {
            return new Node<>(e);
        }
        int cmp = e.compareTo(node.item);
        if (cmp < 0) {
            //要插入的节点值比当前节点值小，则向当前节点的左节点递归插入，插入后的值赋给当前节点的左节点
            node.leftChild = insertNode(node.leftChild, e);
        } else if (cmp > 0) {
            //要插入的节点值比当前节点值大，则向当前节点的右节点递归插入，插入后的值赋给当前节点的右节点
            node.rightChild = insertNode(node.rightChild, e);
        }
        return node;
    }

    /**
     * 获取整个二叉树的最小节点
     *
     * @return
     */
    public Node<E> getMin() {
        return getMin(root);
    }

    /**
     * 获取以指定节点为根结点的子树中的最小节点
     * 由二叉搜索树的特性可知道最小节点一定是最左的一个节点，
     * 所以一直找最左的节点
     *
     * @param node
     * @return
     */
    public Node<E> getMin(Node<E> node) {
        //递归实现
        if (node.leftChild == null) {
            return node;
        }
        return getMin(node.leftChild);
        //遍历实现
        /*Node<E> min = node;
        while (min.leftChild != null) {
            min = min.leftChild;
        }
        return min;*/
    }

    /**
     * 获取整个二叉树的最大节点
     *
     * @return
     */
    public Node<E> getMax() {
        return getMax(root);
    }

    /**
     * 获取以指定节点为根结点的子树中的最大节点
     * 由二叉搜索树的特性可知道最小节点一定是最右的一个节点，
     * 所以一直找最右的节点
     *
     * @param node
     * @return
     */
    public Node<E> getMax(Node<E> node) {
        //递归实现
        if (node.rightChild == null) {
            return node;
        }
        return getMax(node.rightChild);
        //遍历实现
        /*Node<E> max = node;
        while (max.rightChild != null) {
            max = max.rightChild;
        }
        return max;*/
    }

    public Node<E> search(E e) {
        return search(root, e);
    }

    /**
     * 查找某个节点
     *
     * @param node
     * @param e
     * @return
     */
    public Node<E> search(Node<E> node, E e) {
        //遍历实现
        int count = 0;
        Node<E> current = node;
        while (current != null) {
            count++;
            int cmp = current.item.compareTo(e);
            if (cmp > 0) {
                current = current.leftChild;
            } else if (cmp < 0) {
                current = current.rightChild;
            } else if (cmp == 0) {
                System.out.println("查找次数为：" + count);
                return current;
            }
        }
        return null;
        //递归实现
        /*if (node == null) {
            return null;
        }
        int cmp = node.item.compareTo(e);
        if (cmp > 0) {
            return search(node.leftChild, e);
        } else if (cmp < 0) {
            return search(node.rightChild, e);
        } else {
            return node;
        }*/
    }

    /**
     * 删除值为e的节点
     *
     * @param key
     */
    public void delete(E key) {
        root = delete(root, key);
    }


    /**
     * 删除节点
     *
     * @param node
     * @param key
     * @return
     */
    public Node<E> delete(Node<E> node, E key) {
        if (node == null) {
            return null;
        } else {
            int cmp = key.compareTo(node.item);
            if (cmp > 0) {
                //判断删除节点的值比当前节点大，则向当前节点的右节点递归删除
                node.rightChild = delete(node.rightChild, key);
            } else if (cmp < 0) {
                node.leftChild = delete(node.leftChild, key);
            } else {
                if (node.rightChild == null) {
                    return node.leftChild;
                }
                if (node.leftChild == null) {
                    return node.rightChild;
                }
                Node<E> t = node;
                node = getMin(t.rightChild);
                node.rightChild = deleteMin(t.rightChild);
                node.leftChild = t.leftChild;
            }
        }
        return node;
    }

    /**
     * 删除树中最小节点
     */
    public void deleteMin() {
        root = deleteMin(root);
    }

    /**
     * 删除最小节点
     *
     * @param node
     * @return
     */
    public Node<E> deleteMin(Node<E> node) {
        //如果当前节点左节点为空，则当前节点为最小节点，将当前节点的右节点返回赋值给上一节点的左节点，则表示删除了改节点。
        if (node.leftChild == null) {
            return node.rightChild;
        }
        //递归删除当前节点的左节点，并将返回值赋给左节点
        node.leftChild = deleteMin(node.leftChild);
        return node;
    }

    /**
     * 打印节点值
     *
     * @param node
     */
    private void printNode(Node<E> node) {
        System.out.println(node.item);
    }

        /**
     * 先序遍历
     */
    public void firstOrderTraversal() {
        firstOrderTraversal(root);
    }


    /**
     * 先序遍历(递归实现)
     *
     * @param node
     */
    public void firstOrderTraversal(Node<E> node) {
        if (node == null) {
            return;
        }
        printNode(node);
        firstOrderTraversal(node.leftChild);
        firstOrderTraversal(node.rightChild);
    }

    /**
     * 先序遍历(非递归实现)
     *
     * @param node
     */
    public void firstOrderTraversal2(Node<E> node) {
        if (node == null) {
            return;
        }
        Stack<Node<E>> stack = new Stack<>();
        stack.push(node);

        while (!stack.isEmpty()) {
            Node<E> pop = stack.pop();
            if (pop != null) {
                printNode(pop);
                stack.push(pop.rightChild);
                stack.push(pop.leftChild);
            }
        }

    }


    /**
     * 中序遍历
     */
    public void inOrderTraversal() {
        inOrderTraversal2(root);
    }


    /**
     * 中序遍历
     *
     * @param node
     */
    public void inOrderTraversal(Node<E> node) {
        if (node == null) {
            return;
        }
        inOrderTraversal(node.leftChild);
        printNode(node);
        inOrderTraversal(node.rightChild);

    }

    /**
     * 中序遍历(非递归实现)
     *
     * @param node
     */
    public void inOrderTraversal2(Node<E> node) {
        if (node == null) {
            return;
        }
        Stack<Node<E>> stack = new Stack<>();

        while (node != null || !stack.isEmpty()) {
            if (node != null) {
                stack.push(node);
                node = node.leftChild;
            } else {
                Node<E> pop = stack.pop();
                printNode(pop);
                node = pop.rightChild;
            }
        }

    }

    /**
     * 后序遍历
     */
    public void postOrderTraversal() {
        postOrderTraversal2(root);
    }


    /**
     * 后序遍历
     *
     * @param node
     */
    public void postOrderTraversal(Node<E> node) {
        if (node == null) {
            return;
        }
        postOrderTraversal(node.leftChild);
        postOrderTraversal(node.rightChild);
        printNode(node);
    }

    /**
     * 主要思想：首先遍历root根节点的所有左结点，并依次入栈。对出栈的元素，如果没有右子树或者虽然有右子树
     * 但右子树已完成遍历，即可完成出栈；否则，再次入栈，并把右子树入栈，遍历右子树的所有左子树。
     *
     * @param node
     */
    public void postOrderTraversal2(Node<E> node) {
        if (node == null) {
            return;
        }
        Stack<Node<E>> stack = new Stack<>();
        Node<E> prePop = null;
        while (node != null || !stack.isEmpty()) {
            if (node != null) {
                stack.push(node);
                node = node.leftChild;
            } else {
                Node<E> pop = stack.pop();
                if (pop.rightChild == null || pop.rightChild == prePop) {
                    printNode(pop);
                    prePop = pop;
                    node = null;
                } else {
                    stack.push(pop);
                    stack.push(pop.rightChild);
                    node = pop.rightChild.leftChild;
                }
            }
        }
    }

    /**
     * 层序遍历
     */
    public void levelOrderTraversal() {
        levelOrderTraversal(root);
    }

    /**
     * 层序遍历
     *
     * @param node
     */
    public void levelOrderTraversal(Node<E> node) {
        if (node == null) {
            return;
        }
        LinkedList<Node<E>> queue = new LinkedList<>();
        queue.push(node);
        while (!queue.isEmpty()) {
            Node<E> pop = queue.poll();
            if (pop != null) {
                printNode(pop);
                queue.add(pop.leftChild);
                queue.add(pop.rightChild);
            }

        }
    }

}

```

