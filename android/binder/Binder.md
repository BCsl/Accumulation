# Binder浅析

6：defaultServiceManager处理ADD_SERVICE_TRANSACTION过程分析：

MediaPlayerService是BnMediaPlayerService的子类，也就是BBinder的子类

```java
void MediaPlayerService::instantiate() { defaultServiceManager()->addService(
            String16("media.player"), new MediaPlayerService());
}
```

## 1.事件分析处理：

```java
....
case ADD_SERVICE_TRANSACTION: {
//权限检测
            String16 which = data.readString16();
            sp<IBinder> b = data.readStrongBinder();//需要处理的service
            status_t err = addService(which, b);
            reply->writeInt32(err);
            return NO_ERROR;
        } break;
....
```

## 2.具体的方法来处理事件：

```java
virtual status_t addService(const String16& name, const sp<IBinder>& service,
            bool allowIsolated)
    {
        Parcel data, reply;
        data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());//android.os.IServiceManager
        data.writeString16(name);//需要添加的service的名字
        data.writeStrongBinder(service);
        data.writeInt32(allowIsolated ? 1 : 0);//默认为false，写入0//ADD_SERVICE_TRANSACTION等于3

        status_t err = remote()->transact(ADD_SERVICE_TRANSACTION, data, &reply);//交给BpBinder处理
        return err == NO_ERROR ? reply.readExceptionCode() : err;
    }
```

## 3.BpBinder的交由IPCThreadState处理：

```java
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    // Once a binder has died, it will never come back to life.
    if (mAlive) {
        status_t status = IPCThreadState::self()->transact(
            mHandle, code, data, reply, flags);    //mHandle 等于0，code等于3
        if (status == DEAD_OBJECT) mAlive = 0;
        return status;
    }
}
```

## 4.IPCThreadState处理：

```java
status_t IPCThreadState::transact(int32_t handle,uint32_t code, const Parcel& data,Parcel* reply, uint32_t flags)
{
    status_t err = data.errorCheck();

    flags |= TF_ACCEPT_FDS;// flag默认为0，TF_ACCEPT_FDS=0x10

    if (err == NO_ERROR) {
        //写入命令BC_TRANSACTION,flag=TF_ACCPECT_FDS=10,handle=0,code=ADD_SERVICE_TRANSACTION=3
        err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);
    }
    //writeTransactionData有可能会失败
    if (err != NO_ERROR) {
        if (reply) reply->setError(err);
        return (mLastError = err);
    }
    if ((flags & TF_ONE_WAY) == 0) {  //0x10 &  0x01
        if (reply) {
            err = waitForResponse(reply);
        } else {
            Parcel fakeReply;
            err = waitForResponse(&fakeReply);
        }
    } else {
        err = waitForResponse(NULL, NULL);
    }

    return err;
}
```

## 5.剩下的是两个方法writeTransactioData和waitForResponse，一写一等：

writeTransactionData主要工作是填充mOut，但并没有发出数据:

```java
status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer){
    binder_transaction_data tr;

    tr.target.handle = handle;//0
    tr.code = code;// 0 ADD_SERVICE_TRANSACTION等于3
    tr.flags = binderFlags;//10
    tr.cookie = 0;
    tr.sender_pid = 0;
    tr.sender_euid = 0;

    const status_t err = data.errorCheck();
    if (err == NO_ERROR) {
        tr.data_size = data.ipcDataSize();
        tr.data.ptr.buffer = data.ipcData();
        tr.offsets_size = data.ipcObjectsCount()*sizeof(size_t);
        tr.data.ptr.offsets = data.ipcObjects();
    } else if (statusBuffer) {
        tr.flags |= TF_STATUS_CODE;//TF_STATUS_CODE==0X08
        *statusBuffer = err;
        tr.data_size = sizeof(status_t);
        tr.data.ptr.buffer = statusBuffer;
        tr.offsets_size = 0;
        tr.data.ptr.offsets = NULL;
    } else {
        return (mLastError = err);
    }
    mOut.writeInt32(cmd);
    mOut.write(&tr, sizeof(tr));
    return NO_ERROR;
  }
```

等待响应

```java
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    //acquireResult为空
    int32_t cmd;
    int32_t err;

    while (1) {
        if ((err=talkWithDriver()) < NO_ERROR) break;    //与binder通信
        err = mIn.errorCheck();
        if (err < NO_ERROR) break;
        if (mIn.dataAvail() == 0) continue;//dataSize() - dataPosition()

        cmd = mIn.readInt32();
        //我们写入的是BC_TRANSACTION命令，返回来的是BR_xxx格式的命令（BR_TRANSACTION）
        switch (cmd) {
            case BR_TRANSACTION_COMPLETE:
            //...
            case BR_DEAD_REPLY:
            //...
            case BR_FAILED_REPLY:
            //...
            case BR_ACQUIRE_RESULT:
            //...  
            case BR_REPLY:
                      //...
            default:
                err = executeCommand(cmd);//最后似乎交给了executeCommand方法处理
                if (err != NO_ERROR)
                    goto finish;
                break;
        }
    }
finish:
    if (err != NO_ERROR) {
        if (acquireResult) *acquireResult = err;
        if (reply) reply->setError(err);
        mLastError = err;
    }

    return err;
}
```

