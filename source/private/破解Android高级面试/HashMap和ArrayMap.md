|      | HashMap | ArrayMap |
| :--: | :-----: | :------: |
| 查询 |  O(1)   | O(logn)  |
| 插入 |  O(1)   | O(logn)  |
| 扩容 |   2倍   |  1.5倍   |
| 缩容 |   无    |  0.5倍   |
|      |         |          |

元素多（大于1000个）使用HashMap

频繁增删使用HashMap

Key是整形使用SparseArray（避免拆装箱），否则使用ArrayMap

