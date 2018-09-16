# ����Reentranlock��AQS����

- **AbstractQueuedSynchronizer���������**
- **Reentranlock�ж�AQS��ʵ��**
- **��Lock()�ķ���**
- **��UnLock()�ķ���**

-------------------
#### AbstractQueuedSynchronizer���������

 1. AQS�ڲ�����һ��Node�࣬���������Node������ͬ�����У��ٷ�ע�Ͷ�ͬ�����е�����������    +------+  prev
    +-----+       +-----+  head |      | <---- |     | <---- |     |  tail    +------+       +-----+       +-----+
![AQS��һ���ڲ���Node](https://github.com/Jason194113/JavaSeLearn/blob/master/Screenshots/Node.JPG)
 2. **AQS�ṩ��5����������д�ķ�������Թ������Ͷ�ռ����ʵ��**
       tryAcquire(int arg)
       tryAcquireShared(int arg)
       tryRelease(int arg)
       tryReleaseShared(int arg)
       isHeldExclusively()
      
 3. **AQS�е�ģ�巽��**
     **��Щģ�巽���������ָ���������������д�ķ�����**
     acquire(int arg)
     acquireInterruptibly(int arg)
     ��������д��tryAcquire(int args)
     
     acquireShared(int arg)
     acquireSharedInterruptibly(int arg)
     ������д��tryAcquireShared(int args)
    
     releaseShared(int arg)
     ������д��tryReleaseShared(int arg)
     release(int arg)
     ������д��tryRelease(int arg)
     
 4. state
   state��Ϊͬ�������������ͬ�������в�ͬ��ʵ�ַ�ʽ


  
-------------------
#### Reentranlock�ж�AQS��ʵ��
�����ǻ���Reentranlock�ķǹ�ƽ����AQS��������������������Reentranlock����λ���AQS���й���ġ�

 **1. Syc**
    **Syc��ΪReentranlock���ڲ���ʵ���˼̳���AQS������д��         tryRelease��tryAcquire������ͬʱ������lock()���󷽷���������ľ������ȥʵ��**
    
    `protected final boolean tryRelease(int releases) {
            int c = getState() - releases;  /**Reentranlockʹ��state0��1���ֱ����ǰ���Ƿ�ռ��*/
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
         protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }` 
        
 2. NonfairSync
     NonfairSync�̳���Sync���������Ķ�������������Ƿǹ�ƽ����ʵ�֣�NonfairSync��д��AQS�е�tryAcquire������Sync�е�lock()������
 
```
         final void lock() {
        /** ֮ǰ�ᵽ��State��������ǰ����ռ�����
            ����ʹ��CAS�������Ի�ȡ��*/
              setExclusiveOwnerThread(Thread.currentThread());
              /**��ȡ�ɹ��������ռ���̸߳���Ϊ��ǰ�߳�*/
            else
                acquire(1);
                /**����AQS�е�acquire����*/
        }

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
            /**������Sync�еĶ����nonfairTryAcquire����*/
        }  
```


-------------------
## Lock()����
**֮ǰ���������ǹ�ƽ����lock()ʵ��Ϊ����CAS���Ի�ȡ�����ɹ���ռ������ʧ����ִ��AQS�ж����acquire()����.**

 **- acquire()**
`
 public final void acquire(int arg) {  /**arg=1*/
        if (!tryAcquire(arg) &&   
        /**֮ǰ�ᵽ��acquireΪģ�巽�����������ִ��������д��tryacquire()����,����ľ���ʵ��ΪSync���е�nonfairTryAcquire����*/
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
`

```
final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();/**�õ�ͬ��������ֵ*/
            if (c == 0) {/**�����0����ʾ��ǰû���߳�ռ����*/
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                  /**����CAS���Ի�ȡ�����ɹ��򷵻�true*/
                }
            }
            /**���ظ���ȡ�����ж�*/
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

 **- addWaiter()**
   

```
    private Node addWaiter(Node mode) {
    //����ǰ�̰߳�װ��һ��Node�ڵ㣬Node��ά����һ��waitstatus����
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        //��ʼ��ͬ������
        enq(node);
        return node;
    }


        private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                //������һ��ͷ�ڵ�
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                  //����ǰ�̵߳Ľڵ�β���뵽ͬ������
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

 **-acquireQueued()**
   **���������AQS����Ҫ�ķ����������˽ڵ����ͬ�����к��һϵ�в���**
   

```
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
            //��øղ����ǰ�̽ڵ�
                final Node p = node.predecessor();
                //1.���ǰ�̽ڵ���head����ô�ͳ����ٴλ�ȡ��
                //2.���ǰ�̽ڵ㲻��head�����߳��Ի�ȡ��ʧ�� 
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                //�ж��ڻ�ȡ��ʧ���Ժ��ܷ��������
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

   private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        //���ǰ�̽ڵ�ĵȴ�״̬���ڵ�Ĭ�ϵ�waitstatusΪ0
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
           
            return true;
        if (ws > 0) {
           
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
          
             //����CAS������ǰ�̽ڵ��waitstatus����ΪSIGNAL
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        //�ڽ�ǰ�̽ڵ��waitstatus����ΪSIGNAL��
        //�ڵ����ѭ��һ�Σ��ж��ܷ��ȡ��
        return false;
    }
```

**�ڽ�ǰ�̽ڵ��waitstatus����ΪSIGNAL�󣬱���ǰ�̽ڵ�����ͷ���֮���Ѻ�̽ڵ㣬��ô��ǰ�ڵ�Ϳ��԰��ĵĽ�������״̬���ȴ�ǰ�̽ڵ�Ļ��ѡ�**

```
private final boolean parkAndCheckInterrupt() {
         //����LockSupport�����������������ǰ�߳�
        LockSupport.park(this);
        return Thread.interrupted();
    }
```
**�����������Ѿ��������ˣ��߳��ڳɹ���������������;�����ʧ���Ժ���ִ�еĲ������߳��ھ�����ʧ���Ժ󣬻���Node�ڵ����ʽβ���뵽ͬ�������У���ͬ��������ֻ��ǰ�̽ڵ���head��Node�ڵ���л��᳢�Ծ������������ڵ�������ǰ�̽ڵ�ΪSIGNAL�󶼽���������״̬��**

-------------------
### UnLock()

 - unlock()������AQS�е�ģ�巽��release()
   
```
    public final boolean release(int arg) {
          //tryReleaseΪAQS�ṩ�����������д�ķ���
        if (tryRelease(arg)) {
            Node h = head;
            //�ж�head��ֵ��waitStatus
            if (h != null && h.waitStatus != 0)
                //����ͷ�ڵ�ĺ�̽ڵ�
                unparkSuccessor(h);
            return true;
        }
        return false;
    }


       //ReentranLock��unfairSync��tryRelease()
     protected final boolean tryRelease(int releases) {
            //ͨ����state��ֵ��Ϊ0����ʾ�ͷ�����
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }




     private void unparkSuccessor(Node node) {
     
        int ws = node.waitStatus;
        if (ws < 0)
        //��ͷ�ڵ��waitStatus����Ϊ0
            compareAndSetWaitStatus(node, ws, 0);
            
        Node s = node.next;
         //���head�ĺ�̽ڵ�Ϊnull�������Ѿ���ȡ����
        if (s == null || s.waitStatus > 0) {
            s = null;
            //���ôӶ���β��ʼ��ǰ������Ѱ��һ��waitStatus<0�Ľڵ�
            for (Node t = tail; t != null && t != node; t = t.prev)          
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            //����ָ���ڵ�
            LockSupport.unpark(s.thread);
    }
```
     
 - **�����ڵ㱻���Ѻ�**
 

```
 private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        //�����ڵ㱻���Ѻ����Ȼ��ж��������������Ƿ��жϹ�
        return Thread.interrupted();
    }

                //�ٴγ��Ի�ȡ��
                for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    //��ȡ�ɹ�������ǰ�ڵ�����Ϊͷ�ڵ�
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
```
**������ReentranLock��lock��unlock����������Ѿ��������ˣ����������ںܶ�ϸ��û�й˼�����������жϵĴ���ȡ�**

-------------------
     