### talkWithDriver与Binder交互

下面最关键的是ioctl的调用

```java
status_t IPCThreadState::talkWithDriver(bool doReceive)
{
    //doReceive 为true
    if (mProcess->mDriverFD <= 0) {
        return -EBADF;
    }

    binder_write_read bwr;

    // Is the read buffer empty? 还有数据未读完
    const bool needRead = mIn.dataPosition() >= mIn.dataSize();

    // We don't want to write anything if we are still reading
    // from data left in the input buffer and the caller
    // has requested to read the next data.
    //doReceive 默认为true，needRead初始的时候为false
    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;

    bwr.write_size = outAvail;//dataSize ， 前面我们writeTransactionData时候的数据量
    bwr.write_buffer = (long unsigned int)mOut.data();

    // This is what we'll read.
    if (doReceive && needRead) {
        bwr.read_size = mIn.dataCapacity();
        bwr.read_buffer = (long unsigned int)mIn.data();
    } else {
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }

    // Return immediately if there is nothing to do.
    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;

    bwr.write_consumed = 0;
    bwr.read_consumed = 0;
    status_t err;
    do {

#if defined(HAVE_ANDROID_OS)
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)//通过ioctl进行读写
            err = NO_ERROR;
        else
            err = -errno;
#else
        err = INVALID_OPERATION;
#endif
        if (mProcess->mDriverFD <= 0) {
            err = -EBADF;
        }

    } while (err == -EINTR);

    if (err >= NO_ERROR) {
        if (bwr.write_consumed > 0) {
            //有写入
            if (bwr.write_consumed < (ssize_t)mOut.dataSize())
                //可以看出，有可能会不能全部写入
                mOut.remove(0, bwr.write_consumed);
            else
                mOut.setDataSize(0);
        }
        if (bwr.read_consumed > 0) {
            //有读出数据
            mIn.setDataSize(bwr.read_consumed);
            mIn.setDataPosition(0);
        }
        return NO_ERROR;
    }

    return err;
}
```

### Binder内核处理ioctl

arg参数是binder_write_read在用户空间的引用

```java
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    int ret;
    struct binder_proc *proc = filp->private_data;
    struct binder_thread *thread;
    unsigned int size = _IOC_SIZE(cmd);
    void __user *ubuf = (void __user *)arg;//arg实际是binder_write_read在用户空间的引用

    trace_binder_ioctl(cmd, arg);//cmd：BINDER_WRITE_READ

    ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);

    if (ret)
        goto err_unlocked;

    binder_lock(__func__);
    thread = binder_get_thread(proc);
    if (thread == NULL) {
        ret = -ENOMEM;
        goto err;
    }

    switch (cmd) {
    case BINDER_WRITE_READ:
        ret = binder_ioctl_write_read(filp, cmd, arg, thread);//最终会调用binder_ioctl_write_read
        if (ret)
            goto err;
        break;
    case BINDER_SET_MAX_THREADS:
        //...
    case BINDER_SET_CONTEXT_MGR:
    //...
    case BINDER_THREAD_EXIT:
    //...
    case BINDER_VERSION:
    //...
    default:
        ret = -EINVAL;
        goto err;
    }
    ret = 0;
err:
    if (thread)
        thread->looper &= ~BINDER_LOOPER_STATE_NEED_RETURN;
    binder_unlock(__func__);
    wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
    if (ret && ret != -ERESTARTSYS)
        pr_info("%d:%d ioctl %x %lx returned %d\n", proc->pid, current->pid, cmd, arg, ret);
err_unlocked:
    trace_binder_ioctl_done(ret);
    return ret;
}
```

binder_ioctl_write_read，又分了write和read两个操作，先处理write再处理read（因为read是阻塞的）

```java
ic int binder_ioctl_write_read(struct file *filp,unsigned int cmd, unsigned long arg,struct binder_thread *thread)
{
    int ret = 0;
    struct binder_proc *proc = filp->private_data;
    unsigned int size = _IOC_SIZE(cmd);
    void __user *ubuf = (void __user *)arg;//arg实际是binder_write_read在用户空间的引用
    struct binder_write_read bwr;

    if (size != sizeof(struct binder_write_read)) {
        ret = -EINVAL;
        goto out;
    }
    if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {
        ret = -EFAULT;
        goto out;
    }

    if (bwr.write_size > 0) {
        ret = binder_thread_write(proc, thread,bwr.write_buffer,bwr.write_size,&bwr.write_consumed);
        trace_binder_write_done(ret);
        if (ret < 0) {
            bwr.read_consumed = 0;
            if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
                ret = -EFAULT;
            goto out;
        }
    }
    if (bwr.read_size > 0) {
        ret = binder_thread_read(proc, thread, bwr.read_buffer,bwr.read_size,&bwr.read_consumed,filp->f_flags & O_NONBLOCK);
        trace_binder_read_done(ret);
        if (!list_empty(&proc->todo))
            wake_up_interruptible(&proc->wait);
        if (ret < 0) {
            if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
                ret = -EFAULT;
            goto out;
        }
    }
    binder_debug(BINDER_DEBUG_READ_WRITE,
             "%d:%d wrote %lld of %lld, read return %lld of %lld\n",
             proc->pid, thread->pid,
             (u64)bwr.write_consumed, (u64)bwr.write_size,
             (u64)bwr.read_consumed, (u64)bwr.read_size);
    if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
        ret = -EFAULT;
        goto out;
    }
out:
    return ret;
}
```

