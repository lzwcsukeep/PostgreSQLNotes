# Mask

mask 就是你通过 它可以只关注部分 bit 位。

不关注的部分用 0 过滤，关注的用1保持原状。

比如：

```c
#define BKPBLOCK_FLAG_MASK	0xF0
#define BKPBLOCK_FORK_MASK	0x0F
```

`BKPBLOCK_FLAG_MASK` 就只关注高4位，用 `BKPBLOCK_FLAG_MASK` 去做按位与操作，就可以把高四位保持原状，低四位清空。

同理，如何检测某一个bit位?

也是把关注的bit 位用 1 保持原状，其余的全都用0 清空，然后进行按位与操作。结果不为0, 该 bit 为 1 ; 结果为0，该bit 为 0。

---

如果检测关注的 bit 位有多个，那可以使用相等性判断替代是否为0的判断。

