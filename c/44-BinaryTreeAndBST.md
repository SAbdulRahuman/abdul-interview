# Binary Tree & BST

---

## Binary Tree

```c
typedef struct TreeNode {
    int data;
    struct TreeNode *left, *right;
} TreeNode;

TreeNode *new_node(int val) {
    TreeNode *n = malloc(sizeof(TreeNode));
    n->data = val;
    n->left = n->right = NULL;
    return n;
}
```

## Traversals

```c
// Inorder (Left, Root, Right) – gives sorted order for BST
void inorder(TreeNode *root) {
    if (!root) return;
    inorder(root->left);
    printf("%d ", root->data);
    inorder(root->right);
}

// Preorder (Root, Left, Right)
void preorder(TreeNode *root) {
    if (!root) return;
    printf("%d ", root->data);
    preorder(root->left);
    preorder(root->right);
}

// Postorder (Left, Right, Root)
void postorder(TreeNode *root) {
    if (!root) return;
    postorder(root->left);
    postorder(root->right);
    printf("%d ", root->data);
}
```

## BST Operations

```c
// Insert
TreeNode *bst_insert(TreeNode *root, int val) {
    if (!root) return new_node(val);
    if (val < root->data)      root->left  = bst_insert(root->left, val);
    else if (val > root->data) root->right = bst_insert(root->right, val);
    return root;
}

// Search
TreeNode *bst_search(TreeNode *root, int val) {
    if (!root || root->data == val) return root;
    if (val < root->data) return bst_search(root->left, val);
    return bst_search(root->right, val);
}

// Find minimum
TreeNode *find_min(TreeNode *root) {
    while (root->left) root = root->left;
    return root;
}
```

---

## Interview Q&A

**Q: What is the time complexity of BST operations?**

```
Average case: O(log n) for search, insert, delete
Worst case: O(n) – when tree is skewed (like a linked list)
Balanced BSTs (AVL, Red-Black): guarantee O(log n) worst case
```

**Q: How to free an entire tree?**

```c
void free_tree(TreeNode *root) {
    if (!root) return;
    free_tree(root->left);
    free_tree(root->right);
    free(root);  // postorder: free children first
}
```

---
