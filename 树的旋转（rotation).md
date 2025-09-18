![[Pasted image 20250905182357.png]]



==左右旋转这个操作既可以延长或者缩短一个树的高度，也可以保持搜索树的性质不变（左孩子小于你，右孩子大于你）==
# 左旋转
1. G的右孩子P变成G的父节点
2. P的左孩子变成G的右孩子

![[Screenshot_20250905_182703_tv_danmaku_bilibilihd_HDUnitedBizDetailsActivity.jpg]]![[Screenshot_20250905_182621_tv_danmaku_bilibilihd_HDUnitedBizDetailsActivity.jpg]]


# 右旋转

1. G的左孩子C变成G的父节点
2. C的右孩子变成G的左孩子



# 两种对称操作的一个函数实现

> [!NOTE] 
> 编写过程中要小心空指针



```cpp
#include <iostream>
#include <cassert>

Node* root = nullptr;

enum COLOR { R, B }; // 节点颜色
enum Direct { L, R }; // 孩子方向

struct Node {
    Node* father;
    Node* chil[2]; // [0]=左孩子, [1]=右孩子
    bool color;    // 建议使用更明确的字段名
    int value;
    int size;      // 子树大小（包含自身）

    // 构造函数初始化
    Node(int val) : father(nullptr), chil{ nullptr, nullptr },
        color(R), value(val), size(1) {}
};

/*
 * 执行旋转操作
 * @param p 当前子树根节点
 * @param dir 旋转方向: 0=左旋, 1=右旋
 * @return 旋转后新的子树根节点
 */
    
Node* rotation(Node* p, bool dir) {
    Node* gp = p->father; // 祖父节点
    Node* s = p->chil[!dir]; // 新根节点（旋转后替代p的位置）
    assert(s != nullptr); // 确保新根节点存在

    //首先处理需要转移的子树
    // 处理s的子树（可能为空）
    Node* c = s->chil[dir]; // 需要转移的子节点
    if (c) {
        c->father = p; // 更新父指针
    }

    // 重构父子关系
    p->chil[!dir] = c;      // p接管s的原子节点
    s->chil[dir] = p;       // s接管p作为子节点
    
    // 更新父指针
    p->father = s;          // p的新父节点是s
    s->father = gp;         // s的新父节点是原祖父
    
    // 更新祖父节点的链接
    if (gp) {
        // 判断p原先是gp的左子还是右子
        bool p_dir = (p == gp->chil[Direct::R]);
        gp->chil[p_dir] = s; // gp现在指向s
    } else {
        root = s; // 更新整棵树的根节点
    }

    // 更新子树大小（关键补充）
    s->size = p->size;                 // s继承原p的子树大小
    //避免空指针，因此不采用p->size = p->chil[dir]->size + p->chil[!dir]->size + 1
    p->size = 1;                       // 重置p的计数器（自身）
    if (p->chil[dir]) p->size += p->chil[dir]->size;   // 添加保留的子节点大小
    if (p->chil[!dir]) p->size += p->chil[!dir]->size; // 添加转移来的子节点大小

    return s; // 返回新的子树根节点
}

// 辅助函数：获取子树大小（处理空指针）
size_t size(Node* node) {
    return node ? node->size : 0;
}

int main() {
    // 测试用例
    Node* n1 = new Node(10);
    Node* n2 = new Node(20);
    Node* n3 = new Node(30);

    // 构建初始树结构
    root = n2;
    n2->chil[Direct::L] = n1;
    n2->chil[Direct::R] = n3;
    n1->father = n2;
    n3->father = n2;

    // 更新子树大小
    n2->size = 3;

    // 执行左旋测试
    Node* new_root = rotation(n2, Direct::L);
    std::cout << "Rotation test passed!" << std::endl;

    delete n1;
    delete n2;
    delete n3;
    return 0;
}

```