```java
static int binder_thread_write(struct binder_proc *proc,struct binder_thread *thread,binder_uintptr_t binder_buffer, size_t size,
                binder_size_t *consumed)
    {
        uint32_t cmd;
        void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
        void __user *ptr = buffer + *consumed;
        void __user *end = buffer + size;

        while (ptr < end && thread->return_error == BR_OK) {
            if (get_user(cmd, (uint32_t __user *)ptr))
                return -EFAULT;
            ptr += sizeof(uint32_t);
            trace_binder_command(cmd);
            if (_IOC_NR(cmd) < ARRAY_SIZE(binder_stats.bc)) {
                binder_stats.bc[_IOC_NR(cmd)]++;
                proc->stats.bc[_IOC_NR(cmd)]++;
                thread->stats.bc[_IOC_NR(cmd)]++;
            }
            switch (cmd) {
      //与Server进程的Binder本地对象引用计数相关
            case BC_INCREFS:
            case BC_ACQUIRE:
            case BC_RELEASE:
            case BC_DECREFS: {
                uint32_t target;
                struct binder_ref *ref;
                const char *debug_string;

                if (get_user(target, (uint32_t __user *)ptr))
                    return -EFAULT;
                ptr += sizeof(uint32_t);
                if (target == 0 && binder_context_mgr_node &&
                    (cmd == BC_INCREFS || cmd == BC_ACQUIRE)) {
                    ref = binder_get_ref_for_node(proc,
                               binder_context_mgr_node);
                    if (ref->desc != target) {
                        binder_user_error("%d:%d tried to acquire reference to desc 0, got %d instead\n",
                            proc->pid, thread->pid,
                            ref->desc);
                    }
                } else
                    ref = binder_get_ref(proc, target);
                if (ref == NULL) {
                    binder_user_error("%d:%d refcount change on invalid ref %d\n",
                        proc->pid, thread->pid, target);
                    break;
                }
                switch (cmd) {
                case BC_INCREFS:
                    debug_string = "IncRefs";
                    binder_inc_ref(ref, 0, NULL);
                    break;
                case BC_ACQUIRE:
                    debug_string = "Acquire";
                    binder_inc_ref(ref, 1, NULL);
                    break;
                case BC_RELEASE:
                    debug_string = "Release";
                    binder_dec_ref(ref, 1);
                    break;
                case BC_DECREFS:
                default:
                    debug_string = "DecRefs";
                    binder_dec_ref(ref, 0);
                    break;
                }
                binder_debug(BINDER_DEBUG_USER_REFS,
                         "%d:%d %s ref %d desc %d s %d w %d for node %d\n",
                         proc->pid, thread->pid, debug_string, ref->debug_id,
                         ref->desc, ref->strong, ref->weak, ref->node->debug_id);
                break;
            }
            case BC_INCREFS_DONE:
            case BC_ACQUIRE_DONE: {
                binder_uintptr_t node_ptr;
                binder_uintptr_t cookie;
                struct binder_node *node;

                if (get_user(node_ptr, (binder_uintptr_t __user *)ptr))
                    return -EFAULT;
                ptr += sizeof(binder_uintptr_t);
                if (get_user(cookie, (binder_uintptr_t __user *)ptr))
                    return -EFAULT;
                ptr += sizeof(binder_uintptr_t);
                node = binder_get_node(proc, node_ptr);
                if (node == NULL) {
                    binder_user_error("%d:%d %s u%016llx no match\n",
                        proc->pid, thread->pid,
                        cmd == BC_INCREFS_DONE ?
                        "BC_INCREFS_DONE" :
                        "BC_ACQUIRE_DONE",
                        (u64)node_ptr);
                    break;
                }
                if (cookie != node->cookie) {
                    binder_user_error("%d:%d %s u%016llx node %d cookie mismatch %016llx != %016llx\n",
                        proc->pid, thread->pid,
                        cmd == BC_INCREFS_DONE ?
                        "BC_INCREFS_DONE" : "BC_ACQUIRE_DONE",
                        (u64)node_ptr, node->debug_id,
                        (u64)cookie, (u64)node->cookie);
                    break;
                }
                if (cmd == BC_ACQUIRE_DONE) {
                    if (node->pending_strong_ref == 0) {
                        binder_user_error("%d:%d BC_ACQUIRE_DONE node %d has no pending acquire request\n",
                            proc->pid, thread->pid,
                            node->debug_id);
                        break;
                    }
                    node->pending_strong_ref = 0;
                } else {
                    if (node->pending_weak_ref == 0) {
                        binder_user_error("%d:%d BC_INCREFS_DONE node %d has no pending increfs request\n",
                            proc->pid, thread->pid,
                            node->debug_id);
                        break;
                    }
                    node->pending_weak_ref = 0;
                }
                binder_dec_node(node, cmd == BC_ACQUIRE_DONE, 0);
                binder_debug(BINDER_DEBUG_USER_REFS,
                         "%d:%d %s node %d ls %d lw %d\n",
                         proc->pid, thread->pid,
                         cmd == BC_INCREFS_DONE ? "BC_INCREFS_DONE" : "BC_ACQUIRE_DONE",
                         node->debug_id, node->local_strong_refs, node->local_weak_refs);
                break;
            }
            case BC_ATTEMPT_ACQUIRE:
                pr_err("BC_ATTEMPT_ACQUIRE not supported\n");
                return -EINVAL;
            case BC_ACQUIRE_RESULT:
                pr_err("BC_ACQUIRE_RESULT not supported\n");
                return -EINVAL;

            case BC_FREE_BUFFER: {
                binder_uintptr_t data_ptr;
                struct binder_buffer *buffer;

                if (get_user(data_ptr, (binder_uintptr_t __user *)ptr))
                    return -EFAULT;
                ptr += sizeof(binder_uintptr_t);

                buffer = binder_buffer_lookup(proc, data_ptr);
                if (buffer == NULL) {
                    binder_user_error("%d:%d BC_FREE_BUFFER u%016llx no match\n",
                        proc->pid, thread->pid, (u64)data_ptr);
                    break;
                }
                if (!buffer->allow_user_free) {
                    binder_user_error("%d:%d BC_FREE_BUFFER u%016llx matched unreturned buffer\n",
                        proc->pid, thread->pid, (u64)data_ptr);
                    break;
                }
                binder_debug(BINDER_DEBUG_FREE_BUFFER,
                         "%d:%d BC_FREE_BUFFER u%016llx found buffer %d for %s transaction\n",
                         proc->pid, thread->pid, (u64)data_ptr,
                         buffer->debug_id,
                         buffer->transaction ? "active" : "finished");

                if (buffer->transaction) {
                    buffer->transaction->buffer = NULL;
                    buffer->transaction = NULL;
                }
                if (buffer->async_transaction && buffer->target_node) {
                    BUG_ON(!buffer->target_node->has_async_transaction);
                    if (list_empty(&buffer->target_node->async_todo))
                        buffer->target_node->has_async_transaction = 0;
                    else
                        list_move_tail(buffer->target_node->async_todo.next, &thread->todo);
                }
                trace_binder_transaction_buffer_release(buffer);
                binder_transaction_buffer_release(proc, buffer, NULL);
                binder_free_buf(proc, buffer);
                break;
            }

            case BC_TRANSACTION:
            case BC_REPLY: {
                struct binder_transaction_data tr;

                if (copy_from_user(&tr, ptr, sizeof(tr)))
                    return -EFAULT;
                ptr += sizeof(tr);
                binder_transaction(proc, thread, &tr, cmd == BC_REPLY);
                break;
            }

            case BC_REGISTER_LOOPER:
                binder_debug(BINDER_DEBUG_THREADS,
                         "%d:%d BC_REGISTER_LOOPER\n",
                         proc->pid, thread->pid);
                if (thread->looper & BINDER_LOOPER_STATE_ENTERED) {
                    thread->looper |= BINDER_LOOPER_STATE_INVALID;
                    binder_user_error("%d:%d ERROR: BC_REGISTER_LOOPER called after BC_ENTER_LOOPER\n",
                        proc->pid, thread->pid);
                } else if (proc->requested_threads == 0) {
                    thread->looper |= BINDER_LOOPER_STATE_INVALID;
                    binder_user_error("%d:%d ERROR: BC_REGISTER_LOOPER called without request\n",
                        proc->pid, thread->pid);
                } else {
                    proc->requested_threads--;
                    proc->requested_threads_started++;
                }
                thread->looper |= BINDER_LOOPER_STATE_REGISTERED;
                break;
            case BC_ENTER_LOOPER:
                binder_debug(BINDER_DEBUG_THREADS,
                         "%d:%d BC_ENTER_LOOPER\n",
                         proc->pid, thread->pid);
                if (thread->looper & BINDER_LOOPER_STATE_REGISTERED) {
                    thread->looper |= BINDER_LOOPER_STATE_INVALID;
                    binder_user_error("%d:%d ERROR: BC_ENTER_LOOPER called after BC_REGISTER_LOOPER\n",
                        proc->pid, thread->pid);
                }
                thread->looper |= BINDER_LOOPER_STATE_ENTERED;
                break;
            case BC_EXIT_LOOPER:
                binder_debug(BINDER_DEBUG_THREADS,
                         "%d:%d BC_EXIT_LOOPER\n",
                         proc->pid, thread->pid);
                thread->looper |= BINDER_LOOPER_STATE_EXITED;
                break;

            case BC_REQUEST_DEATH_NOTIFICATION:
            case BC_CLEAR_DEATH_NOTIFICATION: {
                uint32_t target;
                binder_uintptr_t cookie;
                struct binder_ref *ref;
                struct binder_ref_death *death;

                if (get_user(target, (uint32_t __user *)ptr))
                    return -EFAULT;
                ptr += sizeof(uint32_t);
                if (get_user(cookie, (binder_uintptr_t __user *)ptr))
                    return -EFAULT;
                ptr += sizeof(binder_uintptr_t);
                ref = binder_get_ref(proc, target);
                if (ref == NULL) {
                    binder_user_error("%d:%d %s invalid ref %d\n",
                        proc->pid, thread->pid,
                        cmd == BC_REQUEST_DEATH_NOTIFICATION ?
                        "BC_REQUEST_DEATH_NOTIFICATION" :
                        "BC_CLEAR_DEATH_NOTIFICATION",
                        target);
                    break;
                }

                binder_debug(BINDER_DEBUG_DEATH_NOTIFICATION,
                         "%d:%d %s %016llx ref %d desc %d s %d w %d for node %d\n",
                         proc->pid, thread->pid,
                         cmd == BC_REQUEST_DEATH_NOTIFICATION ?
                         "BC_REQUEST_DEATH_NOTIFICATION" :
                         "BC_CLEAR_DEATH_NOTIFICATION",
                         (u64)cookie, ref->debug_id, ref->desc,
                         ref->strong, ref->weak, ref->node->debug_id);

                if (cmd == BC_REQUEST_DEATH_NOTIFICATION) {
                    if (ref->death) {
                        binder_user_error("%d:%d BC_REQUEST_DEATH_NOTIFICATION death notification already set\n",
                            proc->pid, thread->pid);
                        break;
                    }
                    death = kzalloc(sizeof(*death), GFP_KERNEL);
                    if (death == NULL) {
                        thread->return_error = BR_ERROR;
                        binder_debug(BINDER_DEBUG_FAILED_TRANSACTION,
                                 "%d:%d BC_REQUEST_DEATH_NOTIFICATION failed\n",
                                 proc->pid, thread->pid);
                        break;
                    }
                    binder_stats_created(BINDER_STAT_DEATH);
                    INIT_LIST_HEAD(&death->work.entry);
                    death->cookie = cookie;
                    ref->death = death;
                    if (ref->node->proc == NULL) {
                        ref->death->work.type = BINDER_WORK_DEAD_BINDER;
                        if (thread->looper & (BINDER_LOOPER_STATE_REGISTERED | BINDER_LOOPER_STATE_ENTERED)) {
                            list_add_tail(&ref->death->work.entry, &thread->todo);
                        } else {
                            list_add_tail(&ref->death->work.entry, &proc->todo);
                            wake_up_interruptible(&proc->wait);
                        }
                    }
                } else {
                    if (ref->death == NULL) {
                        binder_user_error("%d:%d BC_CLEAR_DEATH_NOTIFICATION death notification not active\n",
                            proc->pid, thread->pid);
                        break;
                    }
                    death = ref->death;
                    if (death->cookie != cookie) {
                        binder_user_error("%d:%d BC_CLEAR_DEATH_NOTIFICATION death notification cookie mismatch %016llx !=     llx\n",
                            proc->pid, thread->pid,
                            (u64)death->cookie,
                            (u64)cookie);
                        break;
                    }
                    ref->death = NULL;
                    if (list_empty(&death->work.entry)) {
                        death->work.type = BINDER_WORK_CLEAR_DEATH_NOTIFICATION;
                        if (thread->looper & (BINDER_LOOPER_STATE_REGISTERED | BINDER_LOOPER_STATE_ENTERED)) {
                            list_add_tail(&death->work.entry, &thread->todo);
                        } else {
                            list_add_tail(&death->work.entry, &proc->todo);
                            wake_up_interruptible(&proc->wait);
                        }
                    } else {
                        BUG_ON(death->work.type != BINDER_WORK_DEAD_BINDER);
                        death->work.type = BINDER_WORK_DEAD_BINDER_AND_CLEAR;
                    }
                }
            } break;
            case BC_DEAD_BINDER_DONE: {
                struct binder_work *w;
                binder_uintptr_t cookie;
                struct binder_ref_death *death = NULL;

                if (get_user(cookie, (binder_uintptr_t __user *)ptr))
                    return -EFAULT;

                ptr += sizeof(void *);
                list_for_each_entry(w, &proc->delivered_death, entry) {
                    struct binder_ref_death *tmp_death = container_of(w, struct binder_ref_death, work);

                    if (tmp_death->cookie == cookie) {
                        death = tmp_death;
                        break;
                    }
                }
                binder_debug(BINDER_DEBUG_DEAD_BINDER,
                         "%d:%d BC_DEAD_BINDER_DONE %016llx found %p\n",
                         proc->pid, thread->pid, (u64)cookie,
                         death);
                if (death == NULL) {
                    binder_user_error("%d:%d BC_DEAD_BINDER_DONE %016llx not found\n",
                        proc->pid, thread->pid, (u64)cookie);
                    break;
                }

                list_del_init(&death->work.entry);
                if (death->work.type == BINDER_WORK_DEAD_BINDER_AND_CLEAR) {
                    death->work.type = BINDER_WORK_CLEAR_DEATH_NOTIFICATION;
                    if (thread->looper & (BINDER_LOOPER_STATE_REGISTERED | BINDER_LOOPER_STATE_ENTERED)) {
                        list_add_tail(&death->work.entry, &thread->todo);
                    } else {
                        list_add_tail(&death->work.entry, &proc->todo);
                        wake_up_interruptible(&proc->wait);
                    }
                }
            } break;

            default:
                pr_err("%d:%d unknown command %d\n",
                       proc->pid, thread->pid, cmd);
                return -EINVAL;
            }
            *consumed = ptr - buffer;
        }
        return 0;
    }
```

