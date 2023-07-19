# Lec13 Sleep & Wakeup

### 不允许持有当前进程锁以外的锁

否则在调用swatch时会发生死锁。

![image-20230716191903396](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230716191903396.png)

### lost wakeups



### sleep几乎都和while一起出现

防止并发sleep，然后多个进程一wakeup，丢失了wakeup；
