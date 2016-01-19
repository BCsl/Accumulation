### Step7 处理有序广播


```java
final void processNextBroadcast(boolean fromMsg) {
    synchronized(mService) {
        BroadcastRecord r;
        //....
        // Now take care of the next serialized one...

        // If we are waiting for a process to come up to handle the next
        // broadcast, then do nothing at this point.  Just in case, we
        // check that the process we're waiting for still exists.
        if (mPendingBroadcast != null) {
            boolean isDead;
            synchronized (mService.mPidsSelfLocked) {
                ProcessRecord proc = mService.mPidsSelfLocked.get(mPendingBroadcast.curApp.pid);
                isDead = proc == null || proc.crashing;
            }
            if (!isDead) {
                // It's still alive, so keep waiting
                return;
            } else {
                Slog.w(TAG, "pending app  ["+ mQueueName + "]" + mPendingBroadcast.curApp  + " died before responding to broadcast");
                mPendingBroadcast.state = BroadcastRecord.IDLE;
                mPendingBroadcast.nextReceiver = mPendingBroadcastRecvIndex;
                mPendingBroadcast = null;
            }
        }

        boolean looped = false;

        do {
            if (mOrderedBroadcasts.size() == 0) {
                // No more broadcasts pending, so all done!
                mService.scheduleAppGcsLocked();
                if (looped) {
                    // If we had finished the last ordered broadcast, then make sure all processes have correct oom and sched
                    // adjustments.
                    mService.updateOomAdjLocked();
                }
                return;
            }
            r = mOrderedBroadcasts.get(0);
            boolean forceReceive = false;

            // Ensure that even if something goes awry with the timeout detection, we catch "hung" broadcasts here, discard them,
            // and continue to make progress.
            //
            // This is only done if the system is ready so that PRE_BOOT_COMPLETED receivers don't get executed with timeouts. //They're intended for one time heavy lifting after system upgrades and can take significant amounts of time.
            int numReceivers = (r.receivers != null) ? r.receivers.size() : 0;
            if (mService.mProcessesReady && r.dispatchTime > 0) {
                long now = SystemClock.uptimeMillis();
                if ((numReceivers > 0) &&(now > r.dispatchTime + (2*mTimeoutPeriod*numReceivers))) {
                    Slog.w(TAG, "Hung broadcast ["+ mQueueName + "] discarded after timeout failure:"
                    //超时处理
                    broadcastTimeoutLocked(false); // forcibly finish this broadcast
                    forceReceive = true;
                    r.state = BroadcastRecord.IDLE;
                }
            }

            if (r.state != BroadcastRecord.IDLE) {
              //正在处理，非空闲状态，等待
                return;
            }
            if (r.receivers == null || r.nextReceiver >= numReceivers|| r.resultAbort || forceReceive) {
                // No more receivers for this broadcast!  Send the final result if requested...
                //resultTo就是目标接收者IIntentReceiver类型的Binder本地对象
                if (r.resultTo != null) {
                    try {
                        performReceiveLocked(r.callerApp, r.resultTo,new Intent(r.intent), r.resultCode,
                            r.resultData, r.resultExtras, false, false, r.userId);
                        // Set this to null so that the reference (local and remote) isn't kept in the mBroadcastHistory.
                        r.resultTo = null;
                    } catch (RemoteException e) {
                        Slog.w(TAG, "Failure ["+ mQueueName + "] sending broadcast result of "+ r.intent, e);
                    }
                }
                if (DEBUG_BROADCAST) Slog.v(TAG, "Cancelling BROADCAST_TIMEOUT_MSG");
                cancelBroadcastTimeoutLocked();

                if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG, "Finished with ordered broadcast "+ r);

                // ... and on to the next...
                addBroadcastToHistoryLocked(r);
                mOrderedBroadcasts.remove(0);
                r = null;
                looped = true;
                continue;
            }
        } while (r == null);

        // Get the next receiver...
        int recIdx = r.nextReceiver++;

        // Keep track of when this receiver started, and make sure there
        // is a timeout message pending to kill it if need be.
        r.receiverTime = SystemClock.uptimeMillis();
        if (recIdx == 0) {//首次取出，dispatchTime等于receiverTime
            r.dispatchTime = r.receiverTime;
            r.dispatchClockTime = System.currentTimeMillis();
        }
        if (! mPendingBroadcastTimeoutMessage) {
            //发出并被处理后mPendingBroadcastTimeoutMessage再置false
            long timeoutTime = r.receiverTime + mTimeoutPeriod;
            setBroadcastTimeoutLocked(timeoutTime);
        }

        Object nextReceiver = r.receivers.get(recIdx);
        if (nextReceiver instanceof BroadcastFilter) {
            // Simple case: this is a registered receiver who gets a direct call.
            //对于动态注册的广播，当前进程必定是存在的
            BroadcastFilter filter = (BroadcastFilter)nextReceiver;
            //发送到具体接收者
            deliverToRegisteredReceiverLocked(r, filter, r.ordered);
            if (r.receiver == null || !r.ordered) {
                // The receiver has already finished, so schedule to process the next one.
                if (DEBUG_BROADCAST) Slog.v(TAG, "Quick finishing ["
                        + mQueueName + "]: ordered="
                        + r.ordered + " receiver=" + r.receiver);
                r.state = BroadcastRecord.IDLE;
                scheduleBroadcastsLocked();
            }
            return;
        }

        // Hard case: need to instantiate the receiver, possibly
        // starting its application process to host it.

        ResolveInfo info =
            (ResolveInfo)nextReceiver;
        ComponentName component = new ComponentName(
                info.activityInfo.applicationInfo.packageName,
                info.activityInfo.name);

        boolean skip = false;
        int perm = mService.checkComponentPermission(info.activityInfo.permission,
                r.callingPid, r.callingUid, info.activityInfo.applicationInfo.uid,
                info.activityInfo.exported);
        if (perm != PackageManager.PERMISSION_GRANTED) {
            if (!info.activityInfo.exported) {
                Slog.w(TAG, "Permission Denial: broadcasting "
                        + r.intent.toString()
                        + " from " + r.callerPackage + " (pid=" + r.callingPid
                        + ", uid=" + r.callingUid + ")"
                        + " is not exported from uid " + info.activityInfo.applicationInfo.uid
                        + " due to receiver " + component.flattenToShortString());
            } else {
                Slog.w(TAG, "Permission Denial: broadcasting "
                        + r.intent.toString()
                        + " from " + r.callerPackage + " (pid=" + r.callingPid
                        + ", uid=" + r.callingUid + ")"
                        + " requires " + info.activityInfo.permission
                        + " due to receiver " + component.flattenToShortString());
            }
            skip = true;
        }
        if (info.activityInfo.applicationInfo.uid != Process.SYSTEM_UID &&
            r.requiredPermission != null) {
            try {
                perm = AppGlobals.getPackageManager().
                        checkPermission(r.requiredPermission,
                                info.activityInfo.applicationInfo.packageName);
            } catch (RemoteException e) {
                perm = PackageManager.PERMISSION_DENIED;
            }
            if (perm != PackageManager.PERMISSION_GRANTED) {
                Slog.w(TAG, "Permission Denial: receiving "
                        + r.intent + " to "
                        + component.flattenToShortString()
                        + " requires " + r.requiredPermission
                        + " due to sender " + r.callerPackage
                        + " (uid " + r.callingUid + ")");
                skip = true;
            }
        }
        if (r.appOp != AppOpsManager.OP_NONE) {
            int mode = mService.mAppOpsService.noteOperation(r.appOp,
                    info.activityInfo.applicationInfo.uid, info.activityInfo.packageName);
            if (mode != AppOpsManager.MODE_ALLOWED) {
                if (DEBUG_BROADCAST)  Slog.v(TAG,
                        "App op " + r.appOp + " not allowed for broadcast to uid "
                        + info.activityInfo.applicationInfo.uid + " pkg "
                        + info.activityInfo.packageName);
                skip = true;
            }
        }
        if (!skip) {
            skip = !mService.mIntentFirewall.checkBroadcast(r.intent, r.callingUid,
                    r.callingPid, r.resolvedType, info.activityInfo.applicationInfo.uid);
        }
        boolean isSingleton = false;
        try {
            isSingleton = mService.isSingleton(info.activityInfo.processName,
                    info.activityInfo.applicationInfo,
                    info.activityInfo.name, info.activityInfo.flags);
        } catch (SecurityException e) {
            Slog.w(TAG, e.getMessage());
            skip = true;
        }
        if ((info.activityInfo.flags&ActivityInfo.FLAG_SINGLE_USER) != 0) {
            if (ActivityManager.checkUidPermission(
                    android.Manifest.permission.INTERACT_ACROSS_USERS,
                    info.activityInfo.applicationInfo.uid)
                            != PackageManager.PERMISSION_GRANTED) {
                Slog.w(TAG, "Permission Denial: Receiver " + component.flattenToShortString()
                        + " requests FLAG_SINGLE_USER, but app does not hold "
                        + android.Manifest.permission.INTERACT_ACROSS_USERS);
                skip = true;
            }
        }
        if (r.curApp != null && r.curApp.crashing) {
            // If the target process is crashing, just skip it.
            Slog.w(TAG, "Skipping deliver ordered [" + mQueueName + "] " + r
                    + " to " + r.curApp + ": process crashing");
            skip = true;
        }

        if (skip) {
            if (DEBUG_BROADCAST)  Slog.v(TAG,
                    "Skipping delivery of ordered ["
                    + mQueueName + "] " + r + " for whatever reason");
            r.receiver = null;
            r.curFilter = null;
            r.state = BroadcastRecord.IDLE;
            scheduleBroadcastsLocked();
            return;
        }

        r.state = BroadcastRecord.APP_RECEIVE;
        String targetProcess = info.activityInfo.processName;
        r.curComponent = component;
        if (r.callingUid != Process.SYSTEM_UID && isSingleton) {
            info.activityInfo = mService.getActivityInfoForUser(info.activityInfo, 0);
        }
        r.curReceiver = info.activityInfo;
        if (DEBUG_MU && r.callingUid > UserHandle.PER_USER_RANGE) {
            Slog.v(TAG_MU, "Updated broadcast record activity info for secondary user, "
                    + info.activityInfo + ", callingUid = " + r.callingUid + ", uid = "
                    + info.activityInfo.applicationInfo.uid);
        }

        // Broadcast is being executed, its package can't be stopped.
        try {
            AppGlobals.getPackageManager().setPackageStoppedState(
                    r.curComponent.getPackageName(), false, UserHandle.getUserId(r.callingUid));
        } catch (RemoteException e) {
        } catch (IllegalArgumentException e) {
            Slog.w(TAG, "Failed trying to unstop package "
                    + r.curComponent.getPackageName() + ": " + e);
        }

        // Is this receiver's application already running?
        ProcessRecord app = mService.getProcessRecordLocked(targetProcess,
                info.activityInfo.applicationInfo.uid, false);
        if (app != null && app.thread != null) {
            try {
                app.addPackage(info.activityInfo.packageName, mService.mProcessStats);
                processCurBroadcastLocked(r, app);
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when sending broadcast to "
                      + r.curComponent, e);
            } catch (RuntimeException e) {
                Log.wtf(TAG, "Failed sending broadcast to "
                        + r.curComponent + " with " + r.intent, e);
                // If some unexpected exception happened, just skip
                // this broadcast.  At this point we are not in the call
                // from a client, so throwing an exception out from here
                // will crash the entire system instead of just whoever
                // sent the broadcast.
                logBroadcastReceiverDiscardLocked(r);
                finishReceiverLocked(r, r.resultCode, r.resultData,
                        r.resultExtras, r.resultAbort, false);
                scheduleBroadcastsLocked();
                // We need to reset the state if we failed to start the receiver.
                r.state = BroadcastRecord.IDLE;
                return;
            }

            // If a dead object exception was thrown -- fall through to
            // restart the application.
        }

        // Not running -- get it started, to be executed when the app comes up.
        if (DEBUG_BROADCAST)  Slog.v(TAG,
                "Need to start app ["
                + mQueueName + "] " + targetProcess + " for broadcast " + r);
        if ((r.curApp=mService.startProcessLocked(targetProcess,
                info.activityInfo.applicationInfo, true,
                r.intent.getFlags() | Intent.FLAG_FROM_BACKGROUND,
                "broadcast", r.curComponent,
                (r.intent.getFlags()&Intent.FLAG_RECEIVER_BOOT_UPGRADE) != 0, false, false))
                        == null) {
            // Ah, this recipient is unavailable.  Finish it if necessary,
            // and mark the broadcast record as ready for the next.
            Slog.w(TAG, "Unable to launch app "
                    + info.activityInfo.applicationInfo.packageName + "/"
                    + info.activityInfo.applicationInfo.uid + " for broadcast "
                    + r.intent + ": process is bad");
            logBroadcastReceiverDiscardLocked(r);
            finishReceiverLocked(r, r.resultCode, r.resultData,
                    r.resultExtras, r.resultAbort, false);
            scheduleBroadcastsLocked();
            r.state = BroadcastRecord.IDLE;
            return;
        }

        mPendingBroadcast = r;
        mPendingBroadcastRecvIndex = recIdx;
    }
}

```
## Step7
```java
```