binder_thread_read

```java
static int binder_thread_read(struct binder_proc *proc,struct binder_thread *thread,binder_uintptr_t binder_buffer, size_t size,
                  binder_size_t *consumed, int non_block)
{
    void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
    void __user *ptr = buffer + *consumed;
    void __user *end = buffer + size;

    int ret = 0;
    int wait_for_proc_work;

    if (*consumed == 0) {
        if (put_user(BR_NOOP, (uint32_t __user *)ptr))
            return -EFAULT;
        ptr += sizeof(uint32_t);
    }

retry:
    wait_for_proc_work = thread->transaction_stack == NULL &&
                list_empty(&thread->todo);

    if (thread->return_error != BR_OK && ptr < end) {
        if (thread->return_error2 != BR_OK) {
            if (put_user(thread->return_error2, (uint32_t __user *)ptr))
                return -EFAULT;
            ptr += sizeof(uint32_t);
            binder_stat_br(proc, thread, thread->return_error2);
            if (ptr == end)
                goto done;
            thread->return_error2 = BR_OK;
        }
        if (put_user(thread->return_error, (uint32_t __user *)ptr))
            return -EFAULT;
        ptr += sizeof(uint32_t);
        binder_stat_br(proc, thread, thread->return_error);
        thread->return_error = BR_OK;
        goto done;
    }


    thread->looper |= BINDER_LOOPER_STATE_WAITING;
    if (wait_for_proc_work)
        proc->ready_threads++;

    binder_unlock(__func__);

    trace_binder_wait_for_work(wait_for_proc_work,
                   !!thread->transaction_stack,
                   !list_empty(&thread->todo));
    if (wait_for_proc_work) {
        if (!(thread->looper & (BINDER_LOOPER_STATE_REGISTERED |
                    BINDER_LOOPER_STATE_ENTERED))) {
            binder_user_error("%d:%d ERROR: Thread waiting for process work before calling BC_REGISTER_LOOPER or BC_ENTER_LOOPER (state %x)\n",
                proc->pid, thread->pid, thread->looper);
            wait_event_interruptible(binder_user_error_wait,
                         binder_stop_on_user_error < 2);
        }
        binder_set_nice(proc->default_priority);
        if (non_block) {
            if (!binder_has_proc_work(proc, thread))
                ret = -EAGAIN;
        } else
            ret = wait_event_freezable_exclusive(proc->wait, binder_has_proc_work(proc, thread));
    } else {
        if (non_block) {
            if (!binder_has_thread_work(thread))
                ret = -EAGAIN;
        } else
            ret = wait_event_freezable(thread->wait, binder_has_thread_work(thread));
    }

    binder_lock(__func__);

    if (wait_for_proc_work)
        proc->ready_threads--;
    thread->looper &= ~BINDER_LOOPER_STATE_WAITING;

    if (ret)
        return ret;

    while (1) {
        uint32_t cmd;
        struct binder_transaction_data tr;
        struct binder_work *w;
        struct binder_transaction *t = NULL;

        if (!list_empty(&thread->todo)) {
            w = list_first_entry(&thread->todo, struct binder_work,
                         entry);
        } else if (!list_empty(&proc->todo) && wait_for_proc_work) {
            w = list_first_entry(&proc->todo, struct binder_work,
                         entry);
        } else {
            /* no data added */
            if (ptr - buffer == 4 &&
                !(thread->looper & BINDER_LOOPER_STATE_NEED_RETURN))
                goto retry;
            break;
        }

        if (end - ptr < sizeof(tr) + 4)
            break;

        switch (w->type) {
        case BINDER_WORK_TRANSACTION: {
            t = container_of(w, struct binder_transaction, work);
        } break;
        case BINDER_WORK_TRANSACTION_COMPLETE: {
            cmd = BR_TRANSACTION_COMPLETE;
            if (put_user(cmd, (uint32_t __user *)ptr))
                return -EFAULT;
            ptr += sizeof(uint32_t);

            binder_stat_br(proc, thread, cmd);
            binder_debug(BINDER_DEBUG_TRANSACTION_COMPLETE,
                     "%d:%d BR_TRANSACTION_COMPLETE\n",
                     proc->pid, thread->pid);

            list_del(&w->entry);
            kfree(w);
            binder_stats_deleted(BINDER_STAT_TRANSACTION_COMPLETE);
        } break;
        case BINDER_WORK_NODE: {
            struct binder_node *node = container_of(w, struct binder_node, work);
            uint32_t cmd = BR_NOOP;
            const char *cmd_name;
            int strong = node->internal_strong_refs || node->local_strong_refs;
            int weak = !hlist_empty(&node->refs) || node->local_weak_refs || strong;

            if (weak && !node->has_weak_ref) {
                cmd = BR_INCREFS;
                cmd_name = "BR_INCREFS";
                node->has_weak_ref = 1;
                node->pending_weak_ref = 1;
                node->local_weak_refs++;
            } else if (strong && !node->has_strong_ref) {
                cmd = BR_ACQUIRE;
                cmd_name = "BR_ACQUIRE";
                node->has_strong_ref = 1;
                node->pending_strong_ref = 1;
                node->local_strong_refs++;
            } else if (!strong && node->has_strong_ref) {
                cmd = BR_RELEASE;
                cmd_name = "BR_RELEASE";
                node->has_strong_ref = 0;
            } else if (!weak && node->has_weak_ref) {
                cmd = BR_DECREFS;
                cmd_name = "BR_DECREFS";
                node->has_weak_ref = 0;
            }
            if (cmd != BR_NOOP) {
                if (put_user(cmd, (uint32_t __user *)ptr))
                    return -EFAULT;
                ptr += sizeof(uint32_t);
                if (put_user(node->ptr,
                         (binder_uintptr_t __user *)ptr))
                    return -EFAULT;
                ptr += sizeof(binder_uintptr_t);
                if (put_user(node->cookie,
                         (binder_uintptr_t __user *)ptr))
                    return -EFAULT;
                ptr += sizeof(binder_uintptr_t);

                binder_stat_br(proc, thread, cmd);
                binder_debug(BINDER_DEBUG_USER_REFS,
                         "%d:%d %s %d u%016llx c%016llx\n",
                         proc->pid, thread->pid, cmd_name,
                         node->debug_id,
                         (u64)node->ptr, (u64)node->cookie);
            } else {
                list_del_init(&w->entry);
                if (!weak && !strong) {
                    binder_debug(BINDER_DEBUG_INTERNAL_REFS,
                             "%d:%d node %d u%016llx c%016llx deleted\n",
                             proc->pid, thread->pid,
                             node->debug_id,
                             (u64)node->ptr,
                             (u64)node->cookie);
                    rb_erase(&node->rb_node, &proc->nodes);
                    kfree(node);
                    binder_stats_deleted(BINDER_STAT_NODE);
                } else {
                    binder_debug(BINDER_DEBUG_INTERNAL_REFS,
                             "%d:%d node %d u%016llx c%016llx state unchanged\n",
                             proc->pid, thread->pid,
                             node->debug_id,
                             (u64)node->ptr,
                             (u64)node->cookie);
                }
            }
        } break;
        case BINDER_WORK_DEAD_BINDER:
        case BINDER_WORK_DEAD_BINDER_AND_CLEAR:
        case BINDER_WORK_CLEAR_DEATH_NOTIFICATION: {
            struct binder_ref_death *death;
            uint32_t cmd;

            death = container_of(w, struct binder_ref_death, work);
            if (w->type == BINDER_WORK_CLEAR_DEATH_NOTIFICATION)
                cmd = BR_CLEAR_DEATH_NOTIFICATION_DONE;
            else
                cmd = BR_DEAD_BINDER;
            if (put_user(cmd, (uint32_t __user *)ptr))
                return -EFAULT;
            ptr += sizeof(uint32_t);
            if (put_user(death->cookie,
                     (binder_uintptr_t __user *)ptr))
                return -EFAULT;
            ptr += sizeof(binder_uintptr_t);
            binder_stat_br(proc, thread, cmd);
            binder_debug(BINDER_DEBUG_DEATH_NOTIFICATION,
                     "%d:%d %s %016llx\n",
                      proc->pid, thread->pid,
                      cmd == BR_DEAD_BINDER ?
                      "BR_DEAD_BINDER" :
                      "BR_CLEAR_DEATH_NOTIFICATION_DONE",
                      (u64)death->cookie);

            if (w->type == BINDER_WORK_CLEAR_DEATH_NOTIFICATION) {
                list_del(&w->entry);
                kfree(death);
                binder_stats_deleted(BINDER_STAT_DEATH);
            } else
                list_move(&w->entry, &proc->delivered_death);
            if (cmd == BR_DEAD_BINDER)
                goto done; /* DEAD_BINDER notifications can cause transactions */
        } break;
        }

        if (!t)
            continue;

        BUG_ON(t->buffer == NULL);
        if (t->buffer->target_node) {
            struct binder_node *target_node = t->buffer->target_node;

            tr.target.ptr = target_node->ptr;
            tr.cookie =  target_node->cookie;
            t->saved_priority = task_nice(current);
            if (t->priority < target_node->min_priority &&
                !(t->flags & TF_ONE_WAY))
                binder_set_nice(t->priority);
            else if (!(t->flags & TF_ONE_WAY) ||
                 t->saved_priority > target_node->min_priority)
                binder_set_nice(target_node->min_priority);
            cmd = BR_TRANSACTION;
        } else {
            tr.target.ptr = 0;
            tr.cookie = 0;
            cmd = BR_REPLY;
        }
        tr.code = t->code;
        tr.flags = t->flags;
        tr.sender_euid = from_kuid(current_user_ns(), t->sender_euid);

        if (t->from) {
            struct task_struct *sender = t->from->proc->tsk;

            tr.sender_pid = task_tgid_nr_ns(sender,
                            task_active_pid_ns(current));
        } else {
            tr.sender_pid = 0;
        }

        tr.data_size = t->buffer->data_size;
        tr.offsets_size = t->buffer->offsets_size;
        tr.data.ptr.buffer = (binder_uintptr_t)(
                    (uintptr_t)t->buffer->data +
                    proc->user_buffer_offset);
        tr.data.ptr.offsets = tr.data.ptr.buffer +
                    ALIGN(t->buffer->data_size,
                        sizeof(void *));

        if (put_user(cmd, (uint32_t __user *)ptr))
            return -EFAULT;
        ptr += sizeof(uint32_t);
        if (copy_to_user(ptr, &tr, sizeof(tr)))
            return -EFAULT;
        ptr += sizeof(tr);

        trace_binder_transaction_received(t);
        binder_stat_br(proc, thread, cmd);
        binder_debug(BINDER_DEBUG_TRANSACTION,
                 "%d:%d %s %d %d:%d, cmd %d size %zd-%zd ptr %016llx-%016llx\n",
                 proc->pid, thread->pid,
                 (cmd == BR_TRANSACTION) ? "BR_TRANSACTION" :
                 "BR_REPLY",
                 t->debug_id, t->from ? t->from->proc->pid : 0,
                 t->from ? t->from->pid : 0, cmd,
                 t->buffer->data_size, t->buffer->offsets_size,
                 (u64)tr.data.ptr.buffer, (u64)tr.data.ptr.offsets);

        list_del(&t->work.entry);
        t->buffer->allow_user_free = 1;
        if (cmd == BR_TRANSACTION && !(t->flags & TF_ONE_WAY)) {
            t->to_parent = thread->transaction_stack;
            t->to_thread = thread;
            thread->transaction_stack = t;
        } else {
            t->buffer->transaction = NULL;
            kfree(t);
            binder_stats_deleted(BINDER_STAT_TRANSACTION);
        }
        break;
    }

done:

    *consumed = ptr - buffer;
    if (proc->requested_threads + proc->ready_threads == 0 &&
        proc->requested_threads_started < proc->max_threads &&
        (thread->looper & (BINDER_LOOPER_STATE_REGISTERED |
         BINDER_LOOPER_STATE_ENTERED)) /* the user-space code fails to */
         /*spawn a new thread if we leave this out */) {
        proc->requested_threads++;
        binder_debug(BINDER_DEBUG_THREADS,
                 "%d:%d BR_SPAWN_LOOPER\n",
                 proc->pid, thread->pid);
        if (put_user(BR_SPAWN_LOOPER, (uint32_t __user *)buffer))
            return -EFAULT;
        binder_stat_br(proc, thread, BR_SPAWN_LOOPER);
    }
    return 0;
}
```

```java
status_t IPCThreadState::executeCommand(int32_t cmd)
{
    BBinder* obj; //这里终于出现BBinder了
    RefBase::weakref_type* refs;
    status_t result = NO_ERROR;

    switch (cmd) {
    case BR_ERROR:
        //...        
    case BR_OK:
        break;

    case BR_ACQUIRE:
        //...       
    case BR_RELEASE:
        //...       
    case BR_INCREFS:
        //...       
    case BR_DECREFS:
        //...       
    case BR_ATTEMPT_ACQUIRE:
        //...       
    case BR_TRANSACTION:
        {
            binder_transaction_data tr;
            result = mIn.read(&tr, sizeof(tr));

            if (result != NO_ERROR) break;

            Parcel buffer;
            buffer.ipcSetDataReference(
                reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                tr.data_size,
                reinterpret_cast<const size_t*>(tr.data.ptr.offsets),
                tr.offsets_size/sizeof(size_t), freeBuffer, this);

            const pid_t origPid = mCallingPid;
            const uid_t origUid = mCallingUid;

            mCallingPid = tr.sender_pid;
            mCallingUid = tr.sender_euid;

            //优先级相关设置

            //ALOGI(">>>> TRANSACT from pid %d uid %d\n", mCallingPid, mCallingUid);

            Parcel reply;
            if (tr.target.ptr) {
                sp<BBinder> b((BBinder*)tr.cookie);
                const status_t error = b->transact(tr.code, buffer, &reply, tr.flags);//最终又BBinder处理，而BBinder最终交由实现类实现
                if (error < NO_ERROR) reply.setError(error);

            } else {
                //the_context_object 也是 BBinder对象，fileScope，属于IPCThreadState.cpp
                const status_t error = the_context_object->transact(tr.code, buffer, &reply, tr.flags);
                if (error < NO_ERROR) reply.setError(error);
            }

            //ALOGI("<<<< TRANSACT from pid %d restore pid %d uid %d\n",
            //     mCallingPid, origPid, origUid);

            if ((tr.flags & TF_ONE_WAY) == 0) {
                LOG_ONEWAY("Sending reply to %d!", mCallingPid);
                sendReply(reply, 0);
            } else {
                LOG_ONEWAY("NOT sending reply to %d!", mCallingPid);
            }

            mCallingPid = origPid;
            mCallingUid = origUid;

            IF_LOG_TRANSACTIONS() {
                TextOutput::Bundle _b(alog);
                alog << "BC_REPLY thr " << (void*)pthread_self() << " / obj "
                    << tr.target.ptr << ": " << indent << reply << dedent << endl;
            }
        }
        break;

    case BR_DEAD_BINDER:
        //...       
    case BR_CLEAR_DEATH_NOTIFICATION_DONE:
        //...       
    case BR_FINISHED:
        result = TIMED_OUT;
        break;
    case BR_NOOP:
        break;
    case BR_SPAWN_LOOPER:
        mProcess->spawnPooledThread(false);
        break;
    default:
        printf("*** BAD COMMAND %d received from Binder driver\n", cmd);
        result = UNKNOWN_ERROR;
        break;
    }
    if (result != NO_ERROR) {
        mLastError = result;
    }
    return result;
}
```
